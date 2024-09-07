# k8s-kubeadm
## https://dev.to/ayaan49/kubernetes-cluster-setup-using-kubeadm-on-aws-2096
## https://v1-29.docs.kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/
## quick spin up kubernetes cluster -> master node , choose ec2 instance ubuntu 24.04 and in userdata 
### userdata for all types of nodes master or worker
```
#!/bin/bash
sudo su -
ufw disable
swapoff -a; sed -i '/swap/d' /etc/fstab

cat >>/etc/sysctl.d/kubernetes.conf<<EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
sysctl --system


# Add Docker's official GPG key:
apt-get update
apt-get install -y ca-certificates curl apt-transport-https gpg
install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
chmod a+r /etc/apt/keyrings/docker.asc

# Add the repository to Apt sources:
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  tee /etc/apt/sources.list.d/docker.list > /dev/null
apt-get update
apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
systemctl restart containerd
systemctl enable containerd


curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.29/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg;
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.29/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list;
apt-get update
apt-get install -y kubelet kubeadm kubectl
apt-mark hold kubelet kubeadm kubectl
systemctl restart kubelet
systemctl enable kubelet

curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
chmod 700 get_helm.sh
./get_helm.sh

rm /etc/containerd/config.toml
systemctl restart containerd

# https://stackoverflow.com/questions/55571566/unable-to-bring-up-kubernetes-api-server
containerd config default | tee /etc/containerd/config.toml
sed -i 's/SystemdCgroup = false/SystemdCgroup = true/g' /etc/containerd/config.toml  
service containerd restart
service kubelet restart

```

### ssh to the node 
```
# installing k9s to easy control of k8s
wget https://github.com/derailed/k9s/releases/download/v0.32.0/k9s_Linux_amd64.tar.gz
tar -xzf k9s_Linux_amd64.tar.gz
chmod a+x k9s
mv k9s /bin/

# installing the cluster and replace the kube proxy with cilium's
# https://docs.cilium.io/en/stable/installation/k8s-install-kubeadm/
kubeadm init --skip-phases=addon/kube-proxy
export KUBECONFIG=/etc/kubernetes/admin.conf
kubectl taint nodes {master-node} node-role.kubernetes.io/control-plane:NoSchedule-
helm repo add cilium https://helm.cilium.io/
# ip you can get from config file
# https://docs.cilium.io/en/stable/network/kubernetes/kubeproxy-free/
helm upgrade cilium cilium/cilium --version 1.16.1 --namespace kube-system --set kubeProxyReplacement=true --set k8sServiceHost='apiserverIP' --set k8sServicePort='6443'
```
