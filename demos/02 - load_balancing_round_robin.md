# Calico Ingress Gateway - Round Robin Load Balancing

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

A common use case for round-robin load balancing is distributing incoming traffic evenly across multiple instances of a stateless web application to maximize resource utilization and ensure high availability.

In Kubernetes with Calico Ingress Gateway, round-robin is one of the default load balancing strategies used by Envoy to forward requests to backend pods. When configured (e.g., via `BackendRef` in `HTTPRoute` or a `Service`), Envoy will cycle through available backend endpoints, sending each new request to the next pod in line, ensuring a balanced distribution without overloading any single instance.

---

### High Level Tasks

- Create a deployment named Backend which we will use to test round robin load balancing. The deployment will have 4 replicas.
- Create a Gateway resource
- Create the backend traffic policy and the http route to configure the load balancing
- Retrieve the external IP of the gateway and **test**

### Diagram

Coming Soon in v2

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

#### 4. Wait for 30 seconds to allow services and gateway to be ready

  ```
  sleep 30
  ```

#### 5. Retrieve the external IP of the gateway

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