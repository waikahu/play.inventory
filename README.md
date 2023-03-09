# Play.Inventory
Play Economy Inventory microservice

## Create and publish package
```powershell
$version="1.0.4"
$owner="waikahu"
$gh_pat="[PAT HERE]"

dotnet pack src\Play.Inventory.Contracts\ --configuration Release -p:PackageVersion=$version -p:RepositoryUrl=https://github.com/$owner/play.inventory -o ..\packages

dotnet nuget push ..\packages\Play.Inventory.Contracts.$version.nupkg --api-key $gh_pat --source "github" 
```

## Build the Docker image
```powershell
$env:GH_OWNER="waikahu"
$env:GH_PAT="[PAT HERE]"
$appname="wbplayeconomy"
docker build --secret id=GH_OWNER --secret id=GH_PAT -t "$appname.azurecr.io/play.inventory:$version" .
```

## Run the docker image
```powershell
$cosmosDbConnString="[Conn HERE]"
$serviceBusConnString="[Conn HERE]"
docker run -it --rm -p 5004:5004 --name inventory -e MongoDbSettings__ConnectionString=$cosmosDbConnString -e ServiceBusSettings__ConnectionString=$serviceBusConnString -e ServiceSettings__MessageBroker="SERVICEBUS" play.inventory:$version
```

## Publishing the Docker image
```powershell
az acr login --name $appname
docker push "$appname.azurecr.io/play.inventory:$version"
```

## Create the Kubernetes namespace
```powershell
$namespace="inventory"
kubectl create namespace $namespace
```

## Create the Kubernetes pod
```powershell
kubectl apply -f .\kubernetes\inventory.yaml -n $namespace

# to see list of pods
kubectl get pods -n $namespace -w
# to see list of services
kubectl get services -n $namespace
# to see the logs of pod
kubectl logs <name of pod> -n $namespace
# to see datailed pod
kubectl describe pod <name of pod> -n $namespace

kubectl rollout restart deployment inventory-deployment -n inventory
```

## Creating the Azure Managed Identity and granting it access to the Key Vault secrets
```powershell
$appname="wbplayeconomy"
$namespace="inventory"
az identity create --resource-group $appname --name $namespace
$IDENTITY_CLIENT_ID=az identity show -g $appname -n $namespace --query clientId -otsv
# i've to put the appid manually in the Azure Key Vault # 
az keyvault set-policy -n $appname --secret-permissions get list --spn $IDENTITY_CLIENT_ID
```

## Establish the federated identity credential
```powershell
$AKS_OIDC_ISSUER=az aks show -n $appname -g $appname --query "oidcIssuerProfile.issuerUrl" -otsv

az identity federated-credential create --name $namespace --identity-name $namespace --resource-group $appname --issuer $AKS_OIDC_ISSUER --subject "system:serviceaccount:${namespace}:${namespace}-serviceaccount"
```

## Install the Helm Chart
```powershell
$namespace="inventory"
$appname="wbplayeconomy"

$helmUser=[guid]::Empty.Guid
$helmPassword=az acr login --name $appname --expose-token --output tsv --query accessToken

helm registry login "$appname.azurecr.io" --username $helmUser --password $helmPassword

$chartVersion="0.1.0"
helm install inventory-service oci://$appname.azurecr.io/helm/microservice --version $chartVersion -f .\helm\values.yaml -n $namespace
#or Upgrade
helm upgrade inventory-service oci://$appname.azurecr.io/helm/microservice --version $chartVersion -f .\helm\values.yaml -n $namespace

helm repo update

#To see if the services are running
kubectl get pods -n $namespace
kubectl get services -n $namespace
kubectl get serviceaccount -n $namespace
kubectl get deployment -n $namespace
```