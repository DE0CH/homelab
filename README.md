```bash
source .env
export HCLOUD_TOKEN=$HCLOUD_TOKEN
packer init hcloud.pkr.hcl
packer build hcloud.pkr.hcl
export IMAGE_ID=<image id from the output of the previous command>
```

```bash
hcloud context create talos-cluster
```

```
cd talos-config
talosctl gen config homelab https://k8s.deyaochen.com:6443 --with-examples=false --with-docs=false
```


Edit the `controlplane.yaml` file to include so you only need one single machine.

```yaml
cluster:
    allowSchedulingOnControlPlanes: true
```

Skip creating load balancer because it it too expensive and we only have one control plane

Create the server

```
hcloud server create --name talos-control-plane-1 \
    --image ${IMAGE_ID} \
    --type cx22 --location nbg1 \
    --user-data-from-file controlplane.yaml
```

Apply firewall with only TCP 6443 and TCP 50000 inbound allowed. All outbound allowed.

Get the ips

```
hcloud server list | grep talos-control-plane
```

```
talosctl --talosconfig talosconfig config endpoint <control-plane-1-IP>
talosctl --talosconfig talosconfig config node <control-plane-1-IP>
talosctl --talosconfig talosconfig bootstrap
```

Go to cloudflare console and update the dns record

Generate kubeconfig

```
talosctl --talosconfig talosconfig kubeconfig .
export KUBECONFIG=$PWD/kubeconfig
```


Consider adding Hetzner’s Cloud Controller Manager later since I don't probably need to managed the nodes and network through k8s API

Install Hetzner CSI driver

```
kubectl create secret generic hcloud --from-literal=token=$HCLOUD_TOKEN -n kube-system
kubectl apply -f https://raw.githubusercontent.com/hetznercloud/csi-driver/v2.16.0/deploy/kubernetes/hcloud-csi.yml
```

Deploy a web dashboard

```
# Add kubernetes-dashboard repository
helm repo add kubernetes-dashboard https://kubernetes.github.io/dashboard/
# Deploy a Helm Release named "kubernetes-dashboard" using the kubernetes-dashboard chart
helm upgrade --install kubernetes-dashboard kubernetes-dashboard/kubernetes-dashboard --create-namespace --namespace kubernetes-dashboard
kubectl -n kubernetes-dashboard create serviceaccount admin
kubectl create clusterrolebinding kubernetes-dashboard-admin \
  --clusterrole=cluster-admin \
  --serviceaccount=kubernetes-dashboard:admin
```

Port forward it

```
kubectl -n kubernetes-dashboard port-forward svc/kubernetes-dashboard-kong-proxy 8443:443
```

If it asks for token, get the token by 

```
kubectl -n kubernetes-dashboard create token admin
```

Installed sealed secrets

```
kubectl apply -f https://github.com/bitnami-labs/sealed-secrets/releases/download/v0.30.0/controller.yaml
```
