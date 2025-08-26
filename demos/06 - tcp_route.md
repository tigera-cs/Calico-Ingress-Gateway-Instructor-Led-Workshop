# Calico Ingress Gateway - TCP Routing

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
- [Calico Ingress Gateway - Introduction](etc/01%20-%20Calico%20Ingress%20Gateway%20-%20Introduction.pdf)
- [Calico Ingress Gateway - Capabilities](etc/02%20%20-%20Calico%20Ingress%20Gateway%20-%20Capabilities.pdf)
- [Calico Ingress Gateway - Migration](etc/03%20-%20Calico%20Ingress%20Gateway%20-%20Migration%20From%20Ingress.pdf)

---

### Overview

TCP routing is used when applications communicate over raw TCP connections rather than HTTP, such as databases (PostgreSQL, MySQL), message brokers (Redis, MQTT), or custom TCP-based protocols. In Kubernetes, managing access to these services externally—especially with advanced traffic control or multi-tenant routing—requires TCP-aware solutions.

Envoy Gateway supports TCP routing via Kubernetes Gateway API (e.g., `TCPRoute`), enabling layer 4 (L4) routing for TCP traffic. This allows you to define rules that forward TCP connections to specific backends (like a DB service) based on port or other connection parameters, offering centralized, declarative control over TCP traffic across services.

---

### High Level Tasks

- Create two services foo and bar, which are bound to backend-1 and backend-2 deployments.
- Create a Gateway resource. The gateway will have 2 listeners: 1 per TCP port.
- Create two TCPRoutes tcp-app-1 and tcp-app-2 with different sectionName.
- Retrieve the external IP of the Envoy Gateway and **test**

### Diagram

Coming Soon in v2

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