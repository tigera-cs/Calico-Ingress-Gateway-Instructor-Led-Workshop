# Calico Ingress Gateway - TLS Passthrough

### Table of Contents

* [Welcome!](#welcome)
* [Overview](#overview)
* [Before you begin...](#before-you-begin)
* [Demo](#demo)
* [Clean-up](#clean-up)


### Welcome!

Welcome to the **Calico Ingress Gateway Instructor Led Workshop**. 

The Calico Ingress Gateway Workshop aims to explain the kubernetes' and IngressAPI native limitations, the differences between IngressAPI and GatewayAPI and the most common use cases where Calico Ingress Gateway can solve.

We hope you enjoyed the presentation! Feel free to download the slides:
- [Calico Ingress Gateway - Introduction](etc/Calico%20Ingress%20-%20Gateway%20Workshop%20-%20Introduction.pdf)
- [Calico Ingress Gateway - Capabilities](etc/Calico%20Ingress%20-%20Gateway%20Workshop%20-%20Capabilities.pdf)
- [Calico Ingress Gateway - Migration](etc/Calico%20Ingress%20-%20Gateway%20Workshop%20-%20Migration.pdf)

---

### Overview

TLS passthrough is used when you want to let encrypted TLS traffic pass through the gateway without terminating (decrypting) it at the proxy. This is common for workloads that handle their own TLS termination internally—like databases or services requiring end-to-end encryption—so the gateway just forwards raw TLS packets directly to the backend.


Envoy Gateway supports TLS passthrough by routing TCP connections without terminating TLS, typically configured via `TLSRoute` in the Kubernetes Gateway API. This allows Envoy to forward encrypted traffic transparently to backend services that manage their own TLS certificates, preserving end-to-end security while enabling centralized traffic routing and control.

---

### Before you begin...

**IMPORTANT**

* Supported CE/CC versions:
  - Calico Enterprise v3.21.X
  - Calico Cloud 21.X

**PREREQUISITE**

1.  <details>
    <summary>A valid IP pool for loadbalancer services</summary>

        kubectl apply -f - <<EOF
        apiVersion: projectcalico.org/v3
        kind: IPPool
        metadata:
          name: loadbalancer-ip-pool
        spec:
          cidr: 10.10.10.0/26
          blockSize: 31
          natOutgoing: true
          disabled: false
          assignmentMode: Automatic
          allowedUses:
            - LoadBalancer
        EOF
    </details>

2.  <details>
    <summary>Calico BGP Configuration and BGP Peer to advertise serviceLoadBalancerIPs to the bastion or firewall, and peer it with your cluster nodes </summary>
    
        kubectl apply -f - <<EOF
        apiVersion: projectcalico.org/v3
        kind: BGPConfiguration
        metadata:
          name: default
        spec:
          logSeverityScreen: Info
          nodeToNodeMeshEnabled: true
          nodeMeshMaxRestartTime: 120s
          asNumber: 64512
          serviceLoadBalancerIPs:
          - cidr: 10.10.10.0/26
          listenPort: 179
          bindMode: NodeIP
        ---
        apiVersion: projectcalico.org/v3
        kind: BGPPeer
        metadata:
          name: my-global-peer
        spec:
          peerIP: 10.0.1.10
          asNumber: 64512
        EOF
    </details>

3.  <details>
    <summary>The bastion/firewall is peered with cluster nodes and is accepting advetised loadbalancer service IPs (Ingress Gateways) </summary>

        watch sudo birdc show protocols
      
      Sample of output:

        BIRD 1.6.8 ready.
        name     proto    table    state  since       info
        direct1  Direct   master   up     23:42:33    
        kernel1  Kernel   master   up     23:42:33    
        device1  Device   master   up     23:42:33    
        control1 BGP      master   up     23:42:35    Established   
        worker1  BGP      master   up     23:42:36    Established   
        worker2  BGP      master   up     23:42:35    Established

      Check that the route `if (net ~ 10.10.10.0/26) then accept;` has been added correctly:

        sudo grep -A 8 "Import filter"  /etc/bird/bird.conf

      If the route is not there, add it with this command:

        sudo sed -i 's/if (net ~ 10.50.0.0\/24) then accept;/if (net ~ 10.50.0.0\/24) then accept;\n                        if (net ~ 10.10.10.0\/26) then accept;/g' /etc/bird/bird.conf

      And restart bird

        sudo systemctl restart bird
    </details>

4.  <details>
    <summary>Gateway API support is enabled</summary>

        kubectl apply -f - <<EOF
        apiVersion: operator.tigera.io/v1
        kind: GatewayAPI
        metadata:
          name: tigera-secure
        EOF
    </details>

**About Calico Ingress Gateway**

* **Calico Ingress Gateway** is an enterprise-grade ingress solution based on the Kubernetes Gateway API, integrated with Envoy Gateway. It enables advanced, application-layer (L7) traffic control and routing to services within a Kubernetes cluster. Calico Ingress Gateway supports features such as weighted or blue-green load balancing and is designed to provide secure, scalable, and flexible ingress management for cloud-native applications.

* **Gateway API** is an official Kubernetes API for advanced routing to services in a cluster. To read about its use cases, structure and design, please see the official docs. Calico Enterprise provides the following resources and versions of the Gateway API.

  | Resource         | Versions           |
  |------------------|--------------------|
  | BackendLBPolicy  | v1alpha2           |
  | BackendTLSPolicy | v1alpha3           |
  | GatewayClass     | v1, v1beta1        |
  | Gateway          | v1, v1beta1        |
  | GRPCRoute        | v1, v1alpha2       |
  | HTTPRoute        | v1, v1beta1        |
  | ReferenceGrant   | v1beta1, v1alpha2  |
  | TCPRoute         | v1alpha2           |
  | TLSRoute         | v1alpha2           |
  | UDPRoute         | v1alpha2           |

For more details, see the official documentation: [Configure an ingress gateway](https://docs.tigera.io/calico-enterprise/latest/networking/gateway-api).

### Demo

This sections gives a walkthrough to generate multiple certificates corresponding to different FQDNs. The same Gateway listener can then be configured to terminate TLS traffic for multiple FQDNs based on the SNI matching.

#### 1. Deploy certificates and secret which will be used by the `passthrough-echoserver` application to terminate the traffic

  ```
  openssl req -x509 -sha256 -nodes -days 365 -newkey rsa:2048 -subj '/O=example Inc./CN=example.com' -keyout example.com.key -out example.com.crt
  openssl req -out passthrough.example.com.csr -newkey rsa:2048 -nodes -keyout passthrough.example.com.key -subj "/CN=passthrough.example.com/O=some organization"
  openssl x509 -req -sha256 -days 365 -CA example.com.crt -CAkey example.com.key -set_serial 0 -in passthrough.example.com.csr -out passthrough.example.com.crt
  kubectl create secret tls server-certs --key=passthrough.example.com.key --cert=passthrough.example.com.crt
  ```

#### 2. Create a `Service` and a `Deployment` for the `passthrough-echoserver` application

  ```
  cat <<EOF | kubectl apply -f -
  apiVersion: v1
  kind: Service
  metadata:
    name: passthrough-echoserver
    labels:
      run: passthrough-echoserver
  spec:
    ports:
      - port: 443
        targetPort: 8443
        protocol: TCP
    selector:
      run: passthrough-echoserver
  ---
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    name: passthrough-echoserver
  spec:
    selector:
      matchLabels:
        run: passthrough-echoserver
    replicas: 1
    template:
      metadata:
        labels:
          run: passthrough-echoserver
      spec:
        containers:
          - name: passthrough-echoserver
            image: gcr.io/k8s-staging-gateway-api/echo-basic:v20231214-v1.0.0-140-gf544a46e
            ports:
              - containerPort: 8443
            env:
              - name: HTTPS_PORT
                value: "8443"
              - name: TLS_SERVER_CERT
                value: /etc/server-certs/tls.crt
              - name: TLS_SERVER_PRIVKEY
                value: /etc/server-certs/tls.key
              - name: POD_NAME
                valueFrom:
                  fieldRef:
                    fieldPath: metadata.name
              - name: NAMESPACE
                valueFrom:
                  fieldRef:
                    fieldPath: metadata.namespace
            volumeMounts:
              - name: server-certs
                mountPath: /etc/server-certs
                readOnly: true
        volumes:
          - name: server-certs
            secret:
              secretName: server-certs
  EOF
  ```

#### 3. Create a Gateway resource named "tls-passthrough-gateway" using the "tigera-gateway-class".

  ```
  kubectl apply -f - <<EOF
  apiVersion: gateway.networking.k8s.io/v1
  kind: Gateway
  metadata:
    name: tls-passthrough-gateway
  spec:
    gatewayClassName: tigera-gateway-class
    listeners:
      - name: http
        protocol: HTTP
        port: 80
  EOF
  ```

#### 4. Create a TLSRoute `tls-passthrough` which routes traffic to the `passthrough-echoserver` service for the hostname `passthrough.example.com`:
  ```
  kubectl apply -f - <<EOF
  apiVersion: gateway.networking.k8s.io/v1alpha2
  kind: TLSRoute
  metadata:
    name: tls-passthrough
  spec:
    parentRefs:
      - name: tls-passthrough-gateway
    hostnames:
      - "passthrough.example.com"
    rules:
      - backendRefs:
          - group: ""
            kind: Service
            name: passthrough-echoserver
            port: 443
            weight: 1
  EOF
  ```
#### 5. Patch the Gateway to include a TLS listener that listens on port 6443 and is configured for TLS mode Passthrough

  ```
  kubectl patch gateway tls-passthrough-gateway --type=json --patch '
    - op: add
      path: /spec/listeners/-
      value:
        name: tls
        protocol: TLS
        hostname: passthrough.example.com
        port: 6443
        tls:
          mode: Passthrough
    '
  ```

#### 6. Wait for 30 seconds to allow services and gateway to be ready
sleep 30

#### 7. Retrieve the external IP of the Gateway

  ```
  export GATEWAY_TLS_DEMO=$(kubectl get gateway/tls-passthrough-gateway -o jsonpath='{.status.addresses[0].value}')
  ```

#### 6. Test

From the bastion, send a request to the gateway and get a response as shown below:

  ```
  curl -v -HHost:passthrough.example.com --resolve "passthrough.example.com:6443:${GATEWAY_TLS_DEMO}" \
--cacert example.com.crt https://passthrough.example.com:6443/get
  ```

   A `200` status code should be returned and the body should include `"pod": "passthrough-echoserver-*"` indicating the traffic was routed to the example `passthrough-echoserver` service. Sample of output:

  ```
  * Added passthrough.example.com:6443:10.10.10.23 to DNS cache
  * Hostname passthrough.example.com was found in DNS cache
  *   Trying 10.10.10.23:6443...
  * TCP_NODELAY set
  * Connected to passthrough.example.com (10.10.10.23) port 6443 (#0)
  * ALPN, offering h2
  * ALPN, offering http/1.1
  * successfully set certificate verify locations:
  *   CAfile: example.com.crt
    CApath: /etc/ssl/certs
  * TLSv1.3 (OUT), TLS handshake, Client hello (1):
  * TLSv1.3 (IN), TLS handshake, Server hello (2):
  * TLSv1.3 (IN), TLS handshake, Encrypted Extensions (8):
  * TLSv1.3 (IN), TLS handshake, Certificate (11):
  * TLSv1.3 (IN), TLS handshake, CERT verify (15):
  * TLSv1.3 (IN), TLS handshake, Finished (20):
  * TLSv1.3 (OUT), TLS change cipher, Change cipher spec (1):
  * TLSv1.3 (OUT), TLS handshake, Finished (20):
  * SSL connection using TLSv1.3 / TLS_AES_128_GCM_SHA256
  * ALPN, server accepted to use h2
  * Server certificate:
  *  subject: CN=passthrough.example.com; O=some organization
  *  start date: Aug  4 15:13:57 2025 GMT
  *  expire date: Aug  4 15:13:57 2026 GMT
  *  common name: passthrough.example.com (matched)
  *  issuer: O=example Inc.; CN=example.com
  *  SSL certificate verify ok.
  * Using HTTP2, server supports multi-use
  * Connection state changed (HTTP/2 confirmed)
  * Copying HTTP/2 data in stream buffer to connection buffer after upgrade: len=0
  * Using Stream ID: 1 (easy handle 0x5566e4e8ab60)
  > GET /get HTTP/2
  > Host:passthrough.example.com
  > user-agent: curl/7.68.0
  > accept: */*
  >
  * TLSv1.3 (IN), TLS handshake, Newsession Ticket (4):
  * Connection state changed (MAX_CONCURRENT_STREAMS == 250)!
  < HTTP/2 200
  < content-type: application/json
  < x-content-type-options: nosniff
  < content-length: 441
  < date: Mon, 04 Aug 2025 15:23:32 GMT
  <
  {
  "path": "/get",
  "host": "passthrough.example.com",
  "method": "GET",
  "proto": "HTTP/2.0",
  "headers": {
    "Accept": [
    "*/*"
    ],
    "User-Agent": [
    "curl/7.68.0"
    ]
  },
  "namespace": "default",
  "ingress": "",
  "service": "",
  "pod": "passthrough-echoserver-674f8cf47b-4l64m",
  "tls": {
    "version": "TLSv1.3",
    "serverName": "passthrough.example.com",
    "negotiatedProtocol": "h2",
    "cipherSuite": "TLS_AES_128_GCM_SHA256"
  }
  * Connection #0 to host passthrough.example.com left intact
  ```

### Clean-up

#### 1. Delete app, TLSRoute, service, Gateway and secret

  ```
  kubectl delete service passthrough-echoserver
  kubectl delete deployment passthrough-echoserver
  kubectl delete gateway tls-passthrough-gateway
  kubectl delete TLSRoute tls-passthrough
  kubectl delete secret server-certs
  ```

===
> **Congratulations! You have completed `Calico Ingress Gateway Workshop - TLS Passthrough`!**

---
**Credits:** Portions of this guide are based on or derived from the [Envoy Gateway documentation](https://gateway.envoyproxy.io/docs/tasks/security/tls-passthrough/).