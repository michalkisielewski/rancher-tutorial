# Managing Kubernetes with Rancher

## Useful commands
* list VMs with IPs: 
    ```bash
    virsh net-list
    virsh net-dhcp-leases default
    ```
* clean dhcp leases from VMs:
    ```bash
    sudo rm var/lib/libvirt/dnsmasq/virbr0.*
    ```    
* clone new VM:
    ```bash
    export NEW_VM_NAME=""
    virt-clone --original empty --auto-clone -n $NEW_VM_NAME
    sudo virt-sysprep -d $NEW_VM_NAME --hostname $NEW_VM_NAME --enable net-hostname,customize
    ```


## Install necessary tools on local machine
kubectl:
```bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
```

helm:
```bash
curl https://baltocdn.com/helm/signing.asc | gpg --dearmor | sudo tee /usr/share/keyrings/helm.gpg > /dev/null
sudo apt-get install apt-transport-https --yes
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/helm.gpg] https://baltocdn.com/helm/stable/debian/ all main" | sudo tee /etc/apt/sources.list.d/helm-stable-debian.list
sudo apt-get update
sudo apt-get install helm
```

krew:
```bash
(
  set -x; cd "$(mktemp -d)" &&
  OS="$(uname | tr '[:upper:]' '[:lower:]')" &&
  ARCH="$(uname -m | sed -e 's/x86_64/amd64/' -e 's/\(arm\)\(64\)\?.*/\1\2/' -e 's/aarch64$/arm64/')" &&
  KREW="krew-${OS}_${ARCH}" &&
  curl -fsSLO "https://github.com/kubernetes-sigs/krew/releases/latest/download/${KREW}.tar.gz" &&
  tar zxvf "${KREW}.tar.gz" &&
  ./"${KREW}" install krew
)
export PATH="${KREW_ROOT:-$HOME/.krew}/bin:$PATH"
kubectl krew
```

additional tools (via krew)
```bash
kubectl krew install ctx
kubectl krew install ns

# set proper permissions for above:
sudo chown $(id -u):$(id -g) $HOME/.kube/config
sudo chown -R $(id -u):$(id -g) $HOME/.kube/
```

merge kubeconfigs for kubectx:
```bash
KUBECONFIG=file1:file2 kubectl config view --merge --flatten > out.txt
mv out.txt .kube/config
```

## Setup Rancher Server
* start rancher-server-X virtual machines
* setup rancher-server-1:
    ```bash
    sudo su

    mkdir -p /etc/rancher/rke2/
    echo "token: my-shared-secret" >> /etc/rancher/rke2/config.yaml

    curl -sfL https://get.rke2.io | sh -
    systemctl enable rke2-server.service
    systemctl start rke2-server.service
    ```
* setup other rancher-server nodes:
    ```bash
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
```bash
sudo mkdir .kube
sudo virt-copy-out -d rancher-server-1 /etc/rancher/rke2/rke2.yaml .
sudo mv rke2.yaml .kube/config
```
Config from the VM contains incorrect host (localhost), so it has to be changed appriopriately.

### Verify connection to Rancher Server from localhost
```bash
# with kubeconfig specified
kubectl --kubeconfig rke2.yaml get pods -A

# if config was saved to default .kube/config location
kubectl get pods -A
```

## Install Rancher
```bash
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

For more info look at docs:
https://ranchermanager.docs.rancher.com/pages-for-subheaders/install-upgrade-on-a-kubernetes-cluster

## Setting up new cluster

1. Navigate to Rancher management UI
1. Go to cluster creation and use the switch for RKE2 
1. Create cluster with the default settings
1. Join nodes appropriatelly with the command given by the setup

## Load balancing
1. Navigate to Rancher management UI
1. Go to App -> Charts
1. Install MetalLB

## Storage
1. Navigate to Rancher management UI
1. Go to App -> Charts
1. Install Longhorn
    Make sure `Default Storage Class` is checked in settings
1. Do a `kubectl port-forward` do Longhorn UI

Node configuration:
- disable worker nodes (to use only storage nodes)
- add tags for nodes

Additional configuration:
- configure backups for Longhorn (NFS or S3, where S3 can be handled by integration with Minio)
- create snapshots & backups of volumes

Setting up monitoring:
https://longhorn.io/docs/1.4.0/monitoring/prometheus-and-grafana-setup/

## Monitoring
1. Navigate to Rancher management UI
1. Go to App -> Charts
1. Install Rancher Monitoring
1. Go to Monitoring (new menu)

## Logging
1. Navigate to Rancher management UI
1. Go to App -> Charts
1. Install Rancher Logging

## Rancher cluster backup

1. Navigate to Rancher management UI
1. Go to App -> Charts
1. Install Rancher Backups
    Specify StorageClass as Longhorn
1. Go to Rancher Backups (new menu)
1. Create a backup

## Rancher API


## Troubleshooting

In some scenarios clusters created by Rancher might get stuck (eg. have no connection to Rancher Server). In this case it is not possible to remove the cluster from the UI. However, it is possible to forcefully remove such cluster:
```bash
kubectl get clusters.management.cattle.io  # find the cluster you want to delete 
export CLUSTERID="c-xxxxxxxxx" # 
kubectl patch clusters.management.cattle.io $CLUSTERID -p '{"metadata":{"finalizers":[]}}' --type=merge
kubectl delete clusters.management.cattle.io $CLUSTERID
```