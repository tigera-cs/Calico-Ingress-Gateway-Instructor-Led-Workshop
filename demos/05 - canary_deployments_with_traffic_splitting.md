# Calico Ingress Gateway - Canary Deployment with Traffic Splitting

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

A canary deployment is a progressive release strategy where a new version of an application is gradually rolled out to a small subset of users before a full release. This allows teams to monitor for issues and roll back if necessary with minimal impact.

In Kubernetes using Envoy Gateway, canary deployments can be implemented through traffic splitting. This means you configure routing rules (e.g., via HTTPRoute) to send a specific percentage of traffic to the new version (the canary) and the rest to the stable version. Over time, the percentage can be increased as confidence in the new version grows.

---

### High Level Tasks

- Deploy version1 and version2 of an application, in a dedicated namespace
- Deploy a gateway resource
- Define an HTTPRoute to split traffic between app-v1 and app-v2
- Create a ReferenceGrant to allow HTTPRoute in the default namespace to reference services in my-app namespace
- Retrieve the external IP of the Envoy Gateway and **test**

### Diagram

![Canary Deployment Diagram](etc/canary_diagram.png)

### Demo

#### 1. Create a new namespace called "my-app" to isolate resources
  ```
  cat << EOF | kubectl create -f -
  apiVersion: v1
  kind: Namespace
  metadata:
    name: my-app
  EOF
  ```

#### 2. Deploy version 1 of the app
***2.1*** - Create a ConfigMap with an HTML file for version 1
  ```
  cat << EOF | kubectl apply -f -
  apiVersion: v1
  kind: ConfigMap
  metadata:
    name: app-v1-html
    namespace: my-app
  data:
    index.html: |
      <html><body><h1>App Version 1</h1></body></html>
  EOF
  ```

***2.2*** - Deploy an Nginx container serving the HTML file
  ```
  cat << EOF | kubectl apply -f -
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    name: app-v1
    namespace: my-app
  spec:
    replicas: 1
    selector:
      matchLabels:
        app: my-app
        version: v1
    template:
      metadata:
        labels:
          app: my-app
          version: v1
      spec:
        containers:
          - name: nginx
            image: nginx
            ports:
              - containerPort: 80
            volumeMounts:
              - name: html
                mountPath: /usr/share/nginx/html
        volumes:
          - name: html
            configMap:
              name: app-v1-html
  EOF
  ```

***2.3*** - Expose the deployment as a Service
  ```
  cat << EOF | kubectl apply -f -
  apiVersion: v1
  kind: Service
  metadata:
    name: app-v1
    namespace: my-app
  spec:
    selector:
      app: my-app
      version: v1
    ports:
      - port: 80
        targetPort: 80
  EOF
  ```

#### 3. Deploy version 2 of the app following the same pattern as version 1

***3.1*** - Create a ConfigMap with an HTML file for version 2
  ```
  cat << EOF | kubectl apply -f -
  apiVersion: v1
  kind: ConfigMap
  metadata:
    name: app-v2-html
    namespace: my-app
  data:
    index.html: |
      <html><body><h1>App Version 2</h1></body></html>
  EOF
  ```

***3.2*** - Deploy an Nginx container serving the HTML file
  ```
  cat << EOF | kubectl apply -f -
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    name: app-v2
    namespace: my-app
  spec:
    replicas: 1
    selector:
      matchLabels:
        app: my-app
        version: v2
    template:
      metadata:
        labels:
          app: my-app
          version: v2
      spec:
        containers:
          - name: nginx
            image: nginx
            ports:
              - containerPort: 80
            volumeMounts:
              - name: html
                mountPath: /usr/share/nginx/html
        volumes:
          - name: html
            configMap:
              name: app-v2-html
  EOF
  ```

***3.3*** - Expose the deployment as a Service
  ```
  cat << EOF | kubectl apply -f -
  apiVersion: v1
  kind: Service
  metadata:
    name: app-v2
    namespace: my-app
  spec:
    selector:
      app: my-app
      version: v2
    ports:
      - port: 80
        targetPort: 80
  EOF
  ```

#### 4. Create a Gateway resource named "canary-deployment-gateway" using the "tigera-gateway-class"

  ```
  kubectl apply -f - <<EOF
  apiVersion: gateway.networking.k8s.io/v1
  kind: Gateway
  metadata:
    name: canary-deployment-gateway
  spec:
    gatewayClassName: tigera-gateway-class
    listeners:
      - name: http
        protocol: HTTP
        port: 80
  EOF
  ```

#### 5. Define an HTTPRoute to split traffic between app-v1 and app-v2

The HTTPRoute below routes 80% of requests to app-v1 and 20% to app-v2

  ```
  cat << EOF | kubectl apply -f -
  apiVersion: gateway.networking.k8s.io/v1
  kind: HTTPRoute
  metadata:
    name: traffic-splitting
  spec:
    parentRefs:
      - name: canary-deployment-gateway
        namespace: default
    rules:
      - matches:
          - path:
              type: PathPrefix
              value: /
        backendRefs:
          - name: app-v1
            namespace: my-app
            port: 80
            weight: 80
          - name: app-v2
            namespace: my-app
            port: 80
            weight: 20
  EOF
  ```

#### 6. Create a ReferenceGrant to allow HTTPRoute in the `default` namespace to reference services in `my-app` namespace

  ```
  kubectl apply -f - <<EOF
  apiVersion: gateway.networking.k8s.io/v1beta1
  kind: ReferenceGrant
  metadata:
    name: my-app
    namespace: my-app
  spec:
    from:
    - group: gateway.networking.k8s.io
      kind: HTTPRoute
      namespace: default
    to:
    - group: ""
      kind: Service
  EOF
  ```

#### 7. Wait for 30 seconds to allow services and gateway to be ready

  ```
  sleep 30
  ```

#### 8. Retrieve the external IP of the Envoy Gateway

  ```
  EXTERNAL_IP=$(kubectl get service -n tigera-gateway -l gateway.envoyproxy.io/owning-gateway-name=canary-deployment-gateway \
    -o jsonpath='{.items[0].status.loadBalancer.ingress[0].ip}')
  echo "Envoy Gateway External IP: $EXTERNAL_IP"
  ```

#### 9. Test

From the bastion, continuously send requests to the external IP and print the response HTML header to verify traffic splitting

  ```
  while true; do curl -s http://$EXTERNAL_IP/ | grep "<h1>"; sleep 1; done
  ```

You should see the majority of responses coming from `App Version 1`.

### Clean-up

#### 1. Delete apps, services, HTTPRoute and Gateway

  ```
  kubectl delete gateway canary-deployment-gateway
  kubectl delete httproute traffic-splitting
  kubectl delete namespace my-app
  ```

===
> **Congratulations! You have completed `Calico Ingress Gateway Workshop - Canary Deployment with Traffic Splitting'`!**

---
**Credits:** Portions of this guide are based on or derived from the [Envoy Gateway documentation](https://gateway.envoyproxy.io/docs/tasks/traffic/http-traffic-splitting/).