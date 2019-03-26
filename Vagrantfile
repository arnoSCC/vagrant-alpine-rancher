# -*- mode: ruby -*-
# vi: set ft=ruby :

VM_MEMORY=8172
VM_CORES=4

Vagrant.configure("2") do |config|

  config.vm.box = "generic/alpine38"

  config.vm.provider :vmware_desktop do |v, override|
    v.vmx['memsize'] = VM_MEMORY
    v.vmx['numvcpus'] = VM_CORES
  end

  config.vm.provider :virtualbox do |v, override|
    v.memory = VM_MEMORY
    v.cpus = VM_CORES
  end

  config.vm.provision "shell", inline: <<-SHELL
XIPIO="$(ip a show dev eth0 | grep -m1 inet | awk '{print $2}' | cut -d'/' -f1).xip.io"
echo vagrant.$XIPIO > /etc/hostname
hostname -F /etc/hostname
apk add docker shadow sshpass
rc-update add docker default
service docker start
usermod -aG docker vagrant
ssh-keygen -N '' -f ~/.ssh/id_rsa 1>/dev/null 2>&1
sshpass -p vagrant ssh-copy-id -o StrictHostKeyChecking=no vagrant@vagrant.$XIPIO 1>/dev/null 2>&1
echo "Downloading rke..."
curl -sSL -o /usr/local/bin/rke https://github.com/rancher/rke/releases/download/v0.2.0/rke_linux-amd64
chmod +x /usr/local/bin/rke
mount --make-shared /
cat<<EOF>cluster.yml
nodes:
  - address: vagrant.$XIPIO
    user: vagrant
    role: [controlplane,worker,etcd]

services:
  etcd:
    snapshot: true
    creation: 6h
    retention: 24h
EOF
rke up
echo "Downloading kubectl..."
curl -sSL -o /usr/local/bin/kubectl https://storage.googleapis.com/kubernetes-release/release/v1.13.4/bin/linux/amd64/kubectl
chmod +x /usr/local/bin/kubectl
mkdir -p /home/vagrant/.kube
cp kube_config_cluster.yml /home/vagrant/.kube/config
chown -R vagrant:vagrant /home/vagrant/.kube
mkdir -p /root/.kube
cp kube_config_cluster.yml /root/.kube/config
echo "Waiting for kube-dns deployment..."
kubectl -n kube-system rollout status deploy/kube-dns
echo "Downloading helm..."
curl -sSL https://storage.googleapis.com/kubernetes-helm/helm-v2.13.1-linux-amd64.tar.gz | tar xz
mv linux-amd64/helm /usr/local/bin/
rm -rf linux-amd64
kubectl -n kube-system create serviceaccount tiller
kubectl create clusterrolebinding tiller --clusterrole=cluster-admin --serviceaccount=kube-system:tiller
helm init --service-account tiller
echo "Waiting for tiller deployment..."
kubectl -n kube-system rollout status deploy/tiller-deploy
helm repo add rancher-stable https://releases.rancher.com/server-charts/stable 1>/dev/null 2>&1
helm install stable/cert-manager --name cert-manager --namespace kube-system --version v0.5.2 1>/dev/null 2>&1
echo "Waiting for cert-manager deployment..."
kubectl -n kube-system rollout status deploy/cert-manager
kubectl create namespace cattle-system
helm install rancher-stable/rancher --name rancher --namespace cattle-system --set hostname=rancher.$XIPIO 1>/dev/null 2>&1
echo "Waiting for rancher deployment..."
kubectl -n cattle-system rollout status deploy/rancher
echo "\nRancher is available at https://rancher.$XIPIO"
  SHELL
end