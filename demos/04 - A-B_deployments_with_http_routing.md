# Calico Ingress Gateway - A/B Deployment with HTTP Routing

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

TCP routing is used when applications communicate over raw TCP connections rather than HTTP, such as databases (PostgreSQL, MySQL), message brokers (Redis, MQTT), or custom TCP-based protocols. In Kubernetes, managing access to these services externally—especially with advanced traffic control or multi-tenant routing—requires TCP-aware solutions.

Envoy Gateway supports TCP routing via Kubernetes Gateway API (e.g., `TCPRoute`), enabling layer 4 (L4) routing for TCP traffic. This allows you to define rules that forward TCP connections to specific backends (like a DB service) based on port or other connection parameters, offering centralized, declarative control over TCP traffic across services.

---

### High Level Tasks

- Create four services example, foo, bar and bar-canary, which are bound to example-backend, foo-backend, bar-backend and bar-canary-backend deployments.
- Create a Gateway resource
- Create three TCPRoutes example-route, foo-route and bar-route with different hostnames, rules and paths. Note that the bar-route will have 2 separate backendRefs for bar and bar-canary services
- Retrieve the external IP of the Gateway and **test**

### Diagram

Coming Soon in v2


### Demo

In this example, we have one Gateway resource and two TCPRoute resources that distribute the traffic with the following rules:
- All TCP streams on port 8088 of the Gateway are forwarded to port 3001 of `foo` Kubernetes Service
- All TCP streams on port 8089 of the Gateway are forwarded to port 3002 of `bar` Kubernetes Service
- Two TCP listeners will be applied to the Gateway in order to route them to two separate backend TCPRoutes, note that the protocol set for the listeners on the Gateway is TCP.

#### 1. Create four services `example`, `foo`, `bar` and `bar-canary`, which are bound to `example-backend`, `foo-backend`, `bar-backend` and `bar-canary-backend` deployments.

  ```
  cat <<EOF | kubectl apply -f -
  apiVersion: v1
  kind: Service
  metadata:
    name: example-svc
    labels:
      example: http-routing
  spec:
    ports:
      - name: http
        port: 8080
        targetPort: 3000
    selector:
      app: example-backend
  ---
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    name: example-backend
    labels:
      app: example-backend
      example: http-routing
  spec:
    replicas: 1
    selector:
      matchLabels:
        app: example-backend
    template:
      metadata:
        labels:
          app: example-backend
      spec:
        containers:
          - name: example-backend
            image: gcr.io/k8s-staging-gateway-api/echo-basic:v20231214-v1.0.0-140-gf544a46e
            env:
              - name: POD_NAME
                valueFrom:
                  fieldRef:
                    fieldPath: metadata.name
              - name: NAMESPACE
                valueFrom:
                  fieldRef:
                    fieldPath: metadata.namespace
            resources:
              requests:
                cpu: 10m
  ---
  apiVersion: v1
  kind: Service
  metadata:
    name: foo-svc
    labels:
      example: http-routing
  spec:
    ports:
      - name: http
        port: 8080
        targetPort: 3000
    selector:
      app: foo-backend
  ---
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    name: foo-backend
    labels:
      app: foo-backend
      example: http-routing
  spec:
    replicas: 1
    selector:
      matchLabels:
        app: foo-backend
    template:
      metadata:
        labels:
          app: foo-backend
      spec:
        containers:
          - name: foo-backend
            image: gcr.io/k8s-staging-gateway-api/echo-basic:v20231214-v1.0.0-140-gf544a46e
            env:
              - name: POD_NAME
                valueFrom:
                  fieldRef:
                    fieldPath: metadata.name
              - name: NAMESPACE
                valueFrom:
                  fieldRef:
                    fieldPath: metadata.namespace
            resources:
              requests:
                cpu: 10m
  ---
  apiVersion: v1
  kind: Service
  metadata:
    name: bar-svc
    labels:
      example: http-routing
  spec:
    ports:
      - name: http
        port: 8080
        targetPort: 3000
    selector:
      app: bar-backend
  ---
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    name: bar-backend
    labels:
      app: bar-backend
      example: http-routing
  spec:
    replicas: 1
    selector:
      matchLabels:
        app: bar-backend
    template:
      metadata:
        labels:
          app: bar-backend
      spec:
        containers:
          - name: bar-backend
            image: gcr.io/k8s-staging-gateway-api/echo-basic:v20231214-v1.0.0-140-gf544a46e
            env:
              - name: POD_NAME
                valueFrom:
                  fieldRef:
                    fieldPath: metadata.name
              - name: NAMESPACE
                valueFrom:
                  fieldRef:
                    fieldPath: metadata.namespace
            resources:
              requests:
                cpu: 10m
  ---
  apiVersion: v1
  kind: Service
  metadata:
    name: bar-canary-svc
    labels:
      example: http-routing
  spec:
    ports:
      - name: http
        port: 8080
        targetPort: 3000
    selector:
      app: bar-canary-backend
  ---
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    name: bar-canary-backend
    labels:
      app: bar-canary-backend
      example: http-routing
  spec:
    replicas: 1
    selector:
      matchLabels:
        app: bar-canary-backend
    template:
      metadata:
        labels:
          app: bar-canary-backend
      spec:
        containers:
          - name: bar-canary-backend
            image: gcr.io/k8s-staging-gateway-api/echo-basic:v20231214-v1.0.0-140-gf544a46e
            env:
              - name: POD_NAME
                valueFrom:
                  fieldRef:
                    fieldPath: metadata.name
              - name: NAMESPACE
                valueFrom:
                  fieldRef:
                    fieldPath: metadata.namespace
            resources:
              requests:
                cpu: 10m
  EOF
  ```

#### 2. Create a Gateway resource named "http-routing-gateway" using the "tigera-gateway-class" which will listen always on port 80

  ```
  kubectl apply -f - <<EOF
  apiVersion: gateway.networking.k8s.io/v1
  kind: Gateway
  metadata:
    name: http-routing-gateway
  spec:
    gatewayClassName: tigera-gateway-class
    listeners:
    - name: http
      protocol: HTTP
      port: 80
  EOF
  ```

#### 3. Create three TCPRoutes `example-route`, `foo-route` and `bar-route` with different `hostnames`, `rules` and `paths`. Note that the `bar-route` will have 2 separate `backendRefs` for `bar` and `bar-canary` services:
  ```
  kubectl apply -f - <<EOF
  apiVersion: gateway.networking.k8s.io/v1
  kind: HTTPRoute
  metadata:
    name: example-route
    labels:
      example: http-routing
  spec:
    parentRefs:
      - name: http-routing-gateway
    hostnames:
      - "example.com"
    rules:
      - backendRefs:
          - name: example-svc
            port: 8080
  ---
  apiVersion: gateway.networking.k8s.io/v1
  kind: HTTPRoute
  metadata:
    name: foo-route
    labels:
      example: http-routing
  spec:
    parentRefs:
      - name: http-routing-gateway
    hostnames:
      - "foo.example.com"
    rules:
      - matches:
          - path:
              type: PathPrefix
              value: /login
        backendRefs:
          - name: foo-svc
            port: 8080
  ---
  apiVersion: gateway.networking.k8s.io/v1
  kind: HTTPRoute
  metadata:
    name: bar-route
    labels:
      example: http-routing
  spec:
    parentRefs:
      - name: http-routing-gateway
    hostnames:
      - "bar.example.com"
    rules:
      - matches:
          - headers:
              - type: Exact
                name: env
                value: canary
        backendRefs:
          - name: bar-canary-svc
            port: 8080
      - backendRefs:
          - name: bar-svc
            port: 8080
  EOF
  ```

#### 4. Wait for 30 seconds to allow services and gateway to be ready
  
  ```
  sleep 30
  ```

#### 5. Retrieve the external IP of the Gateway

  ```
  export GATEWAY_HTTP_DEMO=$(kubectl get gateway/http-routing-gateway -o jsonpath='{.status.addresses[0].value}')
  echo "GATEWAY_HTTP_DEMO is: $GATEWAY_HTTP_DEMO"
  ```

#### 6. Test

***6.1*** - Test HTTP routing to the `example` backend:

  ```
  curl -vvv --header "Host: example.com" "http://${GATEWAY_HTTP_DEMO}/"
  ```

  A `200` status code should be returned and the body should include `"pod": "example-backend-*"` indicating the traffic was routed to the example backend service. If you change the hostname to a hostname not represented in any of the `HTTPRoutes`, e.g. `“www.example.com”`, the HTTP traffic will not be routed and a `404` should be returned.

  Sample of output:
  ```
  *   Trying 10.10.10.23:80...
  * TCP_NODELAY set
  * Connected to 10.10.10.23 (10.10.10.23) port 80 (#0)
  > GET / HTTP/1.1
  > Host: example.com
  > User-Agent: curl/7.68.0
  > Accept: */*
  >
  * Mark bundle as not supporting multiuse
  < HTTP/1.1 200 OK
  < content-type: application/json
  < x-content-type-options: nosniff
  < date: Mon, 04 Aug 2025 13:20:47 GMT
  < content-length: 468
  <
  {
  "path": "/",
  "host": "example.com",
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
    "fc4af00b-21ce-4f30-a95a-732a3c751da8"
    ]
  },
  "namespace": "default",
  "ingress": "",
  "service": "",
  "pod": "example-backend-7547c5c689-vrtq5"
  * Connection #0 to host 10.10.10.23 left intact
  ```

***6.2*** - The `foo-route` matches any traffic for `foo.example.com` and applies its routing rules to forward the traffic to the “`foo-svc`” Service. Since there is only one path prefix match for `/login`, only `foo.example.com/login/*` traffic will be forwarded. Test HTTP routing to the `foo-svc` backend.

  ```
  curl -vvv --header "Host: foo.example.com" "http://${GATEWAY_HTTP_DEMO}/login"
  ```

  Sample of output:
  ```
  *   Trying 10.10.10.23:80...
  * TCP_NODELAY set
  * Connected to 10.10.10.23 (10.10.10.23) port 80 (#0)
  > GET /login HTTP/1.1
  > Host: foo.example.com
  > User-Agent: curl/7.68.0
  > Accept: */*
  >
  * Mark bundle as not supporting multiuse
  < HTTP/1.1 200 OK
  < content-type: application/json
  < x-content-type-options: nosniff
  < date: Mon, 04 Aug 2025 13:23:39 GMT
  < content-length: 473
  <
  {
  "path": "/login",
  "host": "foo.example.com",
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
    "d000c319-64e2-4a1f-8de5-a906ed824b92"
    ]
  },
  "namespace": "default",
  "ingress": "",
  "service": "",
  "pod": "foo-backend-678946fbbf-kw4qk"
  * Connection #0 to host 10.10.10.23 left intact
  ```

  A `200` status code should be returned and the body should include `"pod": "foo-backend-*" `indicating the traffic was routed to the foo backend service. Traffic to any other paths that do not begin with `/login` will not be matched by this HTTPRoute. Test this by removing `/login` from the request.

  ```
  curl -vvv --header "Host: foo.example.com" "http://${GATEWAY_HTTP_DEMO}/"
  ```

  The HTTP traffic will not be routed and a `404` should be returned. Sample of output:
  ```
  *   Trying 10.10.10.23:80...
  * TCP_NODELAY set
  * Connected to 10.10.10.23 (10.10.10.23) port 80 (#0)
  > GET / HTTP/1.1
  > Host: foo.example.com
  > User-Agent: curl/7.68.0
  > Accept: */*
  >
  * Mark bundle as not supporting multiuse
  < HTTP/1.1 404 Not Found
  < date: Mon, 04 Aug 2025 13:26:01 GMT
  < content-length: 0
  <
  * Connection #0 to host 10.10.10.23 left intact
  ```

***6.3*** - Similarly, the `bar-route` HTTPRoute matches traffic for `bar.example.com`. All traffic for this hostname will be evaluated against the routing rules. The most specific match will take precedence which means that any traffic with the `env:canary` header will be forwarded to `bar-svc-canary` and if the header is missing or not `canary` then it’ll be forwarded to `bar-svc`. Test HTTP routing to the `bar-svc` backend.

  ```
  curl -vvv --header "Host: bar.example.com" "http://${GATEWAY_HTTP_DEMO}/"
  ```


  A `200` status code should be returned and the body should include `"pod": "bar-backend-*"` indicating the traffic was routed to the `bar` backend service. Sample of output:
  ```
  *   Trying 10.10.10.23:80...
  * TCP_NODELAY set
  * Connected to 10.10.10.23 (10.10.10.23) port 80 (#0)
  > GET / HTTP/1.1
  > Host: bar.example.com
  > User-Agent: curl/7.68.0
  > Accept: */*
  >
  * Mark bundle as not supporting multiuse
  < HTTP/1.1 200 OK
  < content-type: application/json
  < x-content-type-options: nosniff
  < date: Mon, 04 Aug 2025 13:27:50 GMT
  < content-length: 467
  <
  {
  "path": "/",
  "host": "bar.example.com",
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
    "19e58d8f-c81a-42d8-b4d1-b1ada15991f3"
    ]
  },
  "namespace": "default",
  "ingress": "",
  "service": "",
  "pod": "bar-backend-75959c65c-h4l7b"
  * Connection #0 to host 10.10.10.23 left intact
  ```

***6.4*** - Test HTTP routing to the bar-canary-svc backend by adding the env: canary header to the request.

  ```
  curl -vvv --header "Host: bar.example.com" --header "env: canary" "http://${GATEWAY_HTTP_DEMO}/"
  ```

  A `200` status code should be returned and the body should include `"pod": "bar-canary-backend-*"` indicating the traffic was routed to the `bar-canary` backend service. Sample of output:
  ```
  *   Trying 10.10.10.23:80...
  * TCP_NODELAY set
  * Connected to 10.10.10.23 (10.10.10.23) port 80 (#0)
  > GET / HTTP/1.1
  > Host: bar.example.com
  > User-Agent: curl/7.68.0
  > Accept: */*
  > env: canary
  >
  * Mark bundle as not supporting multiuse
  < HTTP/1.1 200 OK
  < content-type: application/json
  < x-content-type-options: nosniff
  < date: Mon, 04 Aug 2025 13:31:14 GMT
  < content-length: 503
  <
  {
  "path": "/",
  "host": "bar.example.com",
  "method": "GET",
  "proto": "HTTP/1.1",
  "headers": {
    "Accept": [
    "*/*"
    ],
    "Env": [
    "canary"
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
    "72dbfef8-b2a6-49b7-b798-e0c7ba89f84e"
    ]
  },
  "namespace": "default",
  "ingress": "",
  "service": "",
  "pod": "bar-canary-backend-5b76958f4f-7vppt"
  * Connection #0 to host 10.10.10.23 left intact
  ```

### Clean-up

#### 1. Delete apps, services, HTTPRoutes and Gateway

  ```
  kubectl delete service bar-canary-svc bar-svc foo-svc example-svc
  kubectl delete deployment bar-backend bar-canary-backend example-backend foo-backend
  kubectl delete gateway http-routing-gateway
  kubectl delete HTTPRoute example-route foo-route bar-route
  ```

===
> **Congratulations! You have completed `Calico Ingress Gateway Workshop - A/B Deployment with HTTP Routing'`!**

---
**Credits:** Portions of this guide are based on or derived from the [Envoy Gateway documentation](https://gateway.envoyproxy.io/docs/tasks/traffic/http-routing/).
