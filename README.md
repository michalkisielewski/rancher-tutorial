# Managing Kubernetes with Rancher

## Useful commands
* list VMs with IPs: 
    ```
    virsh net-list
    virsh net-dhcp-leases default
    ```


## Initial steps
1. Install Rancher Server
https://ranchermanager.docs.rancher.com/v2.6/how-to-guides/new-user-guides/kubernetes-cluster-setup/rke2-for-rancher

2. Install kubectl on local machine
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl

3. Install helm on local machine:
curl https://baltocdn.com/helm/signing.asc | gpg --dearmor | sudo tee /usr/share/keyrings/helm.gpg > /dev/null
sudo apt-get install apt-transport-https --yes
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/helm.gpg] https://baltocdn.com/helm/stable/debian/ all main" | sudo tee /etc/apt/sources.list.d/helm-stable-debian.list
sudo apt-get update
sudo apt-get install helm

4. Copy kubeconfig
sudo virt-copy-out -d rancher-server-1 /etc/rancher/rke2/rke2.yaml .kube/config/

5. Connect to machine via kubectl
kubectl --kubeconfig rke2.yaml get pods -A

6. Install Rancher
https://ranchermanager.docs.rancher.com/pages-for-subheaders/install-upgrade-on-a-kubernetes-cluster

```
helm install rancher rancher-latest/rancher \
  --namespace cattle-system \
  --set hostname=192.168.122.101.sslip.io \
  --set bootstrapPassword=admin
```