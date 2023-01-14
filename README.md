# Managing Kubernetes with Rancher

## Useful commands
* list VMs with IPs: 
    ```
    virsh net-list
    virsh net-dhcp-leases default
    ```

## Install necessary tools on local machine
```
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl

curl https://baltocdn.com/helm/signing.asc | gpg --dearmor | sudo tee /usr/share/keyrings/helm.gpg > /dev/null
sudo apt-get install apt-transport-https --yes
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/helm.gpg] https://baltocdn.com/helm/stable/debian/ all main" | sudo tee /etc/apt/sources.list.d/helm-stable-debian.list
sudo apt-get update
sudo apt-get install helm
```

## Setup Rancher Server
* start rancher-server-X virtual machines
* setup rancher-server-1:
    ```
    sudo su

    mkdir -p /etc/rancher/rke2/
    echo "token: my-shared-secret" >> /etc/rancher/rke2/config.yaml

    curl -sfL https://get.rke2.io | sh -
    systemctl enable rke2-server.service
    systemctl start rke2-server.service
    ```
* setup other rancher-server nodes:
    ```
    sudo su

    mkdir -p /etc/rancher/rke2/
    echo "token: my-shared-secret" >> /etc/rancher/rke2/config.yaml
    echo "server: https://192.168.122.101:9345" >> /etc/rancher/rke2/config.yaml

    curl -sfL https://get.rke2.io | sh -
    systemctl enable rke2-server.service
    systemctl start rke2-server.service
    ```

    For more details see docs: https://ranchermanager.docs.rancher.com/v2.6/how-to-guides/new-user-guides/kubernetes-cluster-setup/rke2-for-rancher

### Copy kubeconfig to localhost
```
sudo mkdir .kube
sudo virt-copy-out -d rancher-server-1 /etc/rancher/rke2/rke2.yaml .
sudo mv rke2.yaml .kube/config
```
Config from the VM contains incorrect host (localhost), so it has to be changed appriopriately.

### Verify connection to Rancher Server from localhost
```
# with kubeconfig specified
kubectl --kubeconfig rke2.yaml get pods -A

# if config was saved to default .kube/config location
kubectl get pods -A
```

## Install Rancher
```
# add Rancher helm repo
helm repo add rancher-latest https://releases.rancher.com/server-charts/latest

# create default namespace
kubectl create namespace cattle-system

# install cert-manager
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.7.1/cert-manager.crds.yaml

helm repo add jetstack https://charts.jetstack.io
helm repo update
helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --version v1.7.1

kubectl get pods --namespace cert-manager

# install Rancher
helm install rancher rancher-latest/rancher \
  --namespace cattle-system \
  --set hostname=192.168.122.101.sslip.io \
  --set bootstrapPassword=admin
```
https://ranchermanager.docs.rancher.com/pages-for-subheaders/install-upgrade-on-a-kubernetes-cluster

```
helm install rancher rancher-latest/rancher \
  --namespace cattle-system \
  --set hostname=192.168.122.101.sslip.io \
  --set bootstrapPassword=admin
```