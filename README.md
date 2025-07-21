# Kubeadm Installation Guide

This guide outlines the steps needed to set up a Kubernetes cluster using `kubeadm`.

## Prerequisites

- Ubuntu OS (Xenial or later)
- `sudo` privileges
- Internet access
- t2.medium instance type or higher

---

## AWS Setup

1. Ensure that all instances are in the same **Security Group**.
2. Expose port **6443** in the **Security Group** to allow worker nodes to join the cluster.
3. Expose port **22** in the **Security Group** to allows SSH access to manage the instance..


## To do above setup, follow below provided steps

### Step 1: Identify or Create a Security Group

1. **Log in to the AWS Management Console**:
    - Go to the **EC2 Dashboard**.

2. **Locate Security Groups**:
    - In the left menu under **Network & Security**, click on **Security Groups**.

3. **Create a New Security Group**:
    - Click on **Create Security Group**.
    - Provide the following details:
      - **Name**: (e.g., `Kubernetes-Cluster-SG`)
      - **Description**: A brief description for the security group (mandatory)
      - **VPC**: Select the appropriate VPC for your instances (default is acceptable)

4. **Add Rules to the Security Group**:
    - **Allow SSH Traffic (Port 22)**:
      - **Type**: SSH
      - **Port Range**: `22`
      - **Source**: `0.0.0.0/0` (Anywhere) or your specific IP
    
    - **Allow Kubernetes API Traffic (Port 6443)**:
      - **Type**: Custom TCP
      - **Port Range**: `6443`
      - **Source**: `0.0.0.0/0` (Anywhere) or specific IP ranges

5. **Save the Rules**:
    - Click on **Create Security Group** to save the settings.

### Step 2: Select the Security Group While Creating Instances

- When launching EC2 instances:
  - Under **Configure Security Group**, select the existing security group (`Kubernetes-Cluster-SG`)

> Note: Security group settings can be updated later as needed.

---


## Execute on Both "Master" & "Worker" Nodes

1. **Disable Swap**: Required for Kubernetes to function correctly.
    ```bash
    sudo swapoff -a
    ```
    For permenant Disable the swap
   ```bash
    sudo sed -i '/ swap / s/^/#/' /etc/fstab
    ```

## 3. Load Necessary Kernel Modules

To support container runtimes and Kubernetes networking, you must load some essential kernel modules.

### overlay

**What it is:**  
The `overlay` module enables the OverlayFS filesystem. It allows multiple filesystem layers to be overlaid into one.

**Why it's needed in Kubernetes:**  
Container runtimes such as Docker and containerd rely on OverlayFS to efficiently manage image layers (e.g., base image + app code).  
Without this module, containers may fail to start because the layered filesystem won't function correctly.

---

### br_netfilter

**What it is:**  
The `br_netfilter` module allows iptables to inspect traffic flowing across Linux bridge interfaces.

**Why it's needed in Kubernetes:**  
Most CNI networking plugins (e.g., Calico, Flannel) rely on Linux bridges to connect pods.  
This module ensures that iptables can apply firewall/NAT rules to pod-to-pod traffic over those bridges.  
Without it, cluster networking, DNS resolution, and service discovery may break.

---

### Load and Persist Modules

Run the following commands to load the required modules and ensure they're loaded on every reboot:

```bash
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

# Load modules immediately
sudo modprobe overlay
sudo modprobe br_netfilter
---

## 4. Set Sysctl Parameters: Helps with networking.
    **1. net.bridge.bridge-nf-call-iptables = 1
    What it does:**
    Allows bridged IPv4 traffic to be passed through iptables firewall chains (e.g., FORWARD, NAT).

    **Why needed:**
    Ensures Kubernetes can apply firewall/NAT rules to pod traffic (which uses Linux bridges).
    Without this, pod-to-pod or pod-to-service communication may not work.

    **2. net.bridge.bridge-nf-call-ip6tables = 1**
    **What it does:**
    Same as above, but for IPv6 traffic through ip6tables.

    **Why needed:**
    If your cluster or CNI uses IPv6, this allows proper firewalling/NAT for IPv6 pod traffic.

    **3. net.ipv4.ip_forward = 1**
    **What it does:**
    Enables Linux to forward IPv4 packets between interfaces.

    **Why needed:**
    Required for routing traffic between pods, nodes, and services.
    Without this, Kubernetes cannot route packets across nodes (multi-node networking breaks).
   ```bash
    cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
    net.bridge.bridge-nf-call-iptables  = 1
    net.bridge.bridge-nf-call-ip6tables = 1
    net.ipv4.ip_forward                 = 1
    EOF

    sudo sysctl --system
    lsmod | grep br_netfilter
    lsmod | grep overlay
    ```

## 5. Install Containerd:
    ```bash
    sudo apt-get update
    sudo apt-get install -y ca-certificates curl
    sudo install -m 0755 -d /etc/apt/keyrings
    sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
    sudo chmod a+r /etc/apt/keyrings/docker.asc

    echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo \"$VERSION_CODENAME\") stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

    sudo apt-get update
    sudo apt-get install -y containerd.io

    containerd config default | sed -e 's/SystemdCgroup = false/SystemdCgroup = true/' -e 's/sandbox_image = "registry.k8s.io\/pause:3.6"/sandbox_image = "registry.k8s.io\/pause:3.9"/' | sudo tee /etc/containerd/config.toml

    sudo systemctl restart containerd
    sudo systemctl status containerd
    ```

## 6. Install Kubernetes Components:
    ```bash
    sudo apt-get update
    sudo apt-get install -y apt-transport-https ca-certificates curl gpg

    curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.29/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

    echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.29/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list

    sudo apt-get update
    sudo apt-get install -y kubelet kubeadm kubectl
    sudo apt-mark hold kubelet kubeadm kubectl
    ```

## Execute ONLY on the "Master" Node

1. **Initialize the Cluster**:
    ```bash
    sudo kubeadm init
    ```

2. **Set Up Local kubeconfig**:
    ```bash
    mkdir -p "$HOME"/.kube
    sudo cp -i /etc/kubernetes/admin.conf "$HOME"/.kube/config
    sudo chown "$(id -u)":"$(id -g)" "$HOME"/.kube/config
    ```

3. **Install a Network Plugin (Calico)**:
    ```bash
    kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.26.0/manifests/calico.yaml
    ```

4. **Generate Join Command**:
    ```bash
    kubeadm token create --print-join-command
    ```

> Copy this generated token for next command.

---

## Execute on ALL of your Worker Nodes

1. Perform pre-flight checks:
    ```bash
    sudo kubeadm reset pre-flight checks
    ```

2. Paste the join command you got from the master node and append `--v=5` at the end:
    ```bash
    sudo kubeadm join <private-ip-of-control-plane>:6443 --token <token> --discovery-token-ca-cert-hash sha256:<hash> --cri-socket 
    "unix:///run/containerd/containerd.sock" --v=5
    ```

    > **Note**: When pasting the join command from the master node:
    > 1. Add `sudo` at the beginning of the command
    > 2. Add `--v=5` at the end
    >
    > Example format:
    > ```bash
    > sudo <paste-join-command-here> --v=5
    > ```

---


# Install Helm on Ubuntu (Method 1)

This guide explains how to install **Helm** (Kubernetes package manager) on Ubuntu using the official installation script.

---

## Prerequisites

- A working Ubuntu system
- `curl` installed (you can install it using `sudo apt install curl`)

---

## Installation Steps

Run the following command in your terminal:

## Install HELM

```bash
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
chmod 700 get_helm.sh
./get_helm.sh
```

---

## 1. Install Kube Prometheus Stack To monitor K8s Cluster

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo add stable https://charts.helm.sh/stable
helm repo update
kubectl create namespace monitoring
helm install prometheus prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --set prometheus.prometheusSpec.maximumStartupDurationSeconds=60 \
  --set prometheus.service.type=NodePort \
  --set prometheus.service.nodePort=30000 \
  --set grafana.service.type=NodePort \
  --set grafana.service.nodePort=31000 \
  --set alertmanager.service.type=NodePort \
  --set alertmanager.service.nodePort=32000 \
  --set prometheus-node-exporter.service.type=NodePort \
  --set prometheus-node-exporter.service.nodePort=32001
kubectl get svc -n monitoring
kubectl get namespace
```

---

```bash
kubectl port-forward svc/kind-prometheus-kube-prome-prometheus -n monitoring 9090:9090 --address=0.0.0.0 &
kubectl port-forward svc/kind-prometheus-grafana -n monitoring 31000:80 --address=0.0.0.0 &
```


---

## 2. Prometheus Queries

```bash
sum (rate (container_cpu_usage_seconds_total{namespace="default"}[1m])) / sum (machine_cpu_cores) * 100

sum (container_memory_usage_bytes{namespace="default"}) by (pod)


sum(rate(container_network_receive_bytes_total{namespace="default"}[5m])) by (pod)
sum(rate(container_network_transmit_bytes_total{namespace="default"}[5m])) by (pod)
sum(kube_pod_container_status_ready) by (pod, namespace)

# Combine with pod phase info
+ on(pod, namespace) group_left(phase)
(
  # Get current phase (using max to deduplicate)
  max(kube_pod_status_phase) by (pod, namespace, phase)
  == 1
)
```


---


This content was originally created by Shubham Londhe , a well-known trainer in DevOps and Cloud technologies, and is part of his popular "TrainWithShubham"  educational series. 
Credit: 

📘 Content Source: TrainWithShubham  
👨‍🏫 Instructor: Shubham Londhe 
🌐 Course: Kubernetes & Docker Training  

