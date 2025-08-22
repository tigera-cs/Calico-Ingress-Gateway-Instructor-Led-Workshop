# Calico Ingress Gateway - Sticky session with Session Persistence Envoy Route

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

Session Persistence allows client requests to be consistently routed to the same backend service instance. This is useful in many scenarios, such as when an application needs to maintain state across multiple requests.

In Kubernetes with Envoy Gateway, session persistence can be achieved using a session identifier (e.g., a cookie or header). This ensures that a client’s traffic is consistently routed to the same pod for the duration of the session, preserving user context and improving user experience. 

Use session persistence when your app is stateful and you need explicit control over session lifecycle (e.g., login sessions, sticky shopping carts).

---

### High Level Tasks

### Diagram

Coming Soon in v2

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