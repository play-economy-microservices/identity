# Identity


## Add the GitHub package source

```powershell
$version="1.0.9"
$owner="play-economy-microservices"
$gh_pat="[PAT HERE]"

dotnet pack src/Play.Identity.Contracts/ --configuration Release -p:PackageVersion=$version -p:RepositoryUrl=https://github.com/$owner/play.identity -o ../packages

dotnet nuget push ../packages/Play.Identity.Contracts.$version.nupkg --api-key $gh_pat --source "github"
```

## Build the docker image

```powershell
$env:GH_OWNER="play-economy-microservices"
$env:GH_PAT="[PAT HERE]"
$appname="playeconomycontainerregistry"
docker build --secret id=GH_OWNER --secret id=GH_PAT -t "$appname.azurecr.io/play.identity:$version" .
```

## Run the docker image

```powershell
$adminPass="[PASSWORD HERE]"
$cosmosDbConnString="[CONN STRING HERE]"
$serviceBusConnString="[CONN STRING HERE]"
docker run -it --rm -p 5002:5002 --name identity -e
MongoDbSettings__ConnectionString=$cosmosDbConnString -e
ServiceBusSettings__ConnectionString=$serviceBusConnString -e
ServiceSettings__MessageBroker="SERVICEBUS" -e IdentitySettings__AdminUserPassword=$adminPass play.identity:$version
```

## Publishing Docker Image

```powershell
$appname="playeconomycontainerregistry"
$version="1.0.3"
az acr login --name $appname
docker tag play.identity:$version "$appname.azurecr.io/play.identity:$version"
docker push "playeconomycontainerregistry.azurecr.io/play.identity:$version"
```

## Create the Kubernetes Namespace

```powershell
$namespace="identity"
kubectl create namespace $namespace
```

## Create Kubernetes Secrets

This is only used with K8s secrets. If you're using Key Vault then skip this.

```powershell
kubectl create secret generic identity-secrets
--from-literal=cosmosdb-connectionString=$cosmosDbConnString
--from-literal=servicebus-connectionString=$serviceBusConnString
--from-literal=admin-password=$adminPass -n $namespace
```

## Creating the Kubernetes Pod

```powershell
$namespace="identity"
kubectl apply -f ./kubernetes/identity.yaml -n $namespace
```

## Creating the Azure Managed Identity and granting it access to the Key Vault

```powershell
$namespace="identity"
$appname="playeconomy"

az identity create --resource-group $appname --name $namespace

$IDENTITY_CLIENT_ID=az identity show -g $appname -n $namespace --query clientId -otsv
az keyvault set-policy -n $appname --secret-permissions get list --spn $IDENTITY_CLIENT_ID
```

## Establish the federated identity credential

```powershell PowerShell
$appname="playeconomy"
$namespace="identity"

$AKS_OIDC_ISSUER=az aks show -n $appname -g $appname --query "oidcIssuerProfile.issuerUrl" -otsv

az identity federated-credential create --name $namespace --identity-name $namespace
--resource-group $appname --issuer $AKS_OIDC_ISSUER --subject "system:serviceaccount:${namespace}:${namespace}-serviceaccount"
```

## Create the signing certificate

```powershell
$namespace="identity"

kubectl apply -f ./kubernetes/signing-cer.yaml -n $namespace
```

## Install the Helm Chart

To run an initial manual deployment without the ACR Helm Chart run this:

```powershell
$namespace="identity"

helm install identity-service ./helm -f ./helm/values.yaml -n $namespace
```

To install the ACR Helm Chart with the Identity values run this:

```powershell
$appname="playeconomyacr"
$namespace="identity"

$helmUser=[guid]::Empty.Guid
$helmPassword=az acr login --name $appname --expose-token --output tsv --query accessToken

# This is no longer needed after Helm v3.8.0
$env:HELM_EXPERIMENTAL_OCI=1

# authenticate
helm registry login "$appname.azurecr.io" --username $helmUser --password $helmPassword

# Install the Helm Chart from ACR with Identity Values
$chartVersion="0.1.0"
helm upgrade identity-service oci://$appname.azurecr.io/helm/microservice --version $chartVersion -f ./helm/values.yaml -n $namespace --install
```

## Required repository secrets for GitHub Workflow

- `GH_PAT`
- `AZURE_CLIENT_ID`
- `AZURE_TENANT_ID`
- `AZURE_SUBSCRIPTION_ID`
