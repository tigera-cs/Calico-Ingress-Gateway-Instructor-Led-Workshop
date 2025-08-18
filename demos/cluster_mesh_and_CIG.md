# Calico Ingress Gateway - Canary Deployment with Traffic Splitting

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

Calico Cluster-Mesh is a multi-cluster Kubernetes deployment pattern that federates clusters to enable seamless service discovery, routing, security, and observability across any environment. It leverages the Calico CNI without requiring a separate control plane, simplifying operations. The dataplane operates at the kernel level, ensuring zero overhead for inter-cluster traffic. Additionally, Cluster-Mesh integrates with native Kubernetes constructs, eliminating the need for platform teams to introduce extra components that could increase deployment complexity. 

Kubernetes Ingress has been the standard for managing external access to services within a cluster, but it has several shortcomings compared to the newer Gateway API. Calico Ingress Gateway supports the Gateway API specification and addresses the limitations of ingress by offering a more extensive API. 

Calico Ingress Gateway can be integrated with Calico Cluster-Mesh to enable high-availability (HA) and fault-tolerant traffic forwarding across multiple Kubernetes clusters. By leveraging Calico’s inter-cluster networking capabilities, traffic can seamlessly route between clusters, ensuring resilience in the event of node or cluster failures. This combination enhances scalability, improves load distribution, and provides a robust failover mechanism, making it ideal for multi-cluster and hybrid cloud deployments.

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
    <summary><code>JQ</code> app is installed on the bastion</summary>

        sudo apt install -y jq
    </details>

6.  <details>
    <summary><code>k3s</code> is deployed on <code>nonk8s1</code> VM</summary>

      A. From the bastion, ssh into the VM:

        ssh nonk8s1

      B. Disable NetworkManager’s cloud setup services, force all kernels to use the legacy cgroup v1 system, and then reboot the system to apply the changes:

        sudo systemctl disable --now nm-cloud-setup.service nm-cloud-setup.timer
        sudo grubby --update-kernel=ALL --args="systemd.unified_cgroup_hierarchy=0"
        sudo reboot

      C. Wait 2 minutes for the VM to be up and running, then ssh into it again:

        sleep 120
        ssh nonk8s1

      D. Load kernel modules needed for Kubernetes networking, set sysctl parameters to enable packet forwarding and bridge traffic filtering, and then apply these settings system-wide:

        sudo modprobe overlay
        sudo modprobe br_netfilter
        cat <<EOF | sudo tee /etc/sysctl.d/90-kubelet.conf
        net.bridge.bridge-nf-call-iptables=1
        net.ipv4.ip_forward=1
        net.bridge.bridge-nf-call-ip6tables=1
        EOF
        sudo sysctl --system

      E. Install iptables utilities:

        sudo yum install -y iptables iptables-services

      F. Download and run the K3s installer to set up a Kubernetes cluster without Flannel or Traefik, using the specified cluster network range and configuration:

        curl -sfL https://get.k3s.io | \
          K3S_KUBECONFIG_MODE="644" \
          INSTALL_K3S_EXEC="--flannel-backend=none --cluster-cidr=192.168.0.0/16 --disable-network-policy --disable=traefik" \
          sh -

      G. Edit k3s config file and copy it in `.kube/config`:

        sudo sed -i 's|server: https://127\.0\.0\.1:6443|server: https://10.0.1.32:6443|' /etc/rancher/k3s/k3s.yaml
        cp /etc/rancher/k3s/k3s.yaml .kube/config
      
      H. Confirm installation was successful:

        sudo systemctl status k3s
        kubectl get nodes -o wide

    </details>

7.  <details>
    <summary><code>Calico Enterprise</code> is installed on the k3s cluster on <code>nonk8s1</code> VM</summary>

      A. Copy repository key and license into the VM:

        scp config.json nonk8s1:config.json
        scp license.yaml nonk8s1:license.yaml

      B. SSH into the VM and install `Helm`:
        ssh nonk8s1
        sudo curl -L https://mirror.openshift.com/pub/openshift-v4/clients/helm/latest/helm-linux-amd64 -o /usr/local/bin/helm
        sudo chmod +x /usr/local/bin/helm

      C. Install Calico Enterprise v3.21 Minimal Install using Helm:

        cat > values.yaml <<EOF
        installation:
          cni:
            type: Calico
          calicoNetwork:
            bgp: Enabled
            ipPools:
            - cidr: 192.168.0.0/16
              encapsulation: VXLAN
        logCollector:
        enabled: false
        logStorage:
        enabled: false
        manager:
        enabled: false
        EOF

        curl -O -L https://downloads.tigera.io/ee/charts/tigera-operator-v3.21.2-0.tgz

        helm install calico-enterprise tigera-operator-v3.21.2-0.tgz -f values.yaml \
        --set-file imagePullSecrets.tigera-pull-secret=config.json,tigera-prometheus-operator.imagePullSecrets.tigera-pull-secret=config.json \
        --namespace tigera-operator  \
        --set-file licenseKeyContent=license.yaml --create-namespace

    </details>

8.  <details>
    <summary><code>Cluster Mesh</code> is deployed between the k8s cluster and the k3s cluster</summary>

      A. Copy the k3s config file to the bastion and merge it with the existing config file:

        ssh nonk8s1 "sudo cat /etc/rancher/k3s/k3s.yaml" > k3s.yaml
        KUBECONFIG=~/.kube/config:k3s.yaml kubectl config view --merge --flatten > /tmp/config
        mv /tmp/config ~/.kube/config

      B. Rename the context of the k8s cluster and copy the following script in the `cluster-mesh.sh` script:

      ADD NOTES ABOUT THIS SCRIPT


        kubectl config rename-context kubernetes-admin@kubernetes kubernetes-admin

      ---

        #!/bin/bash

        CLUSTER1_NAME="kubernetes-admin"
        CLUSTER2_NAME="default"

        #$CLUSTER1_NAME Cluster mesh setup

        kubectl config use-context $CLUSTER1_NAME

        kubectl apply -f https://downloads.tigera.io/ee/v3.20.1/manifests/federation-remote-sa.yaml
        kubectl apply -f https://downloads.tigera.io/ee/v3.20.1/manifests/federation-rem-rbac-kdd.yaml

        kubectl apply -f - <<EOF
        apiVersion: v1
        kind: Secret
        type: kubernetes.io/service-account-token
        metadata:
          name: tigera-federation-remote-cluster
          namespace: kube-system
          annotations:
            kubernetes.io/service-account.name: "tigera-federation-remote-cluster"
        EOF

        USER_TOKEN=$(kubectl get secret -n kube-system tigera-federation-remote-cluster -o jsonpath='{.data.token}' | base64 --decode)
        CA=$(kubectl config view --flatten --minify -o jsonpath='{.clusters[0].cluster.certificate-authority-data}')
        SERVER=$(kubectl config view --flatten --minify -o jsonpath='{.clusters[0].cluster.server}')

        cat << EOF  > $CLUSTER1_NAME-kubeconfig.yaml
        apiVersion: v1
        kind: Config
        current-context: tigera-federation-remote-cluster-ctx
        preferences: {}
        clusters:
        - cluster:
            certificate-authority-data: $CA
            server: $SERVER
          name: tigera-federation-remote-cluster
        contexts:
        - context:
            cluster: tigera-federation-remote-cluster
            user: tigera-federation-remote-cluster
          name: tigera-federation-remote-cluster-ctx
        users:
        - name: tigera-federation-remote-cluster
          user:
            token: $USER_TOKEN
        EOF

        ##$CLUSTER2_NAME Cluster mesh setup

        kubectl config use-context $CLUSTER2_NAME

        kubectl apply -f https://downloads.tigera.io/ee/v3.20.1/manifests/federation-remote-sa.yaml
        kubectl apply -f https://downloads.tigera.io/ee/v3.20.1/manifests/federation-rem-rbac-kdd.yaml

        kubectl apply -f - <<EOF
        apiVersion: v1
        kind: Secret
        type: kubernetes.io/service-account-token
        metadata:
          name: tigera-federation-remote-cluster
          namespace: kube-system
          annotations:
            kubernetes.io/service-account.name: "tigera-federation-remote-cluster"
        EOF

        USER_TOKEN=$(kubectl get secret -n kube-system tigera-federation-remote-cluster -o jsonpath='{.data.token}' | base64 --decode)
        CA=$(kubectl config view --flatten --minify -o jsonpath='{.clusters[0].cluster.certificate-authority-data}')
        SERVER=$(kubectl config view --flatten --minify -o jsonpath='{.clusters[0].cluster.server}')

        cat << EOF  > $CLUSTER2_NAME-kubeconfig.yaml
        apiVersion: v1
        kind: Config
        current-context: tigera-federation-remote-cluster-ctx
        preferences: {}
        clusters:
        - cluster:
            certificate-authority-data: $CA
            server: $SERVER
          name: tigera-federation-remote-cluster
        contexts:
        - context:
            cluster: tigera-federation-remote-cluster
            user: tigera-federation-remote-cluster
          name: tigera-federation-remote-cluster-ctx
        users:
        - name: tigera-federation-remote-cluster
          user:
            token: $USER_TOKEN
        EOF

        ##Setup Cluster-Mesh Remote connections

        ##$CLUSTER1_NAME to $CLUSTER2_NAME

        kubectl config use-context $CLUSTER1_NAME

        kubectl create ns cluster-mesh-$CLUSTER2_NAME

        kubectl create secret generic remote-cluster-secret-name -n cluster-mesh-$CLUSTER2_NAME \
          --from-literal=datastoreType=kubernetes \
          --from-file=kubeconfig=$CLUSTER2_NAME-kubeconfig.yaml

        kubectl create -f - <<EOF
        apiVersion: rbac.authorization.k8s.io/v1
        kind: Role
        metadata:
          name: remote-cluster-secret-access
          namespace: cluster-mesh-$CLUSTER2_NAME
        rules:
        - apiGroups: [""]
          resources: ["secrets"]
          verbs: ["watch", "list", "get"]
        ---
        apiVersion: rbac.authorization.k8s.io/v1
        kind: RoleBinding
        metadata:
          name: remote-cluster-secret-access
          namespace: cluster-mesh-$CLUSTER2_NAME
        roleRef:
          apiGroup: rbac.authorization.k8s.io
          kind: Role
          name: remote-cluster-secret-access
        subjects:
        - kind: ServiceAccount
          name: calico-typha
          namespace: calico-system
        EOF

        kubectl create -f - <<EOF
        apiVersion: projectcalico.org/v3
        kind: RemoteClusterConfiguration
        metadata:
          name: $CLUSTER2_NAME
        spec:
          clusterAccessSecret:
            name: remote-cluster-secret-name
            namespace: cluster-mesh-$CLUSTER2_NAME
            kind: Secret
          syncOptions:
            overlayRoutingMode: Enabled
        EOF

        ##$CLUSTER2_NAME to $CLUSTER1_NAME

        kubectl config use-context $CLUSTER2_NAME

        kubectl create ns cluster-mesh-$CLUSTER1_NAME

        kubectl create secret generic remote-cluster-secret-name -n cluster-mesh-$CLUSTER1_NAME \
          --from-literal=datastoreType=kubernetes \
          --from-file=kubeconfig=$CLUSTER1_NAME-kubeconfig.yaml

        kubectl create -f - <<EOF
        apiVersion: rbac.authorization.k8s.io/v1
        kind: Role
        metadata:
          name: remote-cluster-secret-access
          namespace: cluster-mesh-$CLUSTER1_NAME
        rules:
        - apiGroups: [""]
          resources: ["secrets"]
          verbs: ["watch", "list", "get"]
        ---
        apiVersion: rbac.authorization.k8s.io/v1
        kind: RoleBinding
        metadata:
          name: remote-cluster-secret-access
          namespace: cluster-mesh-$CLUSTER1_NAME
        roleRef:
          apiGroup: rbac.authorization.k8s.io
          kind: Role
          name: remote-cluster-secret-access
        subjects:
        - kind: ServiceAccount
          name: calico-typha
          namespace: calico-system
        EOF

        kubectl create -f - <<EOF
        apiVersion: projectcalico.org/v3
        kind: RemoteClusterConfiguration
        metadata:
          name: $CLUSTER1_NAME
        spec:
          clusterAccessSecret:
            name: remote-cluster-secret-name
            namespace: cluster-mesh-$CLUSTER1_NAME
            kind: Secret
          syncOptions:
            overlayRoutingMode: Enabled
        EOF

        ##Demo federated app on $CLUSTER1_NAME

        kubectl config use-context $CLUSTER1_NAME

        if [ ! -d ~/Calico-Ingress-Gateway-Instructor-Led-Workshop ]; then
          git clone https://github.com/tigera-cs/Calico-Ingress-Gateway-Instructor-Led-Workshop.git ~/Calico-Ingress-Gateway-Instructor-Led-Workshop
        fi

        if [ -d ~/backend-app ]; then
          mv ~/backend-app ~/backend-app.bak.$(date +%Y%m%d%H%M%S)
        fi

        cp -r ~/Calico-Ingress-Gateway-Instructor-Led-Workshop/etc/backend-app ~/backend-app

        kubectl apply -f ~/backend-app/east

        ##Demo federated app on  on $CLUSTER2_NAME

        kubectl config use-context $CLUSTER2_NAME

        kubectl apply -f ~/backend-app/west

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

#### 1. Create a Gateway resource named "canary-deployment-gateway" using the "tigera-gateway-class"

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

#### 2. Create a new namespace called "my-app" to isolate resources
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

#### 4. Define an HTTPRoute to split traffic between app-v1 and app-v2

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

#### 5. Create a ReferenceGrant to allow HTTPRoute in the `default` namespace to reference services in `my-app` namespace

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

#### 6. Wait for 30 seconds to allow services and gateway to be ready
sleep 30

#### 7. Retrieve the external IP of the Envoy Gateway

  ```
  EXTERNAL_IP=$(kubectl get service -n tigera-gateway -l gateway.envoyproxy.io/owning-gateway-name=canary-deployment-gateway \
    -o jsonpath='{.items[0].status.loadBalancer.ingress[0].ip}')
  echo "Envoy Gateway External IP: $EXTERNAL_IP"
  ```

#### 8. Test

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