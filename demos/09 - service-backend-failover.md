# Calico Ingress Gateway - High Availability & Failover

### Table of Contents

* [Welcome!](#welcome)
* [Overview](#overview)
* [High Level Tasks](#high-level-tasks)
* [Diagram](#diagram)
* [Demo](#demo)
* [Clean-up](#clean-up)


### Welcome!

Welcome to the **Calico Ingress Gateway Instructor Led Workshop**. 

The Calico Ingress Gateway Workshop aims to explain the kubernetes' and IngressAPI native limitations, the differences between IngressAPI and GatewayAPI and the most common use cases where Calico Ingress Gateway can solve.

We hope you enjoyed the presentation! Feel free to download the slides:
- [Calico Ingress Gateway - Introduction](etc/01%20-%20Calico%20Ingress%20Gateway%20-%20Introduction%20-%20WIP.pptx)
- [Calico Ingress Gateway - Capabilities](etc/02%20%20-%20Calico%20Ingress%20Gateway%20-%20Capabilities%20-%20WIP.pptx)
- [Calico Ingress Gateway - Migration](etc/03%20-%20Calico%20Ingress%20Gateway%20-%20Migration%20From%20Ingress%20-%20WIP.pptx)

---

### Overview

Active-passive failover in an API gateway setup is like having a backup plan in place to keep things running smoothly if something goes wrong. Here’s why it’s valuable:
- Staying Online: When the main (or “active”) backend has issues or goes offline, the fallback (or “passive”) backend is ready to step in instantly. This helps keep your API accessible and your services running, so users don’t even notice any interruptions.
- Automatic Switch Over: If a problem occurs, the system can automatically switch traffic over to the fallback backend. This avoids needing someone to jump in and fix things manually, which could take time and might even lead to mistakes.
- Lower Costs: In an active-passive setup, the fallback backend doesn’t need to work all the time—it’s just on standby. This can save on costs (like cloud egress costs) compared to setups where both backend are running at full capacity.
- Peace of Mind with Redundancy: Although the fallback backend isn’t handling traffic daily, it’s there as a safety net. If something happens with the primary backend, the backup can take over immediately, ensuring your service doesn’t skip a beat.

In Kubernetes with Envoy Gateway, failover is handled by configuring multiple backend references with health checks and prioritized or weighted routing. If the primary backend fails (e.g., fails readiness or liveness probes), Envoy automatically routes traffic to a secondary (failover) backend, keeping the service online without manual intervention.

---

### High Level Tasks

- Create deployments and services for active and passive applications which we will use to test the failover
- Create the backendAPI resources that are used to represent the active backend and passive backend
- Create a Gateway resource
- Create the `BackendTrafficPolicy` with a passive health check setting to detect an transient errors
- Create the `HTTPRoute` that can route to both backends
- Retrieve the external IP of the Envoy Gateway and **test**

### Diagram

Coming Soon in v2

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

#### 4. Create the `BackendTrafficPolicy` with a passive health check setting to detect an transient errors
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

  ```
  sleep 30
  ```

#### 5. Retrieve the external IP of the Envoy Gateway

  ```
  export GATEWAY_HA_DEMO=$(kubectl get gateway/ha-failover-gateway -o jsonpath='{.status.addresses[0].value}')
  echo "GATEWAY_HA_DEMO is: $GATEWAY_HA_DEMO"
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