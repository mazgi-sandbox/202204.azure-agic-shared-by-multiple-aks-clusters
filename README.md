# 202204.azure-agic-shared-by-multiple-aks-clusters

## Requirements

- Azure Subscription
- [Azure Cloud Shell(bash)](https://azure.microsoft.com/en-us/features/cloud-shell/)

## How to Try

Determine your values.

```console
subsctiptionId="********"
servicePrincipalName="agic-with-multiple-aks"
resourceGroupName="agic-with-multiple-aks"
location="westus2"
```

### Create the Service Principal

```console
az ad sp create-for-rbac --name $servicePrincipalName --role Contributor --scopes /subscriptions/${subsctiptionId} -o json > auth.json
```

```console
jq "." auth.json
jq -r ".appId" auth.json
appId=$(jq -r ".appId" auth.json)
jq -r ".password" auth.json
password=$(jq -r ".password" auth.json)
```

```console
az ad sp show --id $appId --query "objectId" -o tsv
objectId=$(az ad sp show --id $appId --query "objectId" -o tsv)
```

```console
cat <<EOF > parameters.json
{
  "aksServicePrincipalAppId": { "value": "$appId" },
  "aksServicePrincipalClientSecret": { "value": "$password" },
  "aksServicePrincipalObjectId": { "value": "$objectId" },
  "aksEnableRBAC": { "value": false }
}
EOF
jq "." parameters.json
```

### Create Resources using ARM Template

```console
deploymentName="deployment-${resourceGroupName}"
```

```console
az group create -n $resourceGroupName -l $location
```

```console
az deployment group create -g $resourceGroupName -n $deploymentName --template-file azuredeploy.json --parameters parameters.json
```

```console
az deployment group show -g $resourceGroupName -n $deploymentName --query "properties.outputs" -o json > deployment-outputs.json
jq "." deployment-outputs.json
```

### Set up AGIC

```console
# use the deployment-outputs.json created after deployment to get the cluster name and resource group name
jq -r ".aksGreenClusterName.value" deployment-outputs.json
aksGreenClusterName=$(jq -r ".aksGreenClusterName.value" deployment-outputs.json)
jq -r ".aksBlueClusterName.value" deployment-outputs.json
aksBlueClusterName=$(jq -r ".aksBlueClusterName.value" deployment-outputs.json)
jq -r ".resourceGroupName.value" deployment-outputs.json
resourceGroupName=$(jq -r ".resourceGroupName.value" deployment-outputs.json)
```

```console
az aks get-credentials --resource-group $resourceGroupName --name $aksGreenClusterName
az aks get-credentials --resource-group $resourceGroupName --name $aksBlueClusterName
```

```console
kubectl --context $aksGreenClusterName get pods,services,ingresses,AzureIngressProhibitedTargets
kubectl --context $aksBlueClusterName get pods,services,ingresses,AzureIngressProhibitedTargets
```

```console
kubectl create --context $aksGreenClusterName -f https://raw.githubusercontent.com/Azure/aad-pod-identity/master/deploy/infra/deployment.yaml
kubectl create --context $aksBlueClusterName -f https://raw.githubusercontent.com/Azure/aad-pod-identity/master/deploy/infra/deployment.yaml
```

```console
kubectl --context $aksGreenClusterName get pods,services,ingresses,AzureIngressProhibitedTargets
kubectl --context $aksBlueClusterName get pods,services,ingresses,AzureIngressProhibitedTargets
```

### Set up application-gateway-kubernetes-ingress via Helm

```console
helm repo add application-gateway-kubernetes-ingress https://appgwingress.blob.core.windows.net/ingress-azure-helm-package/
helm repo update
```

```console
jq -r ".applicationGatewayName.value" deployment-outputs.json
applicationGatewayName=$(jq -r ".applicationGatewayName.value" deployment-outputs.json)
jq -r ".resourceGroupName.value" deployment-outputs.json
resourceGroupName=$(jq -r ".resourceGroupName.value" deployment-outputs.json)
jq -r ".subscriptionId.value" deployment-outputs.json
subscriptionId=$(jq -r ".subscriptionId.value" deployment-outputs.json)
jq -r ".identityClientId.value" deployment-outputs.json
identityClientId=$(jq -r ".identityClientId.value" deployment-outputs.json)
jq -r ".identityResourceId.value" deployment-outputs.json
identityResourceId=$(jq -r ".identityResourceId.value" deployment-outputs.json)
```

```console
sed -i "s|<subscriptionId>|${subscriptionId}|g" helm-config.yaml
sed -i "s|<resourceGroupName>|${resourceGroupName}|g" helm-config.yaml
sed -i "s|<applicationGatewayName>|${applicationGatewayName}|g" helm-config.yaml
sed -i "s|<identityResourceId>|${identityResourceId}|g" helm-config.yaml
sed -i "s|<identityClientId>|${identityClientId}|g" helm-config.yaml
cat helm-config.yaml
```

```console
helm install --kube-context $aksGreenClusterName --generate-name -f helm-config.yaml application-gateway-kubernetes-ingress/ingress-azure
helm install --kube-context $aksBlueClusterName --generate-name -f helm-config.yaml application-gateway-kubernetes-ingress/ingress-azure
```

```console
kubectl --context $aksGreenClusterName get pods,services,ingresses,AzureIngressProhibitedTargets
kubectl --context $aksBlueClusterName get pods,services,ingresses,AzureIngressProhibitedTargets
```

### Deploy an App on each AKS Cluster

```console
kubectl apply --context $aksGreenClusterName -f app.on-green.yaml
kubectl apply --context $aksBlueClusterName -f app.on-blue.yaml
```

```console
kubectl --context $aksGreenClusterName get pods,services,ingresses,AzureIngressProhibitedTargets
kubectl --context $aksBlueClusterName get pods,services,ingresses,AzureIngressProhibitedTargets
```

```console
kubectl apply --context $aksGreenClusterName -f agic-prohibit.for-green.yaml
kubectl apply --context $aksBlueClusterName -f agic-prohibit.for-blue.yaml
```

```console
kubectl --context $aksGreenClusterName get pods,services,ingresses,AzureIngressProhibitedTargets
kubectl --context $aksBlueClusterName get pods,services,ingresses,AzureIngressProhibitedTargets
```

```console
kubectl delete AzureIngressProhibitedTargets prohibit-all-targets --context $aksGreenClusterName
kubectl delete AzureIngressProhibitedTargets prohibit-all-targets --context $aksBlueClusterName
```

```console
kubectl --context $aksGreenClusterName get pods,services,ingresses,AzureIngressProhibitedTargets
kubectl --context $aksBlueClusterName get pods,services,ingresses,AzureIngressProhibitedTargets
```

```console
jq -r ".applicationGatewayPublicIpAddress.value" deployment-outputs.json
applicationGatewayPublicIpAddress=$(jq -r ".applicationGatewayPublicIpAddress.value" deployment-outputs.json)
curl -H 'Host: green.example.com' -I $applicationGatewayPublicIpAddress
curl -H 'Host: blue.example.com' -I $applicationGatewayPublicIpAddress
```

## References

- [How to Install an Application Gateway Ingress Controller (AGIC) Using a New Application Gateway](https://docs.microsoft.com/en-us/azure/application-gateway/ingress-controller-install-new)
- [Brownfield Deployment#Multi-cluster / Shared App Gateway](https://azure.github.io/application-gateway-kubernetes-ingress/setup/install-existing/#multi-cluster-shared-app-gateway)
- [Azure/application-gateway-kubernetes-ingress](https://github.com/Azure/application-gateway-kubernetes-ingress)
- [Azure/aad-pod-identity](https://github.com/Azure/aad-pod-identity)
