# Highly Available Kubernetes Cluster on GCP (kubeadm + HAProxy)
This repository contains the complete setup of a Highly Available Kubernetes cluster built using kubeadm, HAProxy, and containerd on Google Cloud Platform.

Architecture: 

2 × HAProxy Load Balancers(we only have 1)
2 × Control Plane Nodes 
2 × Worker Nodes
<img width="1096" height="400" alt="Screenshot 2026-02-04 at 5 38 10 PM" src="https://github.com/user-attachments/assets/050b8aaf-7255-49df-bc93-b0cb9e9a35ff" />

**Load Balancer Setup (HAProxy)**

sudo apt update && sudo apt upgrade -y
sudo apt install haproxy -y

**EDIT HAPROXY.CFG**
sudo vi /etc/haproxy/haproxy.cfg

frontend kubernetes-api
         bind *:6443
         mode tcp
         option tcplog
         default_backend kube-master-nodes

What this means:

frontend kubernetes-api

This names the frontend block kubernetes-api. It's just an identifier — you could call it anything, but naming it clearly helps for readability.

bind *:6443

This tells HAProxy to listen for incoming connections on port 6443 on all interfaces (* means all available IPs on the server).

Port 6443 is the standard Kubernetes API port.
This is the port kubeadm and kubectl will connect to.
default_backend kube-masters

Any connection received on this frontend will be forwarded to the kube-masters backend (defined in the next block).

💡 This block is essentially the public-facing listener for Kubernetes API requests.

backend kube-master-nodes
        mode tcp
        balance roundrobin
        option tcp-check
        default-server inter 10s downinter 5s rise 2 fall 2 slowstart 60s maxconn 250 maxqueue 256 weight 100

    
        server master1 <private-ip of master node 01>:6443 check
        server master2 <private-ip of master node 02>:6443 check
(input your masters private IP)

**What this means:**

backend kube-masters

This names the backend pool kube-masters. This matches the default_backend in the frontend section.

mode tcp

HAProxy operates in TCP mode (not HTTP). The Kubernetes API server uses raw TCP (HTTPS), so this mode is required.

balance roundrobin

Load balances incoming connections using round-robin strategy:

First request → master1
Second request → master2
Third → master3
Then it repeats...
This spreads the load evenly across your control plane nodes.

option tcp-check

Enables health checks using TCP connection attempts. If HAProxy cannot make a TCP connection to a node’s port 6443, it considers that node unhealthy and removes it from the rotation.

default-server inter 5s fall 3 rise 2

**After Configuration**

Save and Restart HAProxy:

sudo systemctl restart haproxy

Enable at boot:

sudo systemctl enable haproxy

Check if the port 6443 is open and responding for haproxy connection or not
nc -v localhost 6443

**Run on Any master node it will act as leader**
Run the below steps on the Master VM

SSH into the Master EC2 server
Do sudo -i
Disable Swap using the below commands
swapoff -a
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
Forwarding IPv4 and letting iptables see bridged traffic
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

# Verify that the br_netfilter, overlay modules are loaded by running the following commands:
lsmod | grep br_netfilter
lsmod | grep overlay

# Verify that the net.bridge.bridge-nf-call-iptables, net.bridge.bridge-nf-call-ip6tables, and net.ipv4.ip_forward system variables are set to 1 in your sysctl config by running the following command:
sysctl net.bridge.bridge-nf-call-iptables net.bridge.bridge-nf-call-ip6tables net.ipv4.ip_forward
Install container runtime
curl -LO https://github.com/containerd/containerd/releases/download/v1.7.14/containerd-1.7.14-linux-amd64.tar.gz
sudo tar Cxzvf /usr/local containerd-1.7.14-linux-amd64.tar.gz
curl -LO https://raw.githubusercontent.com/containerd/containerd/main/containerd.service
sudo mkdir -p /usr/local/lib/systemd/system/
sudo mv containerd.service /usr/local/lib/systemd/system/
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml
sudo sed -i 's/SystemdCgroup \= false/SystemdCgroup \= true/g' /etc/containerd/config.toml
sudo systemctl daemon-reload
sudo systemctl enable --now containerd

# Check that containerd service is up and running
systemctl status containerd
Install runc
curl -LO https://github.com/opencontainers/runc/releases/download/v1.1.12/runc.amd64
sudo install -m 755 runc.amd64 /usr/local/sbin/runc
Install CNI plugin
curl -LO https://github.com/containernetworking/plugins/releases/download/v1.5.0/cni-plugins-linux-amd64-v1.5.0.tgz
sudo mkdir -p /opt/cni/bin
sudo tar Cxzvf /opt/cni/bin cni-plugins-linux-amd64-v1.5.0.tgz
Install kubeadm,kublet and kubectl
# Step 1: Update system and install required packages
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl gpg

# Step 2: Add Kubernetes apt repository
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.33/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.33/deb/ /" | sudo tee /etc/apt/sources.list.d/kubernetes.list

# Step 3: Update package list
sudo apt-get update

# Step 4: Install specific Kubernetes versions (adjust version if needed)
sudo apt-get install -y kubelet=1.33.0-1.1 kubeadm=1.33.0-1.1 kubectl=1.33.0-1.1 --allow-downgrades --allow-change-held-packages

# Step 5: Hold the package versions to prevent upgrades
sudo apt-mark hold kubelet kubeadm kubectl

# Step 6: Verify installation
kubeadm version
kubelet --version
kubectl version --client
check kubelet in properly installed and enabled or not

systemctl status kubelet
Initialize kubeadm

kubeadm init \
  --control-plane-endpoint "<load-balancer-private-ip>:6443" \
  --upload-certs \
  --pod-network-cidr=192.168.0.0/16 \
  --apiserver-advertise-address=<private-ip-of-this-ec2-instance>
final command will look like the below

image

To start using your cluster, you need to run the following as a regular user:

mkdir -p $HOME/.kube sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config sudo chown $(id -u):$(id -g) $HOME/.kube/config

Alternatively, if you are the root user, you can run:

export KUBECONFIG=/etc/kubernetes/admin.conf

now you can do kubectl get nodes to see you control plane node is okay or not if you see a not ready status now, it is normal as we need to install and configure cni plugin to start intra pod communication

image

this command will generate two different type of join tokens

Join token for other master nodes to join the control plane
image

Join token for the worker node to join the worker plane
image

copy and save both the commands this will be needed in the future

now install calico (network addon ) to start pod to pod communication

kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.28.0/manifests/tigera-operator.yaml

curl https://raw.githubusercontent.com/projectcalico/calico/v3.28.0/manifests/custom-resources.yaml -O

kubectl apply -f custom-resources.yaml
will see a output like this

image

after this if you do kubectl get nodes you will see your control-plnae node is ready state

image

all done on the leader control plane node
-----------------------------------------

Run steps below on the other control plane nodes
SSH into the Master EC2 server
Do sudo -i
Disable Swap using the below commands
swapoff -a
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
Forwarding IPv4 and letting iptables see bridged traffic
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

# Verify that the br_netfilter, overlay modules are loaded by running the following commands:
lsmod | grep br_netfilter
lsmod | grep overlay

# Verify that the net.bridge.bridge-nf-call-iptables, net.bridge.bridge-nf-call-ip6tables, and net.ipv4.ip_forward system variables are set to 1 in your sysctl config by running the following command:
sysctl net.bridge.bridge-nf-call-iptables net.bridge.bridge-nf-call-ip6tables net.ipv4.ip_forward
Install container runtime
curl -LO https://github.com/containerd/containerd/releases/download/v1.7.14/containerd-1.7.14-linux-amd64.tar.gz
sudo tar Cxzvf /usr/local containerd-1.7.14-linux-amd64.tar.gz
curl -LO https://raw.githubusercontent.com/containerd/containerd/main/containerd.service
sudo mkdir -p /usr/local/lib/systemd/system/
sudo mv containerd.service /usr/local/lib/systemd/system/
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml
sudo sed -i 's/SystemdCgroup \= false/SystemdCgroup \= true/g' /etc/containerd/config.toml
sudo systemctl daemon-reload
sudo systemctl enable --now containerd

# Check that containerd service is up and running
systemctl status containerd
Install runc
curl -LO https://github.com/opencontainers/runc/releases/download/v1.1.12/runc.amd64
sudo install -m 755 runc.amd64 /usr/local/sbin/runc
Install CNI plugin
curl -LO https://github.com/containernetworking/plugins/releases/download/v1.5.0/cni-plugins-linux-amd64-v1.5.0.tgz
sudo mkdir -p /opt/cni/bin
sudo tar Cxzvf /opt/cni/bin cni-plugins-linux-amd64-v1.5.0.tgz
Install kubeadm,kublet and kubectl
# Step 1: Update system and install required packages
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl gpg

# Step 2: Add Kubernetes apt repository
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.33/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.33/deb/ /" | sudo tee /etc/apt/sources.list.d/kubernetes.list

# Step 3: Update package list
sudo apt-get update

# Step 4: Install specific Kubernetes versions (adjust version if needed)
sudo apt-get install -y kubelet=1.33.0-1.1 kubeadm=1.33.0-1.1 kubectl=1.33.0-1.1 --allow-downgrades --allow-change-held-packages

# Step 5: Hold the package versions to prevent upgrades
sudo apt-mark hold kubelet kubeadm kubectl

# Step 6: Verify installation
kubeadm version
kubelet --version
kubectl version --client
check kubelet in properly installed and enabled or not

systemctl status kubelet
now use the join command to join the control plane leader node (I told you to save it in the leader node setup phase )

image

you will see a message like this if everything goes right

This node has joined the cluster and a new control plane instance was created:

Certificate signing request was sent to apiserver and approval was received.
The Kubelet was informed of the new secure connection details.
Control plane label and taint were applied to the new node.
The Kubernetes control plane instances scaled up.
A new etcd member was added to the local/stacked etcd cluster.
To start administering your cluster from this node, you need to run the following as a regular user:

mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

Run 'kubectl get nodes' to see this node join the cluster.

<img width="528" height="134" alt="Screenshot 2026-02-04 at 5 46 14 PM" src="https://github.com/user-attachments/assets/9b774af1-e839-4dcc-9cd3-616d0b1dfeee" />


now you have two master nodes in ready states in the control plane now lets setup the worker nodes

==================================================================================================

Make the ips are assigned properly, it took me a couple of tries to get it down.


