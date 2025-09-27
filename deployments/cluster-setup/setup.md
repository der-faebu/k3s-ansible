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

MetalLB reserved IPs:
- 172.18.1.51 --> Traefik
- 172.18.1.52 --> Audiobookshelf
- 172.18.1.53 --> Syncthing
- 172.18.1.56 --> Rancher

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

### Create Azure Service principal

Ensure azure cli and jq are insatlled

```bash
# Choose a name for the service principal that contacts azure DNS to present
# the challenge.
AZURE_CERT_MANAGER_NEW_SP_NAME=sp-k3s-certmanager
# This is the name of the resource group that you have your dns zone in.
AZURE_DNS_ZONE_RESOURCE_GROUP=gygax-tech-dns
# The DNS zone name. It should be something like domain.com or sub.domain.com.
AZURE_DNS_ZONE=gygax.cloud

DNS_SP=$(az ad sp create-for-rbac --name $AZURE_CERT_MANAGER_NEW_SP_NAME --output json)
AZURE_CERT_MANAGER_SP_APP_ID=$(echo $DNS_SP | jq -r '.appId')
AZURE_CERT_MANAGER_SP_PASSWORD=$(echo $DNS_SP | jq -r '.password')
AZURE_TENANT_ID=$(echo $DNS_SP | jq -r '.tenant')
AZURE_SUBSCRIPTION_ID=$(az account show --output json | jq -r '.id')

az role assignment delete --assignee $AZURE_CERT_MANAGER_SP_APP_ID --role Contributor

DNS_ID=$(az network dns zone show --name $AZURE_DNS_ZONE --resource-group $AZURE_DNS_ZONE_RESOURCE_GROUP --query "id" --output tsv)

az role assignment create --assignee $AZURE_CERT_MANAGER_SP_APP_ID --role "DNS Zone Contributor" --scope $DNS_ID

az role assignment list --all --assignee $AZURE_CERT_MANAGER_SP_APP_ID


echo "AZURE_CERT_MANAGER_SP_APP_ID: $AZURE_CERT_MANAGER_SP_APP_ID"
echo "AZURE_CERT_MANAGER_SP_PASSWORD: $AZURE_CERT_MANAGER_SP_PASSWORD"
echo "AZURE_SUBSCRIPTION_ID: $AZURE_SUBSCRIPTION_ID"
echo "AZURE_TENANT_ID: $AZURE_TENANT_ID"
echo "AZURE_DNS_ZONE: $AZURE_DNS_ZONE"
echo "AZURE_DNS_ZONE_RESOURCE_GROUP: $AZURE_DNS_ZONE_RESOURCE_GROUP"

```
These values can then be inserted into the issuer deployment:

```yml
apiVersion: cert-manager.io/v1
kind: Issuer
metadata:
  name: example-issuer
spec:
  acme:
    ...
    solvers:
    - dns01:
        azureDNS:
          clientID: AZURE_CERT_MANAGER_SP_APP_ID
          clientSecretSecretRef:
            name: azuredns-config
            key: client-secret
          subscriptionID: AZURE_SUBSCRIPTION_ID
          tenantID: AZURE_TENANT_ID
          resourceGroupName: AZURE_DNS_ZONE_RESOURCE_GROUP
          hostedZoneName: AZURE_DNS_ZONE
          # Azure Cloud Environment, default to AzurePublicCloud
          environment: AzurePublicCloud
```

#### Create the secret
This step assumes the session of the previuos step still exists as variables need to be solvable
In order not to check in any sensitive data the secret containing the Cloudflare API key can be created on the fly with the following command. Alternatively, the corresponding section can be uncommented in the traefik deployment file.

Create the namespace
```
kubectl create namespace traefik
```

```
kubectl create secret generic azuredns-config --from-literal=client-secret=$AZURE_CERT_MANAGER_SP_PASSWORD
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
helm install traefik traefik/traefik --namespace=traefik --values=deployments/cluster-setup/traefik/traefik-helm-values.yaml
```

### Rancher ingress
To test the newly deployed traefik reverse proxy, we can now deploy an ingress for Ranger
```
kubectl apply -f deployments/clusters-setup/rancher/ingress-rancher.yaml
```

## Deploy applications
The following repos are in use (not counting infrastrucutre charts installed above like traefik):
  - bjw-s                   https://bjw-s-labs.github.io/helm-charts
  - k8s-home-lab            https://k8s-home-lab.github.io/helm-charts/
  - k8s-at-home             https://k8s-at-home.com/charts/
  - jellyfin                https://jellyfin.github.io/jellyfin-helm
  - k8s-at-home-charts      https://library-charts.k8s-at-home.com
  - gabe565                 https://charts.gabe565.com

To add these repos run this:
``` bash
helm repo add bjw-s https://bjw-s-labs.github.io/helm-charts
helm repo add k8s-home-lab https://k8s-home-lab.github.io/helm-charts/
helm repo add k8s-at-home https://k8s-at-home.com/charts/
helm repo add jellyfin https://jellyfin.github.io/jellyfin-helm
helm repo add k8s-at-home-charts https://library-charts.k8s-at-home.com
helm repo add gabe565 https://charts.gabe565.com
helm repo update
```

All applications shloud be setup in a similar and consistant way. Whenever possible a maintained chart from k8s-home-lab should be used and preferred. It is however possible to use the app template provided by https://bjw-ss.github.io 

tl;dr `helm install <release-name> bjw-s/app-template -f values.yaml`

All deployed charts are kept in the `helm-deployments.md` file.

### Gluetun
In order to setup gluetun a nordvpn private key must be extracted. In order to achieve this, the following prerequisites must be fulfilled:
- install nordvpn on a linux machine (WSL should work)
- authenticate with nordvpn
- set the vpn protocol to 'nordlynx'
- retrieve the pk using this command:
  `sudo wg show nordlynx private-key`

Create a kubernetes secret containing the private key:
```
kubectl create secret generic wg-private-key \
     --from-literal=wg-private-key='<api-token>' \
     --namespace <namespace> \
     --type opaque
```
