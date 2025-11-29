## Install RKE with k3s
Here is the step by step to install RKE with k3s, version v1.33.5+k3s1
### Prerequisites / Scenario
1. We have  node with public IP and private IP (interface wg0), we will use wg0 as flannel interface
2. Node private ip is setup using wireguard
3. Have a domain to be mapped as rancher ui ingress.
4. Full internet capability
5. Use superuser (`sudo -s`)
6. Allow the required port through ufw

### Step By Step - Networking
1. Make sure the default ufw forward is ACCEPT, edit file `/etc/default/ufw`, find the line for
```
DEFAULT_FORWARD_POLICY="ACCEPT"
```
2. Allow communication between wg0 (node to node communication)
```
ufw allow in on wg0
ufw allow out on wg0
```
3. Allow routing for internal communication inside k3s cluster
```
# Allow traffic passing through the WireGuard tunnel to other interfaces (like Pods)
ufw route allow in on wg0 out on any
ufw route allow in on any out on wg0

# Allow traffic passing through the Flannel/CNI interfaces (Internal Pod Traffic)
ufw route allow in on flannel.1 out on any
ufw route allow in on any out on flannel.1
ufw route allow in on cni0 out on any
ufw route allow in on any out on cni0
```
4. Trust internal interface ( flannel & and cni )
```
##OPTIONS A
ufw allow in on flannel.1
ufw allow out on flannel.1
ufw allow in on cni0
ufw allow out on cni0

## OPTIONS B
# Allow Pod Network communication
sudo ufw allow from 10.42.0.0/16 to any
sudo ufw allow from any to 10.42.0.0/16

# Allow Service Network communication
sudo ufw allow from 10.43.0.0/16 to any
sudo ufw allow from any to 10.43.0.0/16
```
5. Allow 22,80,443 tcp for basic access ( ssh, http and https )
```
ufw allow 22,80,443/tcp
```
6. Enable the firewall using `ufw enable`

### Step By Step - Master Node
1. Prepare the `config.yaml` for master node
```yaml
write-kubeconfig-mode: "0644"
node-ip: <PRIVATE_IP>
node-external-ip: <PUBLIC_IP>
flannel-iface: wg0
tls-san:
  - "<DOMAIN_NAME>"
  - "<PUBLIC_IP>"
  - "<PRIVATE_IP>"
```
2. Create directory, `mkdir -p /etc/rancher/k3s`
3. Save the yaml under `/etc/rancher/k3s/config.yaml`
4. Run the install script
```
curl -sfL https://get.k3s.io | sh -
```
5. Wait until all process are complete.
6. Add KUBE_CONFIG environment variable, edit your `~/.bashrc`, add this at the end of the file
```
export KUBECONFIG=/etc/rancher/k3s/k3s.yaml
```
7. Relogin to the SSH, then try to check the node
```
kubectl get nodes -o wide
```
You should see the result
```
NAME          STATUS   ROLES                  AGE   VERSION        INTERNAL-IP   EXTERNAL-IP      OS-IMAGE             KERNEL-VERSION     CONTAINER-RUNTIME
rke-master    Ready    control-plane,master   8h    v1.33.5+k3s1   10.100.0.1    103.82.93.82     Ubuntu 24.04.3 LTS   6.8.0-87-generic   containerd://2.1.4-k3s1
```

### Step By Step - Worker Node
1. Retrieve the token from master node
```
cat /var/lib/rancher/k3s/server/node-token
```
2. Create config.yaml at `/etc/rancher/k3s/config.yaml`, put these value
```yaml
node-ip: <PRIVATE_IP>
node-external-ip: <PUBLIC_IP>
flannel-iface: wg0
```
Also export variable for K3S_URL and K3S_TOKEN
```
export K3S_URL=https://<MASTER_IP>:6443
export K3S_TOKEN=<TOKEN>
```
3. Run the installation
```
curl -sfL https://get.k3s.io | sh -
```
4. Wait until all the process are complete.
5. Check the node availability in master using the same  command `kubectl get node -o wide`. Now you should see
```
NAME          STATUS   ROLES                  AGE   VERSION        INTERNAL-IP   EXTERNAL-IP      OS-IMAGE             KERNEL-VERSION     CONTAINER-RUNTIME
rke-master    Ready    control-plane,master   8h    v1.33.5+k3s1   10.100.0.1    103.82.93.82     Ubuntu 24.04.3 LTS   6.8.0-87-generic   containerd://2.1.4-k3s1
rke-worker1   Ready    <none>                 16m   v1.33.5+k3s1   10.100.0.2    103.172.204.82   Ubuntu 24.04.3 LTS   6.8.0-87-generic   containerd://2.1.4-k3s1
```
### Step By Step - Install Rancher UI
1. Make sure you have `helm` command install, if you have not installed yet, follow this instruction
```
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-4
chmod +x get_helm.sh
./get_helm.sh
```
Test the helm command with `helm list`,
```
root@rke-master2:/home/ubuntu# helm list
NAME	NAMESPACE	REVISION	UPDATED	STATUS	CHART	APP VERSION
```
2. Add helm repository for rancher-stable & jetstack
```
helm repo add rancher-stable https://releases.rancher.com/server-charts/stable
helm repo add jetstack https://charts.jetstack.io
helm repo update
helm repo list
```
You should see the output
```
root@rke-master2:/home/ubuntu# helm repo list
NAME          	URL
rancher-stable	https://releases.rancher.com/server-charts/stable
jetstack      	https://charts.jetstack.io
```
3. Install cert-manager, current version is v1.19.1. You can adjust with your preferred version
```
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.19.1/cert-manager.crds.yaml
helm install cert-manager jetstack/cert-manager   --namespace cert-manager   --version v1.19.1   --set installCRDs=false
```
Wait until all pods are ready, check by running `kubectl get pod -n cert-manager`
```
root@rke-master2:/home/ubuntu# kubectl get pods -n cert-manager
NAME                                       READY   STATUS    RESTARTS   AGE
cert-manager-77b74755d9-wm8h2              1/1     Running   0          86m
cert-manager-cainjector-65fcfd6ccf-js9w9   1/1     Running   0          86m
cert-manager-webhook-9b4dd78-2vgx5         1/1     Running   0          86m
```
4. Install Rancher UI, dont forget to change the `RANCHER_DOMAIN` and `ADMIN_PASSWORD`
```
kubectl create namespace cattle-system
helm install rancher rancher-stable/rancher   \
--namespace cattle-system   \
--set hostname=<RANCHER_DOMAIN>   \
--set bootstrapPassword=<ADMIN_PASSWORD>
```
5. Wait the process until complete, then check the pods readyness by running `kubectl get pods -n cattle-system`. You should see the output, make sure all pods are `1/1 running`
```
root@rke-master2:/home/ubuntu# kubectl get pods -n cattle-system
NAME                                        READY   STATUS    RESTARTS      AGE
rancher-84bc8d56bc-9d6gz                    1/1     Running   1 (87m ago)   88m
rancher-84bc8d56bc-g4jsd                    1/1     Running   0             88m
rancher-84bc8d56bc-hvwv6                    1/1     Running   2 (87m ago)   88m
rancher-webhook-65fd7d4ff-lpdkq             1/1     Running   0             86m
system-upgrade-controller-86f54779d-8sjwr   1/1     Running   0             85m
root@rke-master2:/home/ubuntu#
```
6. Go to your https://<RANCHER_DOMAIN>, login with your password.
