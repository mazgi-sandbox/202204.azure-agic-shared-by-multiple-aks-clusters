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
az ad sp create-for-rbac\
 --name $servicePrincipalName\
 --role Contributor\
 --scopes /subscriptions/${subsctiptionId}\
 -o json\
 > auth.json
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

Use the deployment-outputs.json created after deployment to get the cluster name and resource group name.

```console
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
kubectl --context $aksGreenClusterName get pods,services,ingresses
kubectl --context $aksBlueClusterName get pods,services,ingresses
```

```console
kubectl create --context $aksGreenClusterName -f https://raw.githubusercontent.com/Azure/aad-pod-identity/master/deploy/infra/deployment.yaml
kubectl create --context $aksBlueClusterName -f https://raw.githubusercontent.com/Azure/aad-pod-identity/master/deploy/infra/deployment.yaml
```

```console
kubectl --context $aksGreenClusterName get pods,services,ingresses
kubectl --context $aksBlueClusterName get pods,services,ingresses
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
jq -r ".applicationGatewayPublicIpAddress.value" deployment-outputs.json
applicationGatewayPublicIpAddress=$(jq -r ".applicationGatewayPublicIpAddress.value" deployment-outputs.json)
curl -H 'Host: green.example.com' -I $applicationGatewayPublicIpAddress
curl -H 'Host: blue.example.com' -I $applicationGatewayPublicIpAddress
```

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
az network application-gateway address-pool list\
 --gateway-name $applicationGatewayName\
 --resource-group $resourceGroupName\
 | jq '.[] | "\(.name), \(.backendAddresses)"'
```

```console
az network application-gateway http-listener list\
 --gateway-name $applicationGatewayName\
 --resource-group $resourceGroupName\
 | jq '.[] | "\(.name), \(.protocol), \(.hostNames)"'
```

```console
curl -H 'Host: green.example.com' -I $applicationGatewayPublicIpAddress
curl -H 'Host: blue.example.com' -I $applicationGatewayPublicIpAddress
```

## Example Outputs

```console
$ az network application-gateway address-pool list\
 --gateway-name $applicationGatewayName\
 --resource-group $resourceGroupName\
 | jq '.[] | "\(.name), \(.backendAddresses)"'
"defaultaddresspool, []"
"pool-default-appblue-80-bp-80, [{\"fqdn\":null,\"ipAddress\":\"10.16.0.10\"}]"
"pool-default-appgreen-80-bp-80, [{\"fqdn\":null,\"ipAddress\":\"10.8.0.28\"}]"
```

```console
$ az network application-gateway http-listener list\
 --gateway-name $applicationGatewayName\
 --resource-group $resourceGroupName\
 | jq '.[] | "\(.name), \(.protocol), \(.hostNames)"'
"fl-4eeee60452bce917a80226985da2dd28, Http, [\"blue.example.com\"]"
"fl-93092c57d9659f8b9b48580eca6d8292, Http, [\"green.example.com\"]"
```

```console
$ curl -I $applicationGatewayPublicIpAddress
HTTP/1.1 404 Not Found
Server: Microsoft-Azure-Application-Gateway/v2
Date: Wed, 13 Apr 2022 19:10:24 GMT
Content-Type: text/html
Content-Length: 179
Connection: keep-alive
```

```console
$ curl -H 'Host: green.example.com' -I $applicationGatewayPublicIpAddress
HTTP/1.1 200 OK
Date: Wed, 13 Apr 2022 19:10:54 GMT
Content-Type: text/html
Content-Length: 615
Connection: keep-alive
Server: nginx/1.21.6
Last-Modified: Tue, 25 Jan 2022 15:03:52 GMT
ETag: "61f01158-267"
Accept-Ranges: bytes
```

```console
$ curl -H 'Host: blue.example.com' -I $applicationGatewayPublicIpAddress
HTTP/1.1 200 OK
Date: Wed, 13 Apr 2022 19:11:16 GMT
Content-Type: text/html
Content-Length: 45
Connection: keep-alive
Server: Apache/2.4.53 (Unix)
Last-Modified: Mon, 11 Jun 2007 18:53:14 GMT
ETag: "2d-432a5e4a73a80"
Accept-Ranges: bytes
```

## References

- [How to Install an Application Gateway Ingress Controller (AGIC) Using a New Application Gateway](https://docs.microsoft.com/en-us/azure/application-gateway/ingress-controller-install-new)
- [Brownfield Deployment#Multi-cluster / Shared App Gateway](https://azure.github.io/application-gateway-kubernetes-ingress/setup/install-existing/#multi-cluster-shared-app-gateway)
- [Azure/application-gateway-kubernetes-ingress](https://github.com/Azure/application-gateway-kubernetes-ingress)
- [Azure/aad-pod-identity](https://github.com/Azure/aad-pod-identity)
