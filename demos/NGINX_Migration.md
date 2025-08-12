# Calico Ingress Gateway - Migration from NGINX Ingress 

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

Organizations migrate from the older Ingress API to the newer Gateway API because Gateway API offers more flexibility, better extensibility, and clearer separation of concerns for managing traffic routing in Kubernetes. 

In real-world scenarios, authentication mechanisms (like Basic Auth) are often enforced at the ingress layer to secure applications. To simulate this during migration, Basic Auth can be implemented in Envoy Gateway using Envoy’s `SecurityPolicy` or authentication filters, ensuring that security controls are preserved while adopting the more powerful Gateway API.

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
    <summary><code>htpasswd</code> app is installed on the bastion</summary>

        sudo apt install -y apache2-utils
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

This demo showcases how to deploy a simple echo server on Kubernetes with basic HTTP authentication configured through both an NGINX Ingress and a Gateway API setup.

Key steps demonstrated:
Deploy an echo server with a Deployment and expose it internally with a ClusterIP Service.

Create a basic-auth secret using htpasswd for user authentication.

Configure an NGINX Ingress with annotations to enable basic auth, protecting access to the echo server.

Use curl commands to show how requests are rejected without credentials and accepted with the correct username/password.

Deploy a Gateway and HTTPRoute using the Gateway API (instead of Ingress) with a corresponding security policy for basic auth.

Show how to access the echo server through the Gateway API with authentication, illustrating a migration path from Ingress to Gateway API while maintaining security.

It’s a full example of securing an application with basic authentication via two Kubernetes traffic routing methods.

#### 1. Create a `echoserver-svc` service, which is bound to an `echoserver` deployment.

  ```
  cat << EOF | kubectl apply -f -
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    name: echoserver
  spec:
    replicas: 1
    selector:
      matchLabels:
        app: echoserver
    template:
      metadata:
        labels:
          app: echoserver
      spec:
        containers:
        - name: echoserver
          image: k8s.gcr.io/echoserver:1.10
          ports:
          - containerPort: 8080
  ---
  apiVersion: v1
  kind: Service
  metadata:
    name: echoserver-svc
  spec:
    selector:
      app: echoserver
    ports:
    - port: 8080
      targetPort: 8080
    type: ClusterIP
  EOF
  ```

#### 2. Create an user/password pair which we will use for the authentication for both IngressAPI and GatewayAPI, via a kubernetes `secret`

  ```
  htpasswd -bcs auth foo bar
  cp auth .htpasswd
  kubectl create secret generic basic-auth --from-file=auth --from-file=.htpasswd
  ```

#### 3. Create an `Ingress` for the `echoserver-svc` service, which requires authentication

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
              name: echoserver-svc
              port:
                number: 8080
  EOF
  ```

#### 4. TEST Ingress

***4.1*** - Save the hostname of the Kubernetes control plane of this lab, which is tied to the NGINX Controller:

  ```
  INGRESS_URL="$(kubectl cluster-info | grep control | awk -F ":" '{print $2}' | sed "s/\/\///g")"
  ```

***4.2*** - Test the connection without the user:

  ```
  curl -kv -I -H "Host: $HOSTNAME" https://$INGRESS_URL
  ```

  You should get an `401 - Authentication Required` error:

  ```
  HTTP/2 401
  date: Tue, 12 Aug 2025 18:34:56 GMT
  content-type: text/html
  content-length: 172
  www-authenticate: Basic realm="Authentication Required - demo"
  strict-transport-security: max-age=15724800; includeSubDomains
  ```

***4.3*** - Test the connection with the user:

  ```
  curl -k -I -H "Host: $HOSTNAME" -u foo:bar https://$INGRESS_URL
  ```

  You should get a `200 OK`:

  ```
  HTTP/2 200
  date: Tue, 12 Aug 2025 18:35:49 GMT
  content-type: text/plain
  strict-transport-security: max-age=15724800; includeSubDomains
  ```

#### 5. Create a Gateway resource named `nginx-migration-gateway` using the `tigera-gateway-class` which will listen on port 80 and will replace our Ingress

  ```
  kubectl apply -f - <<EOF
  apiVersion: gateway.networking.k8s.io/v1
  kind: Gateway
  metadata:
    name: nginx-migration-gateway
  spec:
    gatewayClassName: tigera-gateway-class
    listeners:
    - name: http
      protocol: HTTP
      port: 80
  EOF
  ```

#### 6. Create an `HTTPRoute` to forward traffic from the gateway to the `echoserver-svc` service and attach a `SecurityPolicy` to it. The `SecurityPolicy` will enforce the authentication:

  ```
  cat << EOF | kubectl apply -f -
  apiVersion: gateway.networking.k8s.io/v1
  kind: HTTPRoute
  metadata:
    name: echoserver-route
  spec:
    parentRefs:
    - name: nginx-migration-gateway
    hostnames:
    - "$HOSTNAME"
    rules:
    - matches:
      - path:
          type: PathPrefix
          value: "/"
      backendRefs:
      - name: echoserver-svc
        port: 8080
  ---
  apiVersion: gateway.envoyproxy.io/v1alpha1
  kind: SecurityPolicy
  metadata:
    name: basic-auth-example
  spec:
    targetRefs:
    - group: gateway.networking.k8s.io
      kind: HTTPRoute
      name: echoserver-route
    basicAuth:
      users:
        name: "basic-auth"
  EOF
  ```

#### 7. Wait for 30 seconds to allow services and gateway to be ready
sleep 30

#### 8. Retrieve the external IP of the Gateway

  ```
  export MIGRATION_GATEWAY=$(kubectl get gateway/nginx-migration-gateway -o jsonpath='{.status.addresses[0].value}')
  ```

#### 9. TEST Gateway

***4.1*** - Test the connection to the gateway without the user:

  ```
  curl -k -I -H "Host: $HOSTNAME" "http://${MIGRATION_GATEWAY}/"
  ```

  You should get an `401 - Unauthorized` error:

  ```
  HTTP/1.1 401 Unauthorized
  www-authenticate: Basic realm="http://echoserver.myingress.com/"
  content-length: 58
  content-type: text/plain
  date: Tue, 12 Aug 2025 18:42:59 GMT
  ```

***4.2*** - Test the connection with the user:

  ```
  curl -k -I -H "Host: $HOSTNAME" -u foo:bar "http://${MIGRATION_GATEWAY}/"
  ```

  You should get a `200 OK`:

  ```
  HTTP/1.1 200 OK
  date: Tue, 12 Aug 2025 18:43:49 GMT
  content-type: text/plain
  server: echoserver
  transfer-encoding: chunked
  ```

### Clean-up

#### 1. Delete the app, service, HTTPRoute, Ingress, Secret, SecurityPolicy and Gateway

  ```
  kubectl delete service echoserver-svc
  kubectl delete deployment echoserver
  kubectl delete secret basic-auth
  kubectl delete ingress ingress-with-auth
  kubectl delete gateway nginx-migration-gateway
  kubectl delete HTTPRoute echoserver-route
  kubectl delete SecurityPolicy basic-auth-example
  ```

===
> **Congratulations! You have completed `Calico Ingress Gateway Workshop - Migration from NGINX Ingress'`!**

---
**Credits:** Portions of this guide are based on or derived from the [Envoy Gateway documentation](https://gateway.envoyproxy.io/docs/tasks/security/basic-auth/).
