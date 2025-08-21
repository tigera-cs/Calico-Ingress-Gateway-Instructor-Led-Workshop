# Calico Ingress Gateway - Round Robin Load Balancing

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

A common use case for round-robin load balancing is distributing incoming traffic evenly across multiple instances of a stateless web application to maximize resource utilization and ensure high availability.

In Kubernetes with Envoy Gateway, round-robin is one of the default load balancing strategies used by Envoy to forward requests to backend pods. When configured (e.g., via `BackendRef` in `HTTPRoute` or a `Service`), Envoy will cycle through available backend endpoints, sending each new request to the next pod in line, ensuring a balanced distribution without overloading any single instance.

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

5.  <details>
    <summary><code>HEY</code> app is installed on the bastion</summary>

        sudo apt install -y hey
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

#### 1. Create a deployment named `Backend` which we will use to test round robin load balancing. The deployment will have 4 replicas.

  ```
  kubectl apply -f - <<EOF
  apiVersion: v1
  kind: ServiceAccount
  metadata:
    name: backend
  ---
  apiVersion: v1
  kind: Service
  metadata:
    name: backend
    labels:
      app: backend
      service: backend
  spec:
    ports:
      - name: http
        port: 3000
        targetPort: 3000
    selector:
      app: backend
  ---
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    name: backend
  spec:
    replicas: 4
    selector:
      matchLabels:
        app: backend
        version: v1
    template:
      metadata:
        labels:
          app: backend
          version: v1
      spec:
        serviceAccountName: backend
        containers:
          - image: gcr.io/k8s-staging-gateway-api/echo-basic:v20231214-v1.0.0-140-gf544a46e
            imagePullPolicy: IfNotPresent
            name: backend
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

#### 2. Create a Gateway resource named "load-balancing-gateway" using the "tigera-gateway-class"

  ```
  kubectl apply -f - <<EOF
  apiVersion: gateway.networking.k8s.io/v1
  kind: Gateway
  metadata:
    name: load-balancing-gateway
  spec:
    gatewayClassName: tigera-gateway-class
    listeners:
      - name: http
        protocol: HTTP
        port: 80
  EOF
  ```

#### 3. Create the backend traffic policy and the http route to configure the load balancing
  ```
  kubectl apply -f - <<EOF
  apiVersion: gateway.envoyproxy.io/v1alpha1
  kind: BackendTrafficPolicy
  metadata:
    name: round-robin-policy
    namespace: default
  spec:
    targetRefs:
      - group: gateway.networking.k8s.io
        kind: HTTPRoute
        name: round-robin-route
    loadBalancer:
      type: RoundRobin
  ---
  apiVersion: gateway.networking.k8s.io/v1
  kind: HTTPRoute
  metadata:
    name: round-robin-route
    namespace: default
  spec:
    parentRefs:
      - name: load-balancing-gateway
    hostnames:
      - "www.example.com"
    rules:
      - matches:
          - path:
              type: PathPrefix
              value: /round
        backendRefs:
          - name: backend
            port: 3000
  EOF
  ```

#### 4. Wait for 60 seconds to allow services and gateway to be ready

```
sleep 60
```

#### 5. Retrieve the external IP of the Envoy Gateway

  ```
  export GATEWAY_LB_DEMO=$(kubectl get gateway/load-balancing-gateway -o jsonpath='{.status.addresses[0].value}')
  echo "GATEWAY_LB_DEMO is: $GATEWAY_LB_DEMO"
  ```

#### 6. Test

From the bastion, generate 100 concurrent requests using `HEY`

  ```
  hey -n 100 -c 100 -host "www.example.com" http://${GATEWAY_LB_DEMO}/round
  ```

As a result, you should see all available upstream hosts receive traffics evenly with this command:

  ```
  kubectl get pods -l app=backend --no-headers -o custom-columns=":metadata.name" | while read -r pod; do echo "$pod: received $(($(kubectl logs $pod | wc -l) - 2)) requests"; done
  ```

Sample of output:
  ```
  backend-765694d47f-7plj9: received 25 requests
  backend-765694d47f-jm4fg: received 25 requests
  backend-765694d47f-qzbct: received 25 requests
  backend-765694d47f-zk6x5: received 25 requests
  ```

### Clean-up

#### 1. Delete app, service, serviceAccount, HTTPRoute, BackendTrafficPolicy and Gateway

  ```
  kubectl delete ServiceAccount backend
  kubectl delete service backend
  kubectl delete deployment backend
  kubectl delete gateway load-balancing-gateway
  kubectl delete BackendTrafficPolicy round-robin-policy
  kubectl delete HTTPRoute round-robin-route
  ```

===
> **Congratulations! You have completed `Calico Ingress Gateway Workshop - Round Robin Load Balancing`!**

---
**Credits:** Portions of this guide are based on or derived from the [Envoy Gateway documentation](https://gateway.envoyproxy.io/docs/tasks/traffic/load-balancing/#round-robin).