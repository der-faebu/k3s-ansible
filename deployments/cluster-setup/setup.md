## Install tools
The following tools are required to administer the k3s cluster:
- kubectl
- helm

## Run ansible deployement
Run ansible Deployment according to the readme in this repo.
After the deployment has run successfully, copy the kubeconfig from one of the master nodes into ~/.kube or set the variable KUBECONFIG to its path.

## Set up rancher

This guide follows the official rancher documentation.
https://ranchermanager.docs.rancher.com/getting-started/installation-and-upgrade/install-upgrade-on-a-kubernetes-cluster

Add the rancher helm repo
```
helm repo add rancher-stable https://releases.rancher.com/server-charts/stable && helm repo update
```
Create rancher namespace:
```
kubectl create namespace cattle-system
```
Install Cert manager (following their docs: https://cert-manager.io/docs/installation/helm/)

```
helm repo add jetstack https://charts.jetstack.io --force-update

helm install \
  cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --version v1.16.2 \
  --set crds.enabled=true
```

Now we can install rancher
```
helm install rancher rancher-stable/rancher \
  --namespace cattle-system \
  --set hostname=rancher.gygax.cloud \
  --set bootstrapPassword=admin
  ```

This should leave us with a functioning instance of rancher in our k3s cluster.

In order to access our rancher instance we now need to create a LoadBalancer object for it.
Be sure to set the IP address rancher should be exposed at in the yml file. If not, MetalLB will assign one from its configured range. Once Cloudflared is deployed, only a ClusterIP will be needed as traffic will be routed through CloudFlare.

```
kubectl apply -f deployments/cluster-setup/rancher/lb-service-rancher.yml
```
The rancher GUI can now be accessed via https://rancher.k3s-mini.gygax.cloud (where the hostname must point to the configured LoadBalancer IP)

## Longhorn
In order to take advantage of persistant storage, we install Longhorn.

This will be done following https://longhorn.io/docs/1.7.2/deploy/install/

### Requirements
Install the longhorn CLI.
```
curl -sSfL -o longhornctl https://github.com/longhorn/cli/releases/download/v1.7.2/longhornctl-linux-amd64
chmod +x ./longhornctl
```

As of v1.7.2 Longhorn provides an installer script for open-iscsi.
```
kubectl apply -f https://raw.githubusercontent.com/longhorn/longhorn/v1.7.2/deploy/prerequisite/longhorn-iscsi-installation.yaml
```

The following packages must be installed on all worker nodes:
- open-iscsi
- bash
- curl
- findmnt
- grep
- awk
- blkid
- lsblk
- cryptsetup
- nfs-common

he packages `open-iscsi` must be installed on all agent nodes. Also, the `iscsid` deamon must be enabled and started.
```
sudo systemctl start iscsid && sudo systemctl start iscsid
```
We can now check if all the requirements are fulfilled by using the longhorn CLI:

```
./longhornctl check preflight --kube-config ~/.kube/config
```
### Install Longhorn

Install Longhort via the Rancher GUI

## Traefik
In our cluster, we use Traefik as our load balanger / ingress controller.

This is a fairly complex installation consisting of multiple elements.
However, all the

### Cluster issuer
In order to be able to request Let's-Encrypt certificates via cert-manager, we need to setup a cluster issuer. This consists of mutliple parts:

- Cloudflare API secret for the DNS01-challenge
- Persistant volume for storing certificate data.
- The actual certificate(s). In our case we request a wildcard certificate.
    - This cert can be referenced in further deployments
    - I deem a wildcard cert sufficient in this case as traefik will not face the internet.

#### Create the secret
In order not to check in any sensitive data the secret containing the Cloudflare API key can be created on the fly with the following command. Alternatively, the corresponding section can be uncommented in the traefik deployment file.

Create the namespace
```
kubectl create namespace traefik
```

```
kubectl create secret generic cloudflare-api-credentials \
    --from-literal=email='<email>' \
    --from-literal=api-token='<api-key>' \
    --namespace traefik \
    --type opaque
```
Another way would be to store the values in files (.values directories will not be checked in)
The files must only contain the plain values. No trailing new line.
```
kubectl create cloudflare-api-credentials generic cloudflare-api-credentials \
    --from-file=email='deployments/cluster-setup/traefik/.values/cloudflare-email.yaml' \
    --from-file=api-token='.values/cloudflare-api-token.yaml' \
    --namespace traefik \
    --type opaque
```
Once this is done we can deploy the cluster issuer
```
kubectl apply -f deployments/cluster-setup/traefik/01-traefik-issuer-deployment.yaml
```

### Helm chart / values
Add the helm repo
```
helm repo add traefik https://helm.traefik.io/traefik
helm repo update
```
Update the values in deployments\cluster-setup\traefik\02-traefik-helm-values.yaml

Install traefik via helm
```
helm install traefik traefik/traefik --namespace=traefik --values=deployments/cluster-setup/traefik/02-traefik-helm-values.yaml
```

### Rancher ingress
To test the newly deployed traefik reverse proxy, we can now deploy an ingress for Ranger
```
kubectl apply -f deployments/clusters-setup/rancher/ingress-rancher.yaml
```

## Cloudflared
To take the most out of our setup we now deploy cloudflared. This allows us to use cloudflare tunnel.

### Tunnel token as secret
Create a new namespace called 'cloudlflared' and create a secret containing the tunnel token.

```
kubectl create namespace cloudflared
kubectl create secret generic cloudflare-tunnel-token --from-literal=token='<token>' --namespace cloudflare
```

Deploy the cloudflare pods.
```
kubectl apply -f deployments/cluster-setup/cloudflared/cloudflared-deployment.yaml
```
11. Deploy applications
