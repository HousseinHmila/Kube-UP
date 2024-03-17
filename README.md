# Project Overview

This project contains two principal branches:

### app-code

The `app-code` branch contains a dockerized sample web application built using Python Flask. It also includes a GitHub workflow to build and deploy the Docker image. After deployment, it triggers ArgoCD to synchronize the application deployed on the Kubernetes cluster.

### deploy-code

The `deploy-code` branch is designed to present all the Kubernetes resource definitions using Helm. It is also used by ArgoCD to listen for changes.


# Cluster Setup

To set up the cluster, I've utilized Kubeadm. Below are the steps I followed:

1. **Creating Virtual Machines (VMs):**
   I initially created two VMs, one designated as the master and the other as the worker. It's crucial to ensure private connectivity between these VMs.

2. **Disabling Swap:**
   On each VM, disable swap by executing the following commands:
   ```bash
   swapoff -a
   sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab

3. **Container Runtime:**

For the container runtime, I opted for containerd. You can download the suitable version based on your distribution from this link: `https://github.com/containerd/containerd/blob/main/docs/getting-started.md`

After installing containerd, it's crucial to adjust the configuration to prevent potential issues, especially with launching kube-system pods, particularly on Debian's kernel. To do so, modify the /etc/containerd/config.toml file as follows:
```yaml
version = 2
[plugins]
  [plugins."io.containerd.grpc.v1.cri"]
   [plugins."io.containerd.grpc.v1.cri".containerd]
      [plugins."io.containerd.grpc.v1.cri".containerd.runtimes]
        [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc]
          runtime_type = "io.containerd.runc.v2"
          [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
            SystemdCgroup = true
```
then restart the containerd `sudo systemctl restart containerd`

4. **Forwarding IPv4 and letting iptables see bridged traffic**

Enable forwarding of IPv4 and ensure iptables can see bridged traffic by running the following commands:
```yaml
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

# sysctl params required by setup, params persist across reboots
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

# Apply sysctl params without reboot
sudo sysctl --system
```
5. **Install kubelet, kubeadm and kubectl**
```yaml
sudo apt-get update
# apt-transport-https may be a dummy package; if so, you can skip that package
sudo apt-get install -y apt-transport-https ca-certificates curl gpg
# If the directory `/etc/apt/keyrings` does not exist, it should be created before the curl command, read the note below.
# sudo mkdir -p -m 755 /etc/apt/keyrings
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.29/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
# This overwrites any existing configuration in /etc/apt/sources.list.d/kubernetes.list
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.29/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```
6. **Initialize our cluster:**

we execute this command on the vm designed to be the master node:
``` 
sudo kubeadm init --apiserver-advertise-address=192.168.56.11 --pod-network-cidr=10.244.0.0/16
```
you should see something like this:
```yaml
Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

You should now deploy a Pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  /docs/concepts/cluster-administration/addons/

You can now join any number of machines by running the following on each node
as root:

  kubeadm join <control-plane-host>:<control-plane-port> --token <token> --discovery-token-ca-cert-hash sha256:<hash>
  ```
you can choose the CNI you want for me I work with Weave-net, and 
we finish by executing the join command in the other VMs to join them into the cluster.

7. **Deploy metrics server:**
```yaml
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

8. **Deploy ArgoCD:**
```yaml
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

#### Now that we have prepared all the requirements, let's start deploying and testing the application. To continue with the guide, please navigate to the `deploy-code` repository where you will find the next set of instructions.
