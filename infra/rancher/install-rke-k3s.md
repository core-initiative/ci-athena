## Install RKE with k3s
Here is the step by step to install RKE with k3s, version v1.33.5+k3s1
### Prerequisites / Scenario
1. We have  node with public IP and private IP
2. Node private ip is setup using wireguard
3. Have a domain to be mapped as rancher ui ingress.
4. Full internet capability
5. Use superuser (`sudo -s`)

### Step By Step - Master Node
1. Prepare the `config.yaml` for master node
```yaml
write-kubeconfig-mode: "0644"
node-ip: <PRIVATE_IP>
node-external-ip: <PUBLIC_IP>
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
token: <TOKEN_STEP_1>
server: https://<MASTER_IP>:6443
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
