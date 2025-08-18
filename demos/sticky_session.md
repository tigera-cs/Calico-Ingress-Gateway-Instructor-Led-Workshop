# Calico Ingress Gateway - Sticky session with Session Persistence Envoy Route

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

Session Persistence allows client requests to be consistently routed to the same backend service instance. This is useful in many scenarios, such as when an application needs to maintain state across multiple requests.

In Kubernetes with Envoy Gateway, session persistence can be achieved using a session identifier (e.g., a cookie or header). This ensures that a client’s traffic is consistently routed to the same pod for the duration of the session, preserving user context and improving user experience. 

Use session persistence when your app is stateful and you need explicit control over session lifecycle (e.g., login sessions, sticky shopping carts).

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

#### 1. Create a deployment named `Backend` which we will use to test sticky session / session persistence. The deployment will have 4 replicas.

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

#### 2. Create a Gateway resource named "sticky-session-gateway" using the "tigera-gateway-class"

  ```
  kubectl apply -f - <<EOF
  apiVersion: gateway.networking.k8s.io/v1
  kind: Gateway
  metadata:
    name: sticky-session-gateway
  spec:
    gatewayClassName: tigera-gateway-class
    listeners:
      - name: http
        protocol: HTTP
        port: 80
  EOF
  ```

#### 3. Create the http route to configure the session persistence logic
  ```
  kubectl apply -f - <<EOF
  apiVersion: gateway.networking.k8s.io/v1beta1
  kind: HTTPRoute
  metadata:
    name: header
  spec:
    parentRefs:
      - name: sticky-session-gateway
    rules:
      - matches:
          - path:
              type: PathPrefix
              value: /
        backendRefs:
          - name: backend
            port: 3000
        sessionPersistence:
          sessionName: Session-A
          type: Header
          absoluteTimeout: 10s
  EOF
  ```

#### 4. Wait for 30 seconds to allow services and gateway to be ready
sleep 30

#### 5. Retrieve the external IP of the Envoy Gateway

  ```
  export GATEWAY_STICKY_DEMO=$(kubectl get gateway/sticky-session-gateway -o jsonpath='{.status.addresses[0].value}')
  ```

#### 6. Test

From the bastion, send a request to the gateway to get an `header`

  ```
  HEADER=$(curl --verbose http://$GATEWAY_STICKY_DEMO/get 2>&1 | grep "session-a" | awk '{print $3}')
  ```

Send 5 requests to the gateway using that `header`
  ```
  for i in `seq 5`; do
      curl -H "Session-A: $HEADER" http://$GATEWAY_STICKY_DEMO/get 2>/dev/null | grep pod
  done
  ```

As a result, you should see all responses coming from the same pod. Sample:

  ```
  tigera@bastion:~$ for i in `seq 5`; do
  >     curl -H "Session-A: $HEADER" http://$GATEWAY_STICKY_DEMO/get 2>/dev/null | grep pod
  > done
  "pod": "backend-765694d47f-wn8hp"
  "pod": "backend-765694d47f-wn8hp"
  "pod": "backend-765694d47f-wn8hp"
  "pod": "backend-765694d47f-wn8hp"
  "pod": "backend-765694d47f-wn8hp"
  ```

Remove the header and test again:

  ```
  for i in `seq 5`; do
      curl http://$GATEWAY_STICKY_DEMO/get 2>/dev/null | grep pod
  done
  ```

Sample of output:

  ```
  tigera@bastion:~$ for i in `seq 5`; do
  >       curl http://$GATEWAY_STICKY_DEMO/get 2>/dev/null | grep pod
  >   done
  "pod": "backend-765694d47f-whjc9"
  "pod": "backend-765694d47f-9s262"
  "pod": "backend-765694d47f-4fm7c"
  "pod": "backend-765694d47f-wn8hp"
  "pod": "backend-765694d47f-4fm7c"
  ```

### Clean-up

#### 1. Delete app, service, serviceAccount, HTTPRoute and Gateway

  ```
  kubectl delete ServiceAccount backend
  kubectl delete service backend
  kubectl delete deployment backend
  kubectl delete gateway sticky-session-gateway
  kubectl delete HTTPRoute header
  ```

===
> **Congratulations! You have completed `Calico Ingress Gateway Workshop - Sticky session with Session Persistence Envoy Route`!**

---
**Credits:** Portions of this guide are based on or derived from the [Envoy Gateway documentation](https://gateway.envoyproxy.io/docs/tasks/traffic/session-persistence/).