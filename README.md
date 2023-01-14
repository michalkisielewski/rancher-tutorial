# Managing Kubernetes with Rancher

## Useful commands
* list VMs with IPs: 
    ```
    virsh net-list
    virsh net-dhcp-leases default
    ```

## Install necessary tools on local machine
```curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
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
    echo "server: <rancher-server-1-IP-address>:9345" >> /etc/rancher/rke2/config.yaml

    curl -sfL https://get.rke2.io | sh -
    systemctl enable rke2-server.service
    systemctl start rke2-server.service
    ```

    For more details see docs: https://ranchermanager.docs.rancher.com/v2.6/how-to-guides/new-user-guides/kubernetes-cluster-setup/rke2-for-rancher

### Copy kubeconfig to local machine
sudo mkdir .kube/config/
sudo virt-copy-out -d rancher-server-1 /etc/rancher/rke2/rke2.yaml .kube/config/

### Connect to machine via kubectl
kubectl --kubeconfig rke2.yaml get pods -A

## Install Rancher
https://ranchermanager.docs.rancher.com/pages-for-subheaders/install-upgrade-on-a-kubernetes-cluster

```
helm install rancher rancher-latest/rancher \
  --namespace cattle-system \
  --set hostname=192.168.122.101.sslip.io \
  --set bootstrapPassword=admin
```