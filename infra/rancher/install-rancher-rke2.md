# Phase 1: Environment & OS Configuration

## 📋 Overview
This stage configures the system-wide proxy, disables conflicting OS services, and ensures that critical file systems allow binary execution.

## 🛠️ Audit: Mount Flags (noexec check)
RKE2 requires execution permissions on several paths. If these are separate partitions in `/etc/fstab`, ensure they do **not** contain the `noexec` flag:
* `/var/lib/rancher/rke2` (Primary RKE2 directory)
* `/var/lib/kubelet`
* `/tmp` (Used during installation and for certain runtime hooks)

## 🛠️ Network Requirements
Ensure the following ports are open on your physical/virtual firewall:

| Port | Protocol | Description |
| :--- | :--- | :--- |
| 80/443 | TCP | Rancher UI / Ingress |
| 6443 | TCP | Kubernetes API Server |
| 9345 | TCP | RKE2 Node Registration |

## 🚀 Execution Script
Run this script on **all nodes** (Master and Worker). You need to adjust the `PROXY_URL` and `NO_PROXY_LIST`

```bash
#!/bin/bash
# Host Preparation Script

PROXY_URL="http://<ip>:<port>"
# Internal CIDRs must be excluded from proxy
NO_PROXY_LIST="127.0.0.0/8,10.0.0.0/8,172.16.0.0/12,192.168.0.0/16,cattle-system.svc,.svc,.cluster.local"

echo "🔧 Configuring system proxy..."
cat <<EOF > /etc/environment
http_proxy=$PROXY_URL
https_proxy=$PROXY_URL
no_proxy=$NO_PROXY_LIST
EOF

echo "🔧 Configuring shell proxy (~/.bashrc)..."
cat <<EOF >> ~/.bashrc
export http_proxy=$PROXY_URL
export https_proxy=$PROXY_URL
export no_proxy=$NO_PROXY_LIST
EOF

source /etc/environment

echo "🛡️ Disabling Firewalld and SELinux..."
systemctl stop firewalld && systemctl disable firewalld
setenforce 0
sed -i 's/^SELINUX=enforcing/SELINUX=disabled/' /etc/selinux/config

echo "✅ Preparation complete. Please log out and back in."
```
# Phase 2: RKE2 Cluster Deployment

## 📋 Overview
RKE2 services run as systemd units. They require a dedicated proxy configuration file to successfully pull container images in air-gapped environments.

## 1. Configure RKE2 Systemd Proxy
Run this on **all nodes** before installing the RKE2 binary. Use `rke2-server` for master node or `rke2-agent` for worker node

```bash
#!/bin/bash
PROXY_URL="http://<proxy_ip>:<port>"
NO_PROXY_LIST="localhost,127.0.0.1,0.0.0.0,10.42.0.0/16,10.43.0.0/16,.svc,.cluster.local"
SERVICE="rke2-server"
# SERVICE="rke2-agent"
mkdir -p /etc/systemd/system/${SERVICE}.service.d
cat <<EOF > /etc/systemd/system/${SERVICE}.service.d/proxy.conf
[Service]
Environment="HTTP_PROXY=$PROXY_URL"
Environment="HTTPS_PROXY=$PROXY_URL"
Environment="NO_PROXY=$NO_PROXY_LIST"
EOF
done

systemctl daemon-reload
```
## 2. Initialize First Master
Run on Master Node 1. Note the tls-san includes the Load Balancer IP. If you plan to have multi master node, it's best to use a load balancer in front of the master node.

```bash
#!/bin/bash
LB_IP="<IP_ADDRESS>"

curl -sfL [https://get.rke2.io](https://get.rke2.io) | INSTALL_RKE2_TYPE=server sh -

mkdir -p /etc/rancher/rke2
cat <<EOF > /etc/rancher/rke2/config.yaml
tls-san:
  - ${LB_IP}
  - lb.igate-rke2.cluster
EOF

systemctl enable rke2-server.service --now
```
## 3. Join Additional Masters
Run on another master node using  the token from Node 1. Use this command to get the token from Master Node 1
`cat /var/lib/rancher/rke2/server/node-token`


```bash
#!/bin/bash
LB_IP="<IP_ADDRESS>"
TOKEN="<PASTE_TOKEN_HERE>"

curl -sfL [https://get.rke2.io](https://get.rke2.io) | INSTALL_RKE2_TYPE=server sh -

mkdir -p /etc/rancher/rke2
cat <<EOF > /etc/rancher/rke2/config.yaml
server: https://${LB_IP}:9345
token: ${TOKEN}
tls-san:
  - ${LB_IP}
EOF

systemctl enable rke2-server.service --now
```

# Phase 3: Rancher HA Installation

## 📋 Overview
Installation of the Rancher management plane via Helm charts.

## 1. Install Helm & Cert-Manager
Run from **Master Node 1**.

```bash
#!/bin/bash
# Install Helm
curl -fsSL -o get_helm.sh [https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3](https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3)
chmod +x get_helm.sh && ./get_helm.sh

# Install Cert-Manager
helm repo add jetstack [https://charts.jetstack.io](https://charts.jetstack.io)
helm repo update
kubectl apply -f [https://github.com/cert-manager/cert-manager/releases/download/<VERSION>/cert-manager.crds.yaml](https://github.com/cert-manager/cert-manager/releases/download/<VERSION>/cert-manager.crds.yaml)
helm install cert-manager jetstack/cert-manager --namespace cert-manager --create-namespace --version <VERSION>
```
For our setup we use VERSION v1.13.0
## 2. Install Rancher
Configure for High Availability (3 replicas). Replace `RANCHER_HOSTNAME` and `BOOTSTRAP_PASS` value with your own.
```bash
#!/bin/bash
RANCHER_HOSTNAME="<fqdn_hostname>"
BOOTSTRAP_PASS="<password>"

helm repo add rancher-latest [https://releases.rancher.com/server-charts/latest](https://releases.rancher.com/server-charts/latest)
helm install rancher rancher-latest/rancher \
  --namespace cattle-system \
  --create-namespace \
  --set hostname=${RANCHER_HOSTNAME} \
  --set replicas=3 \
  --set bootstrapPassword=${BOOTSTRAP_PASS}
```
Wait a few minutes for the script to provision rancher instance, then check the pods that is running. It should look like this
```
kubectl get pods --all-namespaces

NAMESPACE                   NAME                                      READY   STATUS      RESTARTS   AGE
cattle-fleet-local-system   fleet-agent-699b5fb945-rkbbg              1/1     Running     0          62m
cattle-fleet-system         fleet-controller-6d95df949f-qsrg7         1/1     Running     0          63m
cattle-fleet-system         gitjob-67df6b78d4-xc8cx                   1/1     Running     0          63m
cattle-system               rancher-979ffccc5-2jgkt                   1/1     Running     0          68m
cattle-system               rancher-webhook-5b65595df9-q5z4l          1/1     Running     0          62m
cert-manager                cert-manager-5bf9d49bbd-54j5b             1/1     Running     0          126m
cert-manager                cert-manager-cainjector-9b679cc6-pct6j    1/1     Running     0          126m
cert-manager                cert-manager-webhook-57c994b6b9-sgdjq     1/1     Running     0          126m
kube-system                 coredns-d76bd69b-2tchp                    1/1     Running     0          130m
kube-system                 helm-install-traefik-crd-6jj5b            0/1     Completed   0          130m
kube-system                 helm-install-traefik-h9rr2                0/1     Completed   0          130m
kube-system                 local-path-provisioner-6c79684f77-n6lsd   1/1     Running     0          130m
kube-system                 metrics-server-7cd5fcb6b7-gvt7j           1/1     Running     0          130m
kube-system                 svclb-traefik-5882b881-nwvt7              2/2     Running     0          129m
kube-system                 traefik-df4ff85d6-5flth                   1/1     Running     0          129m
```
# Phase 4: Security Hardening & Verification

## 🔒 Security Headers
Apply custom headers to the Nginx Ingress controller via HelmChartConfig.
1. Create yaml config `ingress-headers.yaml` with values below.
```yaml
apiVersion: helm.cattle.io/v1
kind: HelmChartConfig
metadata:
  name: rke2-ingress-nginx
  namespace: kube-system
spec:
  valuesContent: |-
    controller:
      config:
        server-snippet: |
          add_header Strict-Transport-Security "max-age=31536000 ; includeSubDomains" always;
          add_header X-Frame-Options "deny" always;
          add_header X-Content-Type-Options "nosniff" always;
          add_header Referrer-Policy "no-referrer-when-downgrade" always;
          add_header Content-Security-Policy "script-src 'self' 'unsafe-eval'; worker-src-blob 'self'; style-src 'unsafe-inline 'self'; frame-ancestors 'self'" always;
          add_header Cross-Origin-Embedder-Policy "require-corp" always;
          add_header Cross-Origin-Opener-Policy "same-origin" always;
          add_header Cross-Origin-Resource-Policy "same-origin" always;
          add_header Permissions-Policy "accelerometer=(),ambient-light-sensor=(),autoplay=(),battery=(),camera=(),display-capture=(),document-domain=(),encrypted-media=(),fullscreen=(),gamepad=(),geolocation=(),gyroscope=(),layout-animations=(self),legacy-image-formats=(self),magnetometer=(),microphone=(),midi=(),oversized-images=(self),payment=(),picture-in-picture=(),publickey-credentials-get=(),speaker-selection=(),sync-xhr=(self),unoptimized-images=(self),unsized-media=(self),usb=(),screen-wake-lock=(),web-share=(),xr-spatial-tracking=()" always;
```
2. Apply config, kubectl apply -f ingress-headers.yaml
3. Make user the job is running, check it with `journalctl -u rke2-server -f`
4. If error failed to sync, or requeuing then delete the stuck jobs.
5. Check the stuck job, if the status is pending / completed just delete it
```bash
kubectl get job helm-install-rke2-ingress-nginx -n kube-system
kubectl delete job helm-install-rke2-ingress-nginx -n kube-system
```
6. Wait unitl the  job is complete and daemonset restart succesfully. Then run the command bellow to check the status.
```bash
kubectl -n kube-system rollout status ds rke2-ingress-nginx-controller
```
