# Calico Ingress Gateway - SSL offload / TLS Termination for TCP

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

Organizations adopt TLS termination at the gateway for TCP services (databases, custom TCP apps, message brokers) to centralize certificate handling, simplify backend configuration, and improve performance by offloading cryptographic operations. This ensures secure external access while keeping internal service communication simpler.

Calico Ingress Gateway supports TLS termination for TCP traffic through the Gateway API using `TCPRoute` combined with TLS configuration on a `Gateway`. The TLS handshake is handled at Envoy, which selects the right certificate (via SNI if needed) and then forwards decrypted TCP traffic to the backend service. This allows organizations to perform SSL offload at the edge, while still managing routing and security declaratively in Kubernetes.

---

### High Level Tasks

- Create a ServiceAccount, a Service and a Deployment for the `backend-ssl-offload` application
- Create a Gateway resource
- Create an HTTPRoute `backend-ssl-offload` which routes traffic to the `backend-ssl-offload` service
- Retrieve the external IP of the Envoy Gateway and **test** with no certificate
- Create a root certificate and private key to sign certificates
- Create a certificate and a private key for `www.example.com`
- Store the cert/key in a Secret
- Update the Gateway to include an HTTPS listener that terminates TLS using the `example-cert` secret
- Create a `TCPRoute` for SSL/TLS offload because the Gateway listener terminates TLS at the edge, and the `TCPRoute` tells the gateway where to forward the decrypted TCP traffic (the plain HTTP inside TLS) to your backend service, effectively connecting the listener to the service handling the actual application traffic
- Query the example app through the Gateway on port 443 and validate the server's certificate using `example.com.crt`

### Diagram

Coming Soon in v2

### Demo

This demo will walk through the steps required to configure TLS Terminate mode for TCP traffic via Calico Ingress Gateway. This task uses a self-signed CA, so it should be used for testing and demonstration purposes only.

#### 1. Create a `ServiceAccount`, a `Service` and a `Deployment` for the `backend-ssl-offload` application

  ```
  cat <<EOF | kubectl apply -f -
  apiVersion: v1
  kind: ServiceAccount
  metadata:
    name: backend-ssl-offload
  ---
  apiVersion: v1
  kind: Service
  metadata:
    name: backend-ssl-offload
    labels:
      app: backend-ssl-offload
      service: backend-ssl-offload
  spec:
    ports:
      - name: http
        port: 3000
        targetPort: 3000
    selector:
      app: backend-ssl-offload
  ---
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    name: backend-ssl-offload
  spec:
    replicas: 1
    selector:
      matchLabels:
        app: backend-ssl-offload
        version: v1
    template:
      metadata:
        labels:
          app: backend-ssl-offload
          version: v1
      spec:
        serviceAccountName: backend-ssl-offload
        containers:
          - image: gcr.io/k8s-staging-gateway-api/echo-basic:v20231214-v1.0.0-140-gf544a46e
            imagePullPolicy: IfNotPresent
            name: backend-ssl-offload
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

#### 2. Create a Gateway resource named "ssl-offload-gateway" using the "tigera-gateway-class".

  ```
  kubectl apply -f - <<EOF
  apiVersion: gateway.networking.k8s.io/v1
  kind: Gateway
  metadata:
    name: ssl-offload-gateway
  spec:
    gatewayClassName: tigera-gateway-class
    listeners:
      - name: http
        protocol: HTTP
        port: 80
  EOF
  ```

#### 3. Create an HTTPRoute `backend-ssl-offload` which routes traffic to the `backend-ssl-offload` service:
  ```
  kubectl apply -f - <<EOF
  apiVersion: gateway.networking.k8s.io/v1
  kind: HTTPRoute
  metadata:
    name: backend-ssl-offload
  spec:
    parentRefs:
      - name: ssl-offload-gateway
    hostnames:
      - "www.example.com"
    rules:
      - backendRefs:
          - group: ""
            kind: Service
            name: backend-ssl-offload
            port: 3000
            weight: 1
        matches:
          - path:
              type: PathPrefix
              value: /
  EOF
  ```

#### 4. Wait for 30 seconds to allow services and gateway to be ready

  ```
  sleep 30
  ```

#### 5. Retrieve the external IP of the Envoy Gateway

  ```
  export GATEWAY_SSL_OFFLOAD_DEMO=$(kubectl get gateway/ssl-offload-gateway -o jsonpath='{.status.addresses[0].value}')
  echo "GATEWAY_SSL_OFFLOAD_DEMO is: $GATEWAY_SSL_OFFLOAD_DEMO"
  ```

#### 6. Test with no certificate

From the bastion, send a request to the gateway and get a response as shown below:

  ```
  curl --verbose --header "Host: www.example.com" http://$GATEWAY_SSL_OFFLOAD_DEMO/get
  ```

   A `200` status code should be returned and the body should include `"pod": "backend-ssl-offload-*"` indicating the traffic was routed to the example `backend-ssl-offload` service. Sample of output:

  ```
  *   Trying 10.10.10.22:80...
  * TCP_NODELAY set
  * Connected to 10.10.10.22 (10.10.10.22) port 80 (#0)
  > GET /get HTTP/1.1
  > Host: www.example.com
  > User-Agent: curl/7.68.0
  > Accept: */*
  >
  * Mark bundle as not supporting multiuse
  < HTTP/1.1 200 OK
  < content-type: application/json
  < x-content-type-options: nosniff
  < date: Mon, 04 Aug 2025 14:23:59 GMT
  < content-length: 471
  <
  {
  "path": "/get",
  "host": "www.example.com",
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
    "a0335487-7ea9-48dd-9828-59658607064b"
    ]
  },
  "namespace": "default",
  "ingress": "",
  "service": "",
  "pod": "backend-ssl-offload-74bdbf5d5f-qh227"
  * Connection #0 to host 10.10.10.22 left intact
  ```

#### 7. Create a root certificate and private key to sign certificates

  ```
  openssl req -x509 -sha256 -nodes -days 365 -newkey rsa:2048 -subj '/O=example Inc./CN=example.com' -keyout example.com.key -out example.com.crt
  ```

#### 8. Create a certificate and a private key for `www.example.com`

  ```
  openssl req -out www.example.com.csr -newkey rsa:2048 -nodes -keyout www.example.com.key -subj "/CN=www.example.com/O=example organization"
  openssl x509 -req -days 365 -CA example.com.crt -CAkey example.com.key -set_serial 0 -in www.example.com.csr -out www.example.com.crt

  ```

#### 9. Store the cert/key in a Secret

  ```
  kubectl create secret tls example-cert --key=www.example.com.key --cert=www.example.com.crt
  ```

#### 10. Update the Gateway to include an HTTPS listener that terminates TLS using the `example-cert` secret

  ```
  kubectl patch gateway ssl-offload-gateway --type=json --patch '
    - op: add
      path: /spec/listeners/-
      value:
        name: https
        protocol: HTTPS
        port: 443
        tls:
          mode: Terminate
          certificateRefs:
          - kind: Secret
            group: ""
            name: example-cert
    '
  ```

#### 11. Create a `TCPRoute` for SSL/TLS offload because the Gateway listener terminates TLS at the edge, and the `TCPRoute` tells the gateway where to forward the decrypted TCP traffic (the plain HTTP inside TLS) to your backend service, effectively connecting the listener to the service handling the actual application traffic.

  ```
  cat << EOF | kubectl apply -f -
  apiVersion: gateway.networking.k8s.io/v1alpha2
  kind: TCPRoute
  metadata:
    name: backend-ssl-offload
  spec:
    parentRefs:
    - group: gateway.networking.k8s.io
      kind: Gateway
      name: ssl-offload-gateway
    rules:
    - backendRefs:
      - group: ""
        kind: Service
        name: backend-ssl-offload
        port: 3000
        weight: 1
  EOF
  ```

#### 12. Query the example app through the Gateway on port 443 and validate the server's certificate using `example.com.crt`

  ```
  export GATEWAY_SSL_OFFLOAD_DEMO=$(kubectl get gateway/ssl-offload-gateway -o jsonpath='{.status.addresses[0].value}')
  echo "GATEWAY_SSL_OFFLOAD_DEMO is: $GATEWAY_SSL_OFFLOAD_DEMO"
  curl -v -HHost:www.example.com --resolve "www.example.com:443:${GATEWAY_SSL_OFFLOAD_DEMO}" \
--cacert example.com.crt https://www.example.com/get

  ```

  The output should show `200` response and `SSL certificate verify ok`. Sample of output:

  ```
  * Added www.example.com:443:10.10.10.22 to DNS cache
  * Hostname www.example.com was found in DNS cache
  *   Trying 10.10.10.22:443...
  * TCP_NODELAY set
  * Connected to www.example.com (10.10.10.22) port 443 (#0)
  * ALPN, offering h2
  * ALPN, offering http/1.1
  * successfully set certificate verify locations:
  *   CAfile: example.com.crt
    CApath: /etc/ssl/certs
  * TLSv1.3 (OUT), TLS handshake, Client hello (1):
  * TLSv1.3 (IN), TLS handshake, Server hello (2):
  * TLSv1.3 (IN), TLS handshake, Encrypted Extensions (8):
  * TLSv1.3 (IN), TLS handshake, Certificate (11):
  * TLSv1.3 (IN), TLS handshake, CERT verify (15):
  * TLSv1.3 (IN), TLS handshake, Finished (20):
  * TLSv1.3 (OUT), TLS change cipher, Change cipher spec (1):
  * TLSv1.3 (OUT), TLS handshake, Finished (20):
  * SSL connection using TLSv1.3 / TLS_AES_256_GCM_SHA384
  * ALPN, server accepted to use h2
  * Server certificate:
  *  subject: CN=www.example.com; O=example organization
  *  start date: Aug  4 14:31:48 2025 GMT
  *  expire date: Aug  4 14:31:48 2026 GMT
  *  common name: www.example.com (matched)
  *  issuer: O=example Inc.; CN=example.com
  *  SSL certificate verify ok.
  * Using HTTP2, server supports multi-use
  * Connection state changed (HTTP/2 confirmed)
  * Copying HTTP/2 data in stream buffer to connection buffer after upgrade: len=0
  * Using Stream ID: 1 (easy handle 0x5594f75e7b60)
  > GET /get HTTP/2
  > Host:www.example.com
  > user-agent: curl/7.68.0
  > accept: */*
  >
  * Connection state changed (MAX_CONCURRENT_STREAMS == 100)!
  < HTTP/2 200
  < content-type: application/json
  < x-content-type-options: nosniff
  < date: Mon, 04 Aug 2025 14:40:35 GMT
  < content-length: 472
  <
  {
  "path": "/get",
  "host": "www.example.com",
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
    "https"
    ],
    "X-Request-Id": [
    "da47c96d-4fa4-4591-aeff-0750125687a6"
    ]
  },
  "namespace": "default",
  "ingress": "",
  "service": "",
  "pod": "backend-ssl-offload-74bdbf5d5f-qh227"
  * Connection #0 to host www.example.com left intact
  ```

### Clean-up

#### 1. Delete app, HTTPRoute, TCPRoute, service, Gateway and secrets

  ```
  kubectl delete service backend-ssl-offload
  kubectl delete deployment backend-ssl-offload
  kubectl delete gateway ssl-offload-gateway
  kubectl delete HTTPRoute backend-ssl-offload
  kubectl delete TCPRoute backend-ssl-offload
  kubectl delete secret example-cert
  ```

===
> **Congratulations! You have completed `Calico Ingress Gateway Workshop - SSL offload / TLS Termination for TCP`!**

---
**Credits:** Portions of this guide are based on or derived from the [Envoy Gateway documentation](https://gateway.envoyproxy.io/docs/tasks/security/tls-termination/).