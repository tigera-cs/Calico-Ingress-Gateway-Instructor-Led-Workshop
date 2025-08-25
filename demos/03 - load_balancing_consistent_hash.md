# Calico Ingress Gateway - Consistent Hash Load Balancing

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

A typical use case for consistent hash load balancing is routing requests from the same client to the same backend pod to maintain session affinity—for example, in a shopping cart or user session scenario.

In Kubernetes with Envoy Gateway, consistent hashing can be configured so that requests are distributed based on a hash of a key (like a cookie, HTTP header, or client IP). Envoy calculates the hash and routes each request to a specific backend, ensuring that the same client consistently lands on the same pod, improving cache hit rates and preserving session state.

---

### High Level Tasks

- Create a deployment named Backend which we will use to test consistent hash load balancing. The deployment will have 10 replicas.
- Create a Gateway resource
- Create the backend traffic policy and the http route to configure the load balancing. For the sticky load balancing, we will use cookies
- Retrieve the external IP of the Envoy Gateway and **test**

### Diagram

Coming Soon in v2

### Demo

#### 1. Create a deployment named `Backend` which we will use to test consistent hash load balancing. The deployment will have 10 replicas.

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
    replicas: 10
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

#### 3. Create the backend traffic policy and the http route to configure the load balancing. For the sticky load balancing, we will use cookies
  ```
  cat <<EOF | kubectl apply -f -
  apiVersion: gateway.envoyproxy.io/v1alpha1
  kind: BackendTrafficPolicy
  metadata:
    name: cookie-policy
    namespace: default
  spec:
    targetRefs:
      - group: gateway.networking.k8s.io
        kind: HTTPRoute
        name: cookie-route
    loadBalancer:
      type: ConsistentHash
      consistentHash:
        type: Cookie
        cookie:
          name: FooBar
          ttl: 60s
          attributes:
            SameSite: Strict
  ---
  apiVersion: gateway.networking.k8s.io/v1
  kind: HTTPRoute
  metadata:
    name: cookie-route
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
              value: /cookie
        backendRefs:
          - name: backend
            port: 3000
  EOF
  ```

#### 4. Wait for 30 seconds to allow services and gateway to be ready

  ```
  sleep 30
  ```

#### 5. Retrieve the external IP of the Envoy Gateway

  ```
  export GATEWAY_LB_DEMO=$(kubectl get gateway/load-balancing-gateway -o jsonpath='{.status.addresses[0].value}')
  echo "GATEWAY_LB_DEMO is: $GATEWAY_LB_DEMO"
  ```

#### 6. Test

From the bastion, by sending 10 request with curl to `${GATEWAY_LB_DEMO}/cookie`, you can see that all requests got routed to only one upstream host, since they have same cookie setting.

  ```
  for i in {1..10}; do curl -I --header "Host: www.example.com" --cookie "FooBar=1.2.3.4" http://${GATEWAY_LB_DEMO}/cookie ; sleep 1; done
  ```

You can see the result with this command:

  ```
  kubectl get pods -l app=backend --no-headers -o custom-columns=":metadata.name" | while read -r pod; do echo "$pod: received $(($(kubectl logs $pod | wc -l) - 2)) requests"; done
  ```

Sample of output:
  ```
  backend-765694d47f-6vfl4: received 0 requests
  backend-765694d47f-7lnl9: received 0 requests
  backend-765694d47f-8nf2k: received 0 requests
  backend-765694d47f-8vwcf: received 10 requests
  backend-765694d47f-9qt42: received 0 requests
  backend-765694d47f-c785w: received 0 requests
  backend-765694d47f-cp22v: received 0 requests
  backend-765694d47f-prktn: received 0 requests
  backend-765694d47f-whjnm: received 0 requests
  backend-765694d47f-xgzkj: received 0 requests
  ```

You can try to set different cookie to these requests, the upstream host that receives traffics may vary. The following output happens when you use curl to send another 10 requests with cookie `FooBar: 5.6.7.8`.

  ```
  for i in {1..10}; do curl -I --header "Host: www.example.com" --cookie "FooBar=5.6.7.8" http://${GATEWAY_LB_DEMO}/cookie ; sleep 1; done
  ```

You can see the result with this command:

  ```
  kubectl get pods -l app=backend --no-headers -o custom-columns=":metadata.name" | while read -r pod; do echo "$pod: received $(($(kubectl logs $pod | wc -l) - 2)) requests"; done
  ```

Sample of output:
  ```
  backend-765694d47f-6vfl4: received 10 requests
  backend-765694d47f-7lnl9: received 0 requests
  backend-765694d47f-8nf2k: received 0 requests
  backend-765694d47f-8vwcf: received 10 requests
  backend-765694d47f-9qt42: received 0 requests
  backend-765694d47f-c785w: received 0 requests
  backend-765694d47f-cp22v: received 0 requests
  backend-765694d47f-prktn: received 0 requests
  backend-765694d47f-whjnm: received 0 requests
  backend-765694d47f-xgzkj: received 0 requests
  ```

*NOTE*: It is possible that the same backend pod is used by the gateway. If that's the case, keep trying with a different cookie.

### Clean-up

#### 1. Delete serviceAccount, app, service, HTTPRoute, BackendTrafficPolicy and Gateway

  ```
  kubectl delete ServiceAccount backend
  kubectl delete service backend
  kubectl delete deployment backend
  kubectl delete gateway load-balancing-gateway
  kubectl delete BackendTrafficPolicy cookie-policy
  kubectl delete HTTPRoute cookie-route
  ```

===
> **Congratulations! You have completed `Calico Ingress Gateway Workshop - Consistent Hash Load Balancing`!**

---
**Credits:** Portions of this guide are based on or derived from the [Envoy Gateway documentation](https://gateway.envoyproxy.io/docs/tasks/traffic/load-balancing/#consistent-hash).