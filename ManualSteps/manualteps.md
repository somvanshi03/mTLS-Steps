
## create VM on azure
```bash
# Resource Group
az group create --name demo-rg --location eastus

# vNet with Subnet
az network vnet create --resource-group demo-rg --name demo-vnet --address-prefix 10.0.0.0/16 --subnet-name demo-subnet --subnet-prefix 10.0.1.0/24

# NSG
az network nsg create --resource-group demo-rg --name demo-nsg

# Create Inbound rule
az network nsg rule create --resource-group demo-rg --nsg-name demo-nsg --name allow-ssh --priority 1000 --direction Inbound --access Allow --protocol Tcp --destination-port-range 22 --source-address-prefixes '*' --destination-address-prefixes '*'

# Attach NSG with subnet
az network vnet subnet update --resource-group demo-rg --vnet-name demo-vnet --name demo-subnet --network-security-group demo-nsg

# Create VM
az vm create --resource-group demo-rg --name ubuntu22-vm --vnet-name demo-vnet --subnet demo-subnet --image Ubuntu2204 --size Standard_B2s --admin-username azureuser --authentication-type password --admin-password 'Str0ngP@ssword2026!' --public-ip-sku Standard

# Show Public IP
az vm show --resource-group demo-rg --name ubuntu22-vm -d --query publicIps -o tsv

```

## Install helmfile and helmchart with diff
```bash
curl -L -o helmfile.tar.gz https://github.com/helmfile/helmfile/releases/download/v0.155.0/helmfile_0.155.0_linux_amd64.tar.gz

tar -xzf helmfile.tar.gz
chmod +x helmfile
sudo mv helmfile /usr/local/bin/
helmfile --version


curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
helm version

helm plugin install https://github.com/databus23/helm-diff


```

## Create AKS Cluster 

```bash 
az group create --location eastus --resource-group agicdemo

# Create AKS Cluser
az aks create --name agic-cluster \
              --resource-group agicdemo \
              --network-plugin azure \
              --enable-managed-identity \
              --enable-addons ingress-appgw \
              --appgw-name agic-appgw \
              --appgw-subnet-cidr "10.225.0.0/16" \
              --node-count 1 \
              --enable-cluster-autoscaler \
              --min-count 1 \
              --max-count 2 \
              --node-vm-size Standard_F2as_v6 \
              --generate-ssh-keys 
			  

# Get application gateway id from AKS addon profile
appGatewayId=$(az aks show -n agic-cluster -g agicdemo -o tsv --query "addonProfiles.ingressApplicationGateway.config.effectiveApplicationGatewayId")
echo $appGatewayId

# Get Application Gateway subnet id
appGatewaySubnetId=$(az network application-gateway show --ids $appGatewayId -o tsv --query "gatewayIPConfigurations[0].subnet.id")
echo $appGatewaySubnetId

# Get AGIC addon identity
agicAddonIdentity=$(az aks show -n agic-cluster -g agicdemo -o tsv --query "addonProfiles.ingressApplicationGateway.identity.clientId")
echo $agicAddonIdentity

# Assign network contributor role to AGIC addon Managed Identity to subnet that contains the Application Gateway
az role assignment create --assignee $agicAddonIdentity --scope $appGatewaySubnetId --role "Network Contributor"



# configure kubeconfig 
az aks get-credentials --resource-group agicdemo --name agic-cluster
az aks get-credentials -g agicdemo -n agic-cluster --overwrite-existing

# List Kubernetes Nodes
kubectl get nodes

# Check node size
az aks show -g agicdemo -n agic-cluster --query "agentPoolProfiles[].vmSize"

# Delete Resource Group 
az group delete -n agicdemo


appGatewayId=$(az aks show -n agic-cluster -g agicdemo -o tsv --query "addonProfiles.ingressApplicationGateway.config.effectiveApplicationGatewayId")

appGatewaySubnetId=$(MSYS_NO_PATHCONV=1 az network application-gateway show --ids "$appGatewayId" -o tsv --query "gatewayIPConfigurations[0].subnet.id")

agicIdentityClientId=$(az aks show -n agic-cluster -g agicdemo -o tsv --query "addonProfiles.ingressApplicationGateway.identity.clientId")

MSYS_NO_PATHCONV=1 az role assignment create --assignee "$agicIdentityClientId" --role "Network Contributor" --scope "$appGatewaySubnetId"

az aks show -g agicdemo -n agic-cluster --query "agentPoolProfiles[].vmSize"

az aks get-credentials -g agicdemo -n agic-cluster --overwrite-existing

```
## Install kubectl
```bash
sudo apt update
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.31/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.31/deb/ /" | sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo apt update
sudo apt install -y kubectl
kubectl version --client

```
## Install az cli
```bash
sudo apt update
sudo apt install -y ca-certificates curl apt-transport-https lsb-release gnupg
curl -sLS https://packages.microsoft.com/keys/microsoft.asc | \
gpg --dearmor | sudo tee /etc/apt/trusted.gpg.d/microsoft.gpg > /dev/null
AZ_REPO=$(lsb_release -cs)

echo "deb [arch=amd64] https://packages.microsoft.com/repos/azure-cli/ $AZ_REPO main" | \
sudo tee /etc/apt/sources.list.d/azure-cli.list
sudo apt update
sudo apt install -y azure-cli
az version
```

## Check testing
```bash
# Use sh instead of bash
kubectl exec -it curl-test -n flows-pen-test -- sh

# Test direct connection (should be blocked)
curl -vk --connect-timeout 5 https://r1r2.3disystems.com:443

# Test various paths through HAProxy
curl -v http://haproxy-egress:8443/api -H "Host: r1r2.3disystems.com"
curl -v http://haproxy-egress:8443/v1 -H "Host: r1r2.3disystems.com"
curl -v http://haproxy-egress:8443/health -H "Host: r1r2.3disystems.com"
curl -v http://haproxy-egress:8443/status -H "Host: r1r2.3disystems.com"

# Check environment variables
env | grep -E "DIRECT|HAPROXY|CMD"


# Test direct connection (should be blocked)
kubectl exec curl-test -n flows-pen-test -- curl -vk --connect-timeout 5 https://r1r2.3disystems.com:443

# Try common API paths
kubectl exec curl-test -n flows-pen-test -- curl -v http://haproxy-egress:8443/api -H "Host: r1r2.3disystems.com"
kubectl exec curl-test -n flows-pen-test -- curl -v http://haproxy-egress:8443/v1 -H "Host: r1r2.3disystems.com"
kubectl exec curl-test -n flows-pen-test -- curl -v http://haproxy-egress:8443/health -H "Host: r1r2.3disystems.com"
kubectl exec curl-test -n flows-pen-test -- curl -v http://haproxy-egress:8443/status -H "Host: r1r2.3disystems.com"
kubectl exec curl-test -n flows-pen-test -- curl -v http://haproxy-egress:8443/swagger -H "Host: r1r2.3disystems.com"
```
