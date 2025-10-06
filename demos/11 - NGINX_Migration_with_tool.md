# Calico Ingress Gateway - Migration from NGINX Ingress with ingress2gateway

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

Organizations migrate from the older Ingress API to the newer Gateway API because Gateway API offers more flexibility, better extensibility, and clearer separation of concerns for managing traffic routing in Kubernetes. 

In real-world scenarios, authentication mechanisms (like OIDC, JWT, Basic Auth) are often enforced at the ingress layer to secure applications. To simulate this during migration, They can be implemented in Calico Ingress Gateway using Envoy’s `SecurityPolicy` or authentication filters, ensuring that security controls are preserved while adopting the more powerful Gateway API.

***Because OIDC is more complex to setup, for the scope of the demo, we are going to use Basic Auth. However, for a more realistic authentication such as OIDC, steps are the same.***

---

### High Level Tasks

- Deploy an application version1 & version2 and expose them internally with ClusterIP Services.
- Create a basic-auth secret using htpasswd for user authentication.
- Configure an NGINX Ingress with:
  - Annotations to enable basic auth, protecting access to the application
  - Annotations to enable canary deployment.
- Use curl commands to show how requests are:
  - Rejected without credentials
  - Accepted with the correct username/password
  - Split between version1 and version2
- Deploy a Gateway and HTTPRoute using the Gateway API (instead of Ingress) with a corresponding security policy for basic auth and traffic splitting.
- Show how to access the application through the Gateway API with authentication, illustrating a migration path from Ingress to Gateway API while maintaining security and canary deployment.

### Diagram

Coming Soon in v2

### Demo

This demo showcases how to deploy a simple app deployment on Kubernetes with basic HTTP authentication and canary deployment (version1 & version2) configured through both an NGINX Ingress and a Gateway API setup.

It’s a full example of securing an application with authentication and canary deployment, via two Kubernetes traffic routing methods.

#### 1. Deploy version 1 of the app

***IMPORTANT:*** Make sure that you are connected to the right context:

  ```
  kubectl config use-context kubernetes-admin
  ```

***1.1*** - Create a ConfigMap with an HTML file for version 1
  ```
  cat << EOF | kubectl apply -f -
  apiVersion: v1
  kind: ConfigMap
  metadata:
    name: app-v1-html
  data:
    index.html: |
      <html><body><h1>App Version 1</h1></body></html>
  EOF
  ```

***1.2*** - Deploy an Nginx container serving the HTML file
  ```
  cat << EOF | kubectl apply -f -
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    name: app-v1
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

***1.3*** - Expose the deployment as a Service
  ```
  cat << EOF | kubectl apply -f -
  apiVersion: v1
  kind: Service
  metadata:
    name: app-v1
  spec:
    selector:
      app: my-app
      version: v1
    ports:
      - port: 8080
        targetPort: 80
  EOF
  ```

#### 2. Deploy version 2 of the app following the same pattern as version 1

***2.1*** - Create a ConfigMap with an HTML file for version 2
  ```
  cat << EOF | kubectl apply -f -
  apiVersion: v1
  kind: ConfigMap
  metadata:
    name: app-v2-html
  data:
    index.html: |
      <html><body><h1>App Version 2</h1></body></html>
  EOF
  ```

***2.2*** - Deploy an Nginx container serving the HTML file
  ```
  cat << EOF | kubectl apply -f -
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    name: app-v2
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

***2.3*** - Expose the deployment as a Service
  ```
  cat << EOF | kubectl apply -f -
  apiVersion: v1
  kind: Service
  metadata:
    name: app-v2
  spec:
    selector:
      app: my-app
      version: v2
    ports:
      - port: 8080
        targetPort: 80
  EOF
  ```

#### 3. Create an user/password pair which we will use for the authentication for both IngressAPI and GatewayAPI, via a kubernetes `secret`

  ```
  htpasswd -bcs auth foo bar
  cp auth .htpasswd
  kubectl create secret generic basic-auth --from-file=auth --from-file=.htpasswd
  ```

#### 4. Create an `Ingress` for the `app-v1` service, which requires authentication

  ```
  HOSTNAME="echoserver.myingress.com"

  cat << EOF | kubectl apply -f -
  apiVersion: networking.k8s.io/v1
  kind: Ingress
  metadata:
    name: ingress-with-auth
    annotations:
      nginx.ingress.kubernetes.io/auth-type: basic
      nginx.ingress.kubernetes.io/auth-secret: basic-auth
      nginx.ingress.kubernetes.io/auth-realm: 'Authentication Required - demo'
      kubernetes.io/ingress.class: "nginx"
  spec:
    rules:
    - host: "$HOSTNAME"
      http:
        paths:
        - path: /
          pathType: Prefix
          backend:
            service:
              name: app-v1
              port:
                number: 8080
  EOF
  ```

#### 5. Create an `Ingress` for the `app-v2` service, which requires authentication. This ingress will have the `canary` annotations

  ```
  HOSTNAME="echoserver.myingress.com"

  cat << EOF | kubectl apply -f -
  apiVersion: networking.k8s.io/v1
  kind: Ingress
  metadata:
    name: ingress-with-auth-canary
    annotations:
      nginx.ingress.kubernetes.io/auth-type: basic
      nginx.ingress.kubernetes.io/auth-secret: basic-auth
      nginx.ingress.kubernetes.io/auth-realm: 'Authentication Required - demo'
      kubernetes.io/ingress.class: "nginx"
      nginx.ingress.kubernetes.io/canary: "true"
      nginx.ingress.kubernetes.io/canary-weight: "20"
  spec:
    rules:
    - host: "$HOSTNAME"
      http:
        paths:
        - path: /
          pathType: Prefix
          backend:
            service:
              name: app-v2
              port:
                number: 8080
  EOF
  ```

#### 5. TEST Ingress

***5.1*** - Save the hostname of the Kubernetes control plane of this lab, which is tied to the NGINX Controller:

  ```
  export HOSTNAME="echoserver.myingress.com"
  export INGRESS_URL="$(kubectl cluster-info | grep control | awk -F ":" '{print $2}' | sed "s/\/\///g")"
  ```

***5.2*** - Test the connection without the user:

  ```
  while true; do curl -s -kv -H "Host: $HOSTNAME" https://$INGRESS_URL/ | grep "<h1>"; sleep 1; done
  ```

  You should get an `401 - Authentication Required` error:

  ```
    * Connection #0 to host 172.212.46.95 left intact
    <center><h1>401 Authorization Required</h1></center>
    *   Trying 172.212.46.95:80...
    * Connected to 172.212.46.95 (172.212.46.95) port 80
    > GET / HTTP/1.1
    > Host: echoserver.myingress.com
    > User-Agent: curl/8.7.1
    > Accept: */*
    >
    * Request completely sent off
    < HTTP/1.1 401 Unauthorized
    < Date: Tue, 19 Aug 2025 10:48:03 GMT
    < Content-Type: text/html
    < Content-Length: 172
    < Connection: keep-alive
    < WWW-Authenticate: Basic realm="Authentication Required - demo"
  ```

***5.3*** - Test the connection with the user:

  ```
  while true; do curl -s -kv -H "Host: $HOSTNAME" -u foo:bar https://$INGRESS_URL/ | grep "<h1>"; sleep 1; done

  ```

  You should get a `200 OK`:

  ```
    *   Trying 172.212.46.95:80...
    * Connected to 172.212.46.95 (172.212.46.95) port 80
    * Server auth using Basic with user 'foo'
    > GET / HTTP/1.1
    > Host: echoserver.myingress.com
    > Authorization: Basic Zm9vOmJhcg==
    > User-Agent: curl/8.7.1
    > Accept: */*
    >
    * Request completely sent off
    < HTTP/1.1 200 OK
    < Date: Tue, 19 Aug 2025 10:47:49 GMT
    < Content-Type: text/html
    < Content-Length: 49
    < Connection: keep-alive
    < Last-Modified: Tue, 19 Aug 2025 10:37:13 GMT
    < ETag: "68a453d9-31"
    < Accept-Ranges: bytes
    <
    { [49 bytes data]
    * Connection #0 to host 172.212.46.95 left intact
    <html><body><h1>App Version 1</h1></body></html>
  ```

  You should also see the canary deployment in action:

  Command:
  ```
  while true; do curl -s -k -H "Host: $HOSTNAME" -u foo:bar https://$INGRESS_URL/ | grep "<h1>"; sleep 1; done
  ```

  Output:
  ```
    <html><body><h1>App Version 1</h1></body></html>
    <html><body><h1>App Version 1</h1></body></html>
    <html><body><h1>App Version 1</h1></body></html>
    <html><body><h1>App Version 1</h1></body></html>
    <html><body><h1>App Version 1</h1></body></html>
    <html><body><h1>App Version 2</h1></body></html>
    <html><body><h1>App Version 1</h1></body></html>
    <html><body><h1>App Version 1</h1></body></html>
    <html><body><h1>App Version 1</h1></body></html>
    <html><body><h1>App Version 2</h1></body></html>
  ```

#### 6. Use the `ingress2gateway` tool to automatically create the gateway and the `HTTPRoute`

***6.1*** - Get the translated resourses in the `gateway-resources.yaml` file

  ```
  ~/ingress2gateway/ingress2gateway print --providers=ingress-nginx > gateway-resources.yaml
  ```

  The `gateway-resources.yaml` should look like this:

  ```
  apiVersion: gateway.networking.k8s.io/v1
  kind: Gateway
  metadata:
    creationTimestamp: null
    name: nginx
    namespace: default
  spec:
    gatewayClassName: nginx
    listeners:
    - hostname: echoserver.myingress.com
      name: echoserver-myingress-com-http
      port: 80
      protocol: HTTP
  status: {}
  ---
  apiVersion: gateway.networking.k8s.io/v1
  kind: HTTPRoute
  metadata:
    creationTimestamp: null
    name: ingress-with-auth-echoserver-myingress-com
    namespace: default
  spec:
    hostnames:
    - echoserver.myingress.com
    parentRefs:
    - name: nginx
    rules:
    - backendRefs:
      - name: app-v1
        port: 8080
        weight: 80
      - name: app-v2
        port: 8080
        weight: 20
      matches:
      - path:
          type: PathPrefix
          value: /
  status:
    parents: []
  ```

***6.2*** - Edit the file to:
  - If present, remove the initial lines (description of the `ingress2gateway` activity)
  - Replace the `gatewayClassName` from `nginx` to `tigera-gateway-class`

***6.3*** - Note that the Authentication annotation has not been translated in a GatewayAPI resource because the tool is not able to do it. Manually add a `SecurityPolicy` resource which will enforce the authentication. For more information about `basic auth` with `SecurityPolicy` please visit [this](https://gateway.envoyproxy.io/docs/tasks/security/basic-auth/) page.

The final file should look like this:

  ```
  apiVersion: gateway.networking.k8s.io/v1
  kind: Gateway
  metadata:
    creationTimestamp: null
    name: nginx
    namespace: default
  spec:
    gatewayClassName: tigera-gateway-class
    listeners:
    - hostname: echoserver.myingress.com
      name: echoserver-myingress-com-http
      port: 80
      protocol: HTTP
  status: {}
  ---
  apiVersion: gateway.networking.k8s.io/v1
  kind: HTTPRoute
  metadata:
    creationTimestamp: null
    name: ingress-with-auth-echoserver-myingress-com
    namespace: default
  spec:
    hostnames:
    - echoserver.myingress.com
    parentRefs:
    - name: nginx
    rules:
    - backendRefs:
      - name: app-v1
        port: 8080
        weight: 80
      - name: app-v2
        port: 8080
        weight: 20
      matches:
      - path:
          type: PathPrefix
          value: /
  status:
    parents: []
  ---
  apiVersion: gateway.envoyproxy.io/v1alpha1
  kind: SecurityPolicy
  metadata:
    name: basic-auth-example
  spec:
    targetRefs:
    - group: gateway.networking.k8s.io
      kind: HTTPRoute
      name: ingress-with-auth-echoserver-myingress-com
    basicAuth:
      users:
        name: "basic-auth"
  ```

***6.4*** - Apply the edit `gateway-resources.yaml` file:

  ```
  kubectl apply -f gateway-resources.yaml
  ```

#### 7. Wait for 30 seconds to allow services and gateway to be ready

  ```
  sleep 30
  ```

#### 8. Retrieve the external IP of the Gateway

  ```
  export HOSTNAME="echoserver.myingress.com"
  export MIGRATION_GATEWAY=$(kubectl get gateway/nginx -o jsonpath='{.status.addresses[0].value}')
  echo "HOSTNAME is: $HOSTNAME"
  echo "MIGRATION_GATEWAY is: $MIGRATION_GATEWAY"
  ```

#### 9. TEST Gateway

***9.1*** - Test the connection to the gateway without the user:

  ```
  while true; do curl -vs -H "Host: $HOSTNAME" http://$MIGRATION_GATEWAY/ | grep "<h1>"; sleep 1; done
  ```

  You should get an `401 - Unauthorized` error:

  ```
  Connected to 48.217.130.0 (48.217.130.0) port 80
  > GET / HTTP/1.1
  > Host: echoserver.myingress.com
  > User-Agent: curl/8.7.1
  > Accept: */*
  >
  * Request completely sent off
  < HTTP/1.1 401 Unauthorized
  < www-authenticate: Basic realm="http://echoserver.myingress.com/"
  < content-length: 58
  < content-type: text/plain
  < date: Tue, 19 Aug 2025 12:00:16 GMT
  <
  { [58 bytes data]
  * Connection #0 to host 48.217.130.0 left intact
  ```

***9.2*** - Test the connection with the user:

  ```
  while true; do curl -vs -H "Host: $HOSTNAME" -u foo:bar http://$MIGRATION_GATEWAY/ | grep "<h1>"; sleep 1; done
  ```

  You should get a `200 OK`:

  ```
  *   Trying 48.217.130.0:80...
  * Connected to 48.217.130.0 (48.217.130.0) port 80
  * Server auth using Basic with user 'foo'
  > GET / HTTP/1.1
  > Host: echoserver.myingress.com
  > Authorization: Basic Zm9vOmJhcg==
  > User-Agent: curl/8.7.1
  > Accept: */*
  >
  * Request completely sent off
  < HTTP/1.1 200 OK
  < server: nginx/1.29.1
  < date: Tue, 19 Aug 2025 12:01:21 GMT
  < content-type: text/html
  < content-length: 49
  < last-modified: Tue, 19 Aug 2025 10:37:13 GMT
  < etag: "68a453d9-31"
  < accept-ranges: bytes
  <
  { [49 bytes data]
  * Connection #0 to host 48.217.130.0 left intact
  <html><body><h1>App Version 1</h1></body></html>
  ```

  You should also see the canary deployment in action:

  Command:
  ```
  while true; do curl -s -H "Host: $HOSTNAME" -u foo:bar http://$MIGRATION_GATEWAY/ | grep "<h1>"; sleep 1; done
  ```

  Output:
  ```
    <html><body><h1>App Version 1</h1></body></html>
    <html><body><h1>App Version 1</h1></body></html>
    <html><body><h1>App Version 1</h1></body></html>
    <html><body><h1>App Version 1</h1></body></html>
    <html><body><h1>App Version 1</h1></body></html>
    <html><body><h1>App Version 2</h1></body></html>
    <html><body><h1>App Version 1</h1></body></html>
    <html><body><h1>App Version 1</h1></body></html>
    <html><body><h1>App Version 1</h1></body></html>
    <html><body><h1>App Version 2</h1></body></html>
  ```

### Clean-up

#### 1. Delete the app, service, HTTPRoute, Ingress, Secret, SecurityPolicy and Gateway

  ```
  kubectl delete service app-v1 app-v2
  kubectl delete deployment app-v1 app-v2
  kubectl delete secret basic-auth
  kubectl delete ingress ingress-with-auth ingress-with-auth-canary
  kubectl delete gateway nginx
  kubectl delete HTTPRoute ingress-with-auth-echoserver-myingress-com
  kubectl delete SecurityPolicy basic-auth-example
  ```

===
> **Congratulations! You have completed `Calico Ingress Gateway Workshop - Migration from NGINX Ingress with ingress2gateway`!**

---
**Credits:** Portions of this guide are based on or derived from the [Envoy Gateway documentation](https://gateway.envoyproxy.io/docs/install/migrating-to-envoy/).
