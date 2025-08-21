# Calico Ingress Gateway - TCP Routing

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

TCP routing is used when applications communicate over raw TCP connections rather than HTTP, such as databases (PostgreSQL, MySQL), message brokers (Redis, MQTT), or custom TCP-based protocols. In Kubernetes, managing access to these services externally—especially with advanced traffic control or multi-tenant routing—requires TCP-aware solutions.

Envoy Gateway supports TCP routing via Kubernetes Gateway API (e.g., `TCPRoute`), enabling layer 4 (L4) routing for TCP traffic. This allows you to define rules that forward TCP connections to specific backends (like a DB service) based on port or other connection parameters, offering centralized, declarative control over TCP traffic across services.

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

In this example, we have one Gateway resource and two TCPRoute resources that distribute the traffic with the following rules:
- All TCP streams on port 8088 of the Gateway are forwarded to port 3001 of `foo` Kubernetes Service
- All TCP streams on port 8089 of the Gateway are forwarded to port 3002 of `bar` Kubernetes Service
- Two TCP listeners will be applied to the Gateway in order to route them to two separate backend TCPRoutes, note that the protocol set for the listeners on the Gateway is TCP.

#### 1. Create two services `foo` and `bar`, which are bound to `backend-1` and `backend-2` deployments.

  ```
  cat <<EOF | kubectl apply -f -
  apiVersion: v1
  kind: Service
  metadata:
    name: foo
    labels:
      app: backend-1
  spec:
    ports:
      - name: http
        port: 3001
        targetPort: 3000
    selector:
      app: backend-1
  ---
  apiVersion: v1
  kind: Service
  metadata:
    name: bar
    labels:
      app: backend-2
  spec:
    ports:
      - name: http
        port: 3002
        targetPort: 3000
    selector:
      app: backend-2
  ---
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    name: backend-1
  spec:
    replicas: 1
    selector:
      matchLabels:
        app: backend-1
        version: v1
    template:
      metadata:
        labels:
          app: backend-1
          version: v1
      spec:
        containers:
          - image: gcr.io/k8s-staging-gateway-api/echo-basic:v20231214-v1.0.0-140-gf544a46e
            imagePullPolicy: IfNotPresent
            name: backend-1
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
              - name: SERVICE_NAME
                value: foo
  ---
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    name: backend-2
  spec:
    replicas: 1
    selector:
      matchLabels:
        app: backend-2
        version: v1
    template:
      metadata:
        labels:
          app: backend-2
          version: v1
      spec:
        containers:
          - image: gcr.io/k8s-staging-gateway-api/echo-basic:v20231214-v1.0.0-140-gf544a46e
            imagePullPolicy: IfNotPresent
            name: backend-2
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
              - name: SERVICE_NAME
                value: bar
  EOF
  ```

#### 2. Create a Gateway resource named "tcp-routing-gateway" using the "tigera-gateway-class". The gateway will have 2 listeners.

  ```
  kubectl apply -f - <<EOF
  apiVersion: gateway.networking.k8s.io/v1
  kind: Gateway
  metadata:
    name: tcp-routing-gateway
  spec:
    gatewayClassName: tigera-gateway-class
    listeners:
    - name: foo
      protocol: TCP
      port: 8088
      allowedRoutes:
        kinds:
        - kind: TCPRoute
    - name: bar
      protocol: TCP
      port: 8089
      allowedRoutes:
        kinds:
        - kind: TCPRoute
  EOF
  ```

#### 3. Create two TCPRoutes `tcp-app-1` and `tcp-app-2` with different `sectionName`:
  ```
  kubectl apply -f - <<EOF
  apiVersion: gateway.networking.k8s.io/v1alpha2
  kind: TCPRoute
  metadata:
    name: tcp-app-1
  spec:
    parentRefs:
    - name: tcp-routing-gateway
      sectionName: foo
    rules:
    - backendRefs:
      - name: foo
        port: 3001
  ---
  apiVersion: gateway.networking.k8s.io/v1alpha2
  kind: TCPRoute
  metadata:
    name: tcp-app-2
  spec:
    parentRefs:
    - name: tcp-routing-gateway
      sectionName: bar
    rules:
    - backendRefs:
      - name: bar
        port: 3002
  EOF
  ```

#### 4. Wait for 30 seconds to allow services and gateway to be ready

  ```
  sleep 30
  ```

#### 5. Retrieve the external IP of the Envoy Gateway

  ```
  export GATEWAY_TCP_DEMO=$(kubectl get gateway/tcp-routing-gateway -o jsonpath='{.status.addresses[0].value}')
  echo "GATEWAY_TCP_DEMO is: $GATEWAY_TCP_DEMO"
  ```

#### 6. Test

From the bastion, send requests to the gateway and get responses as shown below:

  For `foo` service:
  ```
  curl -i "http://${GATEWAY_TCP_DEMO}:8088"
  ```

  Sample of output:

  ```
  HTTP/1.1 200 OK
  Content-Type: application/json
  X-Content-Type-Options: nosniff
  Date: Mon, 04 Aug 2025 12:45:51 GMT
  Content-Length: 268

  {
  "path": "/",
  "host": "10.10.10.22:8088",
  "method": "GET",
  "proto": "HTTP/1.1",
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
  "service": "foo",
  "pod": "backend-1-75748d59cf-fj269"
  ```

  For `bar` service:
  ```
  curl -i "http://${GATEWAY_TCP_DEMO}:8089"
  ```

  Sample of output:

  ```
  HTTP/1.1 200 OK
  Content-Type: application/json
  X-Content-Type-Options: nosniff
  Date: Mon, 04 Aug 2025 12:46:23 GMT
  Content-Length: 268

  {
  "path": "/",
  "host": "10.10.10.22:8089",
  "method": "GET",
  "proto": "HTTP/1.1",
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
  "service": "bar",
  "pod": "backend-2-79d9dcccf4-r55jb"
  ```

### Clean-up

#### 1. Delete apps, services, TCPRoutes and Gateway

  ```
  kubectl delete service foo bar
  kubectl delete deployment backend-1 backend-2
  kubectl delete gateway tcp-routing-gateway
  kubectl delete TCPRoute tcp-app-1 tcp-app-2
  ```

===
> **Congratulations! You have completed `Calico Ingress Gateway Workshop - TCP Routing`!**

---
**Credits:** Portions of this guide are based on or derived from the [Envoy Gateway documentation](https://gateway.envoyproxy.io/docs/tasks/traffic/tcp-routing/).