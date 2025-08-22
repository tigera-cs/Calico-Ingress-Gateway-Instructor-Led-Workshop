# Calico Ingress Gateway - Cluster Mesh and multi-cluster routing

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

Calico Cluster-Mesh is a multi-cluster Kubernetes deployment pattern that federates clusters to enable seamless service discovery, routing, security, and observability across any environment. It leverages the Calico CNI without requiring a separate control plane, simplifying operations. The dataplane operates at the kernel level, ensuring zero overhead for inter-cluster traffic. Additionally, Cluster-Mesh integrates with native Kubernetes constructs, eliminating the need for platform teams to introduce extra components that could increase deployment complexity. 

Kubernetes Ingress has been the standard for managing external access to services within a cluster, but it has several shortcomings compared to the newer Gateway API. Calico Ingress Gateway supports the Gateway API specification and addresses the limitations of ingress by offering a more extensive API. 

Calico Ingress Gateway can be integrated with Calico Cluster-Mesh to enable high-availability (HA) and fault-tolerant traffic forwarding across multiple Kubernetes clusters. By leveraging Calico’s inter-cluster networking capabilities, traffic can seamlessly route between clusters, ensuring resilience in the event of node or cluster failures. This combination enhances scalability, improves load distribution, and provides a robust failover mechanism, making it ideal for multi-cluster and hybrid cloud deployments.

---

### High Level Tasks

### Diagram

Coming Soon in v2


### Demo

#### 1. Create a Gateway resource named "canary-deployment-gateway" using the "tigera-gateway-class"

  ```
  kubectl apply -f - <<EOF
  apiVersion: gateway.networking.k8s.io/v1
  kind: Gateway
  metadata:
    name: cluster-mesh-gateway
  spec:
    gatewayClassName: tigera-gateway-class
    listeners:
      - name: http
        protocol: HTTP
        port: 80
  EOF
  ```

#### 2. Define an HTTPRoute to split traffic between backend-app in us-east (local k8s cluster) and backend-app in us-west (remote k3s cluster). To "call" the remote deployment, we need to use the `federated` service `federated-backend-us-west` as `backandRef`.

The HTTPRoute below routes 99% of requests to the local cluster `us-east` and 1% to the remote cluster `us-west`.

  ```
  cat << EOF | kubectl apply -f -
  apiVersion: gateway.networking.k8s.io/v1
  kind: HTTPRoute
  metadata:
    name: cluster-mesh
  spec:
    parentRefs:
      - name: cluster-mesh-gateway
        namespace: default
    rules:
      - matches:
          - path:
              type: PathPrefix
              value: /
        backendRefs:
          - group: ""
            kind: Service
            name: backend-us-east
            port: 3000
            weight: 99
          - group: ""
            kind: Service
            name: federated-backend-us-west
            port: 3000
            weight: 1
  EOF
  ```

#### 3. Wait for 30 seconds to allow services and gateway to be ready
sleep 30

#### 4. Retrieve the external IP of the Envoy Gateway

  ```
  EXTERNAL_IP=$(kubectl get service -n tigera-gateway -l gateway.envoyproxy.io/owning-gateway-name=cluster-mesh-gateway \
    -o jsonpath='{.items[0].status.loadBalancer.ingress[0].ip}')
  echo "Envoy Gateway External IP: $EXTERNAL_IP"
  ```

#### 5. Test

From the bastion, continuously send requests to the external IP and print the response HTML header to verify traffic splitting. You should see that 99% of the time, traffic is sent to `backend-us-east-*`:

  ```
  for i in {1..500}; do
    curl -s -H “Host: www.example.com” http://$EXTERNAL_IP | jq -r .pod
  done | sort | uniq -c
  ```

  Sample of output:
  ```
  494 backend-us-east-7f76b6f796-9dclj
    6 backend-us-west-5cb5fb745d-8dqzr
  ```

### Clean-up

#### 1. Delete apps, services, HTTPRoute and Gateway

  ```
  kubectl config use-context kubernetes-admin
  kubectl delete deploy backend-us-east
  kubectl delete svc backend-us-east
  kubectl delete svc federated-backend-us-west
  kubectl delete sa backend
  kubectl delete gateway cluster-mesh-gateway
  kubectl delete httproute cluster-mesh
  kubectl config use-context default
  kubectl delete deploy backend-us-west
  kubectl delete svc backend-us-west
  kubectl delete svc federated-backend-us-east
  kubectl delete sa backend
  ```

===
> **Congratulations! You have completed `Calico Ingress Gateway Workshop - Cluster Mesh and multi-cluster routing`!**

---
**Credits:** Portions of this guide are based on or derived from the [Envoy Gateway documentation](https://gateway.envoyproxy.io/docs/tasks/traffic/http-traffic-splitting/).