# Calico Ingress Gateway - High Availability & Failover

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

Active-passive failover in an API gateway setup is like having a backup plan in place to keep things running smoothly if something goes wrong. Here’s why it’s valuable:
- Staying Online: When the main (or “active”) backend has issues or goes offline, the fallback (or “passive”) backend is ready to step in instantly. This helps keep your API accessible and your services running, so users don’t even notice any interruptions.
- Automatic Switch Over: If a problem occurs, the system can automatically switch traffic over to the fallback backend. This avoids needing someone to jump in and fix things manually, which could take time and might even lead to mistakes.
- Lower Costs: In an active-passive setup, the fallback backend doesn’t need to work all the time—it’s just on standby. This can save on costs (like cloud egress costs) compared to setups where both backend are running at full capacity.
- Peace of Mind with Redundancy: Although the fallback backend isn’t handling traffic daily, it’s there as a safety net. If something happens with the primary backend, the backup can take over immediately, ensuring your service doesn’t skip a beat.

In Kubernetes with Envoy Gateway, failover is handled by configuring multiple backend references with health checks and prioritized or weighted routing. If the primary backend fails (e.g., fails readiness or liveness probes), Envoy automatically routes traffic to a secondary (failover) backend, keeping the service online without manual intervention.

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
    <summary><code>JQ</code> app is installed on the bastion</summary>

        sudo apt install -y jq
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

#### 1. Create deployments and services for `active` and `passive` applications which we will use to test the failover.

  ```
  kubectl apply -f - <<EOF
  apiVersion: v1
  kind: Service
  metadata:
    name: active 
    labels:
      app: active
      service: active
  spec:
    ports:
      - name: http
        port: 3000
        targetPort: 3000
    selector:
      app: active
  ---
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    name: active
  spec:
    replicas: 1
    selector:
      matchLabels:
        app: active
        version: v1
    template:
      metadata:
        labels:
          app: active
          version: v1
      spec:
        containers:
          - image: gcr.io/k8s-staging-gateway-api/echo-basic:v20231214-v1.0.0-140-gf544a46e
            imagePullPolicy: IfNotPresent
            name: active 
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
  ---
  apiVersion: v1
  kind: Service
  metadata:
    name: passive 
    labels:
      app: passive
      service: passive
  spec:
    ports:
      - name: http
        port: 3000
        targetPort: 3000
    selector:
      app: passive
  ---
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    name: passive
  spec:
    replicas: 1
    selector:
      matchLabels:
        app: passive
        version: v1
    template:
      metadata:
        labels:
          app: passive
          version: v1
      spec:
        containers:
          - image: gcr.io/k8s-staging-gateway-api/echo-basic:v20231214-v1.0.0-140-gf544a46e
            imagePullPolicy: IfNotPresent
            name: passive 
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

#### 2. Create the backendAPI resources that are used to represent the active backend and passive backend. Note, we’ve set `fallback: true` for the passive backend to indicate its a passive backend

  ```
  kubectl apply -f - <<EOF
  apiVersion: gateway.envoyproxy.io/v1alpha1
  kind: Backend
  metadata:
    name: passive 
  spec:
    fallback: true
    endpoints:
      - fqdn:
          hostname: passive.default.svc.cluster.local
          port: 3000 
  ---
  apiVersion: gateway.envoyproxy.io/v1alpha1
  kind: Backend
  metadata:
    name: active
  spec:
    endpoints:
    - fqdn:
        hostname: active.default.svc.cluster.local 
        port: 3000
  EOF
  ```

#### 3. Create a Gateway resource named "ha-failover-gateway" using the "tigera-gateway-class"

  ```
  kubectl apply -f - <<EOF
  apiVersion: gateway.networking.k8s.io/v1
  kind: Gateway
  metadata:
    name: ha-failover-gateway
  spec:
    gatewayClassName: tigera-gateway-class
    listeners:
      - name: http
        protocol: HTTP
        port: 80
  EOF
  ```

#### 4. Create the BackendTrafficPolicy with a passive health check setting to detect an transient errors
  ```
  kubectl apply -f - <<EOF
  apiVersion: gateway.envoyproxy.io/v1alpha1
  kind: BackendTrafficPolicy
  metadata:
    name: passive-health-check
  spec:
    targetRefs:
      - group: gateway.networking.k8s.io
        kind: HTTPRoute
        name: ha-failover 
    healthCheck:
      passive:
        baseEjectionTime: 10s
        interval: 2s
        maxEjectionPercent: 100
        consecutive5XxErrors: 1 
        consecutiveGatewayErrors: 0
        consecutiveLocalOriginFailures: 1
        splitExternalLocalOriginErrors: false
  EOF
  ```

#### 5. Create the HTTPRoute that can route to both backends

  ```
  kubectl apply -f - <<EOF
  apiVersion: gateway.networking.k8s.io/v1
  kind: HTTPRoute
  metadata:
    name: ha-failover
    namespace: default
  spec:
    hostnames:
    - www.example.com
    parentRefs:
    - group: gateway.networking.k8s.io
      kind: Gateway
      name: ha-failover-gateway 
      namespace: default
    rules:
    - backendRefs:
      - group: gateway.envoyproxy.io
        kind: Backend
        name: active
        namespace: default
        port: 3000
      - group: gateway.envoyproxy.io
        kind: Backend
        name: passive 
        namespace: default
        port: 3000
      matches:
      - path:
          type: PathPrefix
          value: /test
  EOF
  ```

#### 4. Wait for 30 seconds to allow services and gateway to be ready
sleep 30

#### 5. Retrieve the external IP of the Envoy Gateway

  ```
  export GATEWAY_HA_DEMO=$(kubectl get gateway/ha-failover-gateway -o jsonpath='{.status.addresses[0].value}')
  ```

#### 6. Test

From the bastion, send 10 requests.

  ```
  for i in {1..10}; do curl --verbose --header "Host: www.example.com" http://$GATEWAY_HA_DEMO/test 2>/dev/null | jq .pod; done
  ```

As a result, You should see that they all go to the active backend:

  ```
  "active-8cf8787d5-klpgv"
  "active-8cf8787d5-klpgv"
  "active-8cf8787d5-klpgv"
  "active-8cf8787d5-klpgv"
  "active-8cf8787d5-klpgv"
  "active-8cf8787d5-klpgv"
  "active-8cf8787d5-klpgv"
  "active-8cf8787d5-klpgv"
  "active-8cf8787d5-klpgv"
  "active-8cf8787d5-klpgv"
  ```

Lets simulate a failure in the active backend by changing the server listening port to 5000:

  ```
  kubectl apply -f - <<EOF
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    name: active
  spec:
    replicas: 1
    selector:
      matchLabels:
        app: active
        version: v1
    template:
      metadata:
        labels:
          app: active
          version: v1
      spec:
        containers:
          - image: gcr.io/k8s-staging-gateway-api/echo-basic:v20231214-v1.0.0-140-gf544a46e
            imagePullPolicy: IfNotPresent
            name: active 
            ports:
              - containerPort: 3000
            env:
              - name: HTTP_PORT
                value: "5000"
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

From the bastion, send 10 requests again.

  ```
  for i in {1..10}; do curl --verbose --header "Host: www.example.com" http://$GATEWAY_HA_DEMO/test 2>/dev/null | jq .pod; done
  ```

As a result, You should see that they all go to the active backend:

  ```
  "passive-f894bd7cf-8gkqr"
  "passive-f894bd7cf-8gkqr"
  "passive-f894bd7cf-8gkqr"
  "passive-f894bd7cf-8gkqr"
  "passive-f894bd7cf-8gkqr"
  "passive-f894bd7cf-8gkqr"
  "passive-f894bd7cf-8gkqr"
  "passive-f894bd7cf-8gkqr"
  "passive-f894bd7cf-8gkqr"
  ```

### Clean-up

#### 1. Delete apps, services, backends, HTTPRoute, BackendTrafficPolicy and Gateway

  ```
  kubectl delete backend active passive
  kubectl delete service active passive
  kubectl delete deployment active passive
  kubectl delete gateway ha-failover-gateway
  kubectl delete BackendTrafficPolicy passive-health-check
  kubectl delete HTTPRoute ha-failover
  ```

===
> **Congratulations! You have completed `Calico Ingress Gateway Workshop - High Availability & Failover`!**

---
**Credits:** Portions of this guide are based on or derived from the [Envoy Gateway documentation](https://gateway.envoyproxy.io/docs/tasks/traffic/failover/).