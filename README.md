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

6.  <details>
    <summary><code>JQ</code> app is installed on the bastion</summary>

        sudo apt install -y jq
    </details>

7.  <details>
    <summary><code>HEY</code> app is installed on the bastion</summary>

        sudo apt install -y hey
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