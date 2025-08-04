# Calico Ingress Gateway - SNI based Certificate selection

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

In multi-tenant or multi-domain environments, a single ingress point often needs to serve multiple TLS-secured applications, each with its own domain and certificate (e.g., `www.example.com` and `www.sample.com`). Server Name Indication (SNI) in TLS allows the server to choose the correct certificate based on the hostname requested by the client during the handshake.

Envoy Gateway supports SNI-based TLS routing using the Kubernetes Gateway API with Gateway and TLSRoute resources. You can associate different TLS certificates with different hostnames, and Envoy will automatically select the appropriate certificate during the TLS handshake using SNI. This enables secure, multi-domain ingress with clean separation and centralized management—all declaratively configured in Kubernetes.

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

#### 1. Create a `ServiceAccount`, a `Service` and a `Deployment` for the `backend-sni` application

  ```
  cat <<EOF | kubectl apply -f -
  apiVersion: v1
  kind: ServiceAccount
  metadata:
    name: backend-sni
  ---
  apiVersion: v1
  kind: Service
  metadata:
    name: backend-sni
    labels:
      app: backend-sni
      service: backend-sni
  spec:
    ports:
      - name: http
        port: 3000
        targetPort: 3000
    selector:
      app: backend-sni
  ---
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    name: backend-sni
  spec:
    replicas: 1
    selector:
      matchLabels:
        app: backend-sni
        version: v1
    template:
      metadata:
        labels:
          app: backend-sni
          version: v1
      spec:
        serviceAccountName: backend-sni
        containers:
          - image: gcr.io/k8s-staging-gateway-api/echo-basic:v20231214-v1.0.0-140-gf544a46e
            imagePullPolicy: IfNotPresent
            name: backend-sni
            ports:
              - containerPort: 3000
            env:
              - name: POD_NAME
                valueFrom:
                  fieldRef:
                    fieldPath: metadata.name
              - name: NAMESPACE
                valueFrom:
                  fieldRef:
                    fieldPath: metadata.namespace
  EOF
  ```

#### 2. Create a Gateway resource named "sni-gateway" using the "tigera-gateway-class".

  ```
  kubectl apply -f - <<EOF
  apiVersion: gateway.networking.k8s.io/v1
  kind: Gateway
  metadata:
    name: sni-gateway
  spec:
    gatewayClassName: tigera-gateway-class
    listeners:
      - name: http
        protocol: HTTP
        port: 80
  EOF
  ```

#### 3. Create an HTTPRoute `backend-sni` which routes traffic to the `backend-sni` service:
  ```
  kubectl apply -f - <<EOF
  apiVersion: gateway.networking.k8s.io/v1
  kind: HTTPRoute
  metadata:
    name: backend-sni
  spec:
    parentRefs:
      - name: sni-gateway
    hostnames:
      - "www.example.com"
    rules:
      - backendRefs:
          - group: ""
            kind: Service
            name: backend-sni
            port: 3000
            weight: 1
        matches:
          - path:
              type: PathPrefix
              value: /
  EOF
  ```

#### 4. Wait for 30 seconds to allow services and gateway to be ready
sleep 30

#### 5. Retrieve the external IP of the Envoy Gateway

  ```
  export GATEWAY_SNI_DEMO=$(kubectl get gateway/sni-gateway -o jsonpath='{.status.addresses[0].value}')
  ```

#### 6. Test with no certificate

***6.1*** - From the bastion, send a request to the gateway and get a response as shown below:

  ```
  curl --verbose --header "Host: www.example.com" http://$GATEWAY_SNI_DEMO/get
  ```

   A `200` status code should be returned and the body should include `"pod": "backend-sni-*"` indicating the traffic was routed to the example `backend-sni` service. Sample of output:

  ```
  *   Trying 10.10.10.22:80...
  * TCP_NODELAY set
  * Connected to 10.10.10.22 (10.10.10.22) port 80 (#0)
  > GET /get HTTP/1.1
  > Host: www.example.com
  > User-Agent: curl/7.68.0
  > Accept: */*
  >
  * Mark bundle as not supporting multiuse
  < HTTP/1.1 200 OK
  < content-type: application/json
  < x-content-type-options: nosniff
  < date: Mon, 04 Aug 2025 14:23:59 GMT
  < content-length: 471
  <
  {
  "path": "/get",
  "host": "www.example.com",
  "method": "GET",
  "proto": "HTTP/1.1",
  "headers": {
    "Accept": [
    "*/*"
    ],
    "User-Agent": [
    "curl/7.68.0"
    ],
    "X-Envoy-External-Address": [
    "10.0.1.10"
    ],
    "X-Forwarded-For": [
    "10.0.1.10"
    ],
    "X-Forwarded-Proto": [
    "http"
    ],
    "X-Request-Id": [
    "a0335487-7ea9-48dd-9828-59658607064b"
    ]
  },
  "namespace": "default",
  "ingress": "",
  "service": "",
  "pod": "backend-sni-74bdbf5d5f-qh227"
  * Connection #0 to host 10.10.10.22 left intact
  ```

***6.2*** - Generate self-signed RSA derived Server certificate and private key, and configure those in the Gateway listener configuration to terminate HTTPS traffic for `example.com`

  ```
  openssl req -x509 -sha256 -nodes -days 365 -newkey rsa:2048 -subj '/O=example Inc./CN=example.com' -keyout example.com.key -out example.com.crt
  openssl req -out www.example.com.csr -newkey rsa:2048 -nodes -keyout www.example.com.key -subj "/CN=www.example.com/O=example organization"
  openssl x509 -req -days 365 -CA example.com.crt -CAkey example.com.key -set_serial 0 -in www.example.com.csr -out www.example.com.crt
  kubectl create secret tls example-cert --key=www.example.com.key --cert=www.example.com.crt
  ```

***6.3*** - Update the Gateway to include an HTTPS listener that listens on port `443` and references the `example-cert` `Secret`:

  ```
  kubectl patch gateway sni-gateway --type=json --patch '
    - op: add
      path: /spec/listeners/-
      value:
        name: https
        protocol: HTTPS
        port: 443
        tls:
          mode: Terminate
          certificateRefs:
          - kind: Secret
            group: ""
            name: example-cert
    '
  ```
***6.4*** - Query the example app through the Gateway on port 443 and validate the server's certificate using example.com.crt

  ```
  curl -v -HHost:www.example.com --resolve "www.example.com:443:${GATEWAY_SNI_DEMO}" \
--cacert example.com.crt https://www.example.com/get
  ```

  The output should show `200` response and `SSL certificate verify ok`. Sample of output:

  ```
  * Added www.example.com:443:10.10.10.22 to DNS cache
  * Hostname www.example.com was found in DNS cache
  *   Trying 10.10.10.22:443...
  * TCP_NODELAY set
  * Connected to www.example.com (10.10.10.22) port 443 (#0)
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
  * SSL connection using TLSv1.3 / TLS_AES_256_GCM_SHA384
  * ALPN, server accepted to use h2
  * Server certificate:
  *  subject: CN=www.example.com; O=example organization
  *  start date: Aug  4 14:31:48 2025 GMT
  *  expire date: Aug  4 14:31:48 2026 GMT
  *  common name: www.example.com (matched)
  *  issuer: O=example Inc.; CN=example.com
  *  SSL certificate verify ok.
  * Using HTTP2, server supports multi-use
  * Connection state changed (HTTP/2 confirmed)
  * Copying HTTP/2 data in stream buffer to connection buffer after upgrade: len=0
  * Using Stream ID: 1 (easy handle 0x5594f75e7b60)
  > GET /get HTTP/2
  > Host:www.example.com
  > user-agent: curl/7.68.0
  > accept: */*
  >
  * Connection state changed (MAX_CONCURRENT_STREAMS == 100)!
  < HTTP/2 200
  < content-type: application/json
  < x-content-type-options: nosniff
  < date: Mon, 04 Aug 2025 14:40:35 GMT
  < content-length: 472
  <
  {
  "path": "/get",
  "host": "www.example.com",
  "method": "GET",
  "proto": "HTTP/1.1",
  "headers": {
    "Accept": [
    "*/*"
    ],
    "User-Agent": [
    "curl/7.68.0"
    ],
    "X-Envoy-External-Address": [
    "10.0.1.10"
    ],
    "X-Forwarded-For": [
    "10.0.1.10"
    ],
    "X-Forwarded-Proto": [
    "https"
    ],
    "X-Request-Id": [
    "da47c96d-4fa4-4591-aeff-0750125687a6"
    ]
  },
  "namespace": "default",
  "ingress": "",
  "service": "",
  "pod": "backend-sni-74bdbf5d5f-qh227"
  * Connection #0 to host www.example.com left intact
  ```

***6.5*** - Generate self-signed RSA derived Server certificate and private key, and configure those in the Gateway listener configuration to terminate HTTPS traffic for `'sample.com'`
Note that all occurrences of `example.com` were just replaced with `sample.com`.

  ```
  openssl req -x509 -sha256 -nodes -days 365 -newkey rsa:2048 -subj '/O=sample Inc./CN=sample.com' -keyout sample.com.key -out sample.com.crt
  openssl req -out www.sample.com.csr -newkey rsa:2048 -nodes -keyout www.sample.com.key -subj "/CN=www.sample.com/O=sample organization"
  openssl x509 -req -days 365 -CA sample.com.crt -CAkey sample.com.key -set_serial 0 -in www.sample.com.csr -out www.sample.com.crt
  kubectl create secret tls sample-cert --key=www.sample.com.key --cert=www.sample.com.crt
  ```

***6.6*** -  Update the Gateway configuration to accommodate the new Certificate which will be used to Terminate TLS traffic for `www.sample.com`:

  ```
  kubectl patch gateway sni-gateway --type=json --patch '
    - op: add
      path: /spec/listeners/1/tls/certificateRefs/-
      value:
        name: sample-cert
    '
  ```

***6.7*** -  Path the `HTTPRoute` to include `www.sample.com`:

  ```
  kubectl patch httproute backend-sni --type=json --patch '
    - op: add
      path: /spec/hostnames/-
      value: www.sample.com
    '
  ```

***6.8*** - Query the backend-sni application thourgh the gateway for `www.example.com`:

  ```
  curl -v -HHost:www.example.com --resolve "www.example.com:443:${GATEWAY_SNI_DEMO}" \
--cacert example.com.crt https://www.example.com/get -I
  ```

  You should have received the same result as before.

***6.9*** - Query the same backend-sni application thourgh the same gateway for `www.sample.com` instead:

  ```
  curl -v -HHost:www.sample.com --resolve "www.sample.com:443:${GATEWAY_SNI_DEMO}" \
--cacert sample.com.crt https://www.sample.com/get -I
  ```

  The output should show `200` response and `SSL certificate verify ok`. Sample of output:
  ```
  > --cacert sample.com.crt https://www.sample.com/get -I
  * Added www.sample.com:443:10.10.10.22 to DNS cache
  * Hostname www.sample.com was found in DNS cache
  *   Trying 10.10.10.22:443...
  * TCP_NODELAY set
  * Connected to www.sample.com (10.10.10.22) port 443 (#0)
  * ALPN, offering h2
  * ALPN, offering http/1.1
  * successfully set certificate verify locations:
  *   CAfile: sample.com.crt
    CApath: /etc/ssl/certs
  * TLSv1.3 (OUT), TLS handshake, Client hello (1):
  * TLSv1.3 (IN), TLS handshake, Server hello (2):
  * TLSv1.3 (IN), TLS handshake, Encrypted Extensions (8):
  * TLSv1.3 (IN), TLS handshake, Certificate (11):
  * TLSv1.3 (IN), TLS handshake, CERT verify (15):
  * TLSv1.3 (IN), TLS handshake, Finished (20):
  * TLSv1.3 (OUT), TLS change cipher, Change cipher spec (1):
  * TLSv1.3 (OUT), TLS handshake, Finished (20):
  * SSL connection using TLSv1.3 / TLS_AES_256_GCM_SHA384
  * ALPN, server accepted to use h2
  * Server certificate:
  *  subject: CN=www.sample.com; O=sample organization
  *  start date: Aug  4 14:44:16 2025 GMT
  *  expire date: Aug  4 14:44:16 2026 GMT
  *  common name: www.sample.com (matched)
  *  issuer: O=sample Inc.; CN=sample.com
  *  SSL certificate verify ok.
  * Using HTTP2, server supports multi-use
  * Connection state changed (HTTP/2 confirmed)
  * Copying HTTP/2 data in stream buffer to connection buffer after upgrade: len=0
  * Using Stream ID: 1 (easy handle 0x55d71378fb60)
  > HEAD /get HTTP/2
  > Host:www.sample.com
  > user-agent: curl/7.68.0
  > accept: */*
  >
  * Connection state changed (MAX_CONCURRENT_STREAMS == 100)!
  < HTTP/2 200
  HTTP/2 200
  < content-type: application/json
  content-type: application/json
  < x-content-type-options: nosniff
  x-content-type-options: nosniff
  < date: Mon, 04 Aug 2025 14:54:35 GMT
  date: Mon, 04 Aug 2025 14:54:35 GMT
  < content-length: 472
  content-length: 472

  <
  * Connection #0 to host www.sample.com left intact
  ```

### Clean-up

#### 1. Delete app, HTTPRoute, service, Gateway and secrets

  ```
  kubectl delete service backend-sni
  kubectl delete deployment backend-sni
  kubectl delete gateway sni-gateway
  kubectl delete HTTPRoute backend-sni
  kubectl delete secret example-cert sample-cert
  ```

===
> **Congratulations! You have completed `Calico Ingress Gateway Workshop - SNI based Certificate selection`!**

---
**Credits:** Portions of this guide are based on or derived from the [Envoy Gateway documentation](https://gateway.envoyproxy.io/docs/tasks/security/secure-gateways/#sni-based-certificate-selection).