# Large File Download

Example architecture to download very large files. Option to collect many files into one file.

This project demonstrates how to download large files using several Azure technologies:

- Azure Functions
- Azure Containers
- Azure Storage

Business Use Case:

![Architecture Overview](docs/BusinessOverview.png "Business Overview")

- Users add or remove files to the cart
- Selecting `Download All` will package selected files into one download

Locacal Architecture:

![Architecture Overview](docs/LogicalArchitecture.png "Logical Architecture")

1. The website sends a list of filenames to an Azure Function.
2. Azure Functions retrieves the files and bundles them into a new package.
3. Azure Functions saves the package in blob storage.
4. Azure Functions returns a link to the newly uploaded package file.

# Setup

This setup will deploy the core infrastructure needed to run the the solution:

```bash
# Global
export RG_NAME=large_files
export RG_REGION=westus
export STORAGE_ACCOUNT_NAME=largefilessa

# Container Instance
export ACR_REGISTRY_NAME=packagefiles
export ACR_USER=sp_largefileuser
export CONTAINER_IMAGE_NAME=packagefiles

# Function App
export FX_PLAN_NAME=largefilesplan
export FX_NAME=largefiles
export FX_RUNTIME=python

```

### Resource Group

Create a resource group for this project

```bash
az group create --name $RG_NAME --location $RG_REGION
```

### Storage Account

Create a storage account.

```bash
az storage account create -n $STORAGE_ACCOUNT_NAME -g $RG_NAME -l $RG_REGION --sku Standard_LRS
```

### Azure Container Registry

```bash
az acr create --resource-group $RG_NAME --name $CONTAINER_NAME --sku Basic
```

### Build and Upload Docker Image

```bash
#Build image
 docker build --pull --rm -f "dockerfile" -t $CONTAINER_IMAGE_NAME:latest "."

#Tag
docker tag $CONTAINER_IMAGE_NAME:latest $ACR_REGISTRY_NAME.azurecr.io/$CONTAINER_IMAGE_NAME:latest

# Push to Registry
az acr login --name $ACR_REGISTRY_NAME
docker push $ACR_REGISTRY_NAME.azurecr.io/$CONTAINER_IMAGE_NAME:latest
```

### Service Principal

```bash
# Create identity
az ad sp create-for-rbac --name $ACR_USER --skip-assignment --sdk-auth > local-sp.json

# Create the role assignment
ACR_REGISTRY_ID=$(az acr show --name $ACR_REGISTRY_NAME  --query id --output tsv)
ACR_USER_ID=$(cat local-sp.json | grep clientId | awk -F\" '{print $4}')

az role assignment create --assignee $ACR_USER_ID  --scope $ACR_REGISTRY_ID --role acrpull

```


### Function App

Create a function app using a Private ACR image.

```bash
ACR_PASSWORD=$(cat local-sp.json | grep clientSecret | awk -F\" '{print $4}')

az functionapp plan create -g $RG_NAME -n $FX_PLAN_NAME --min-instances 1 --max-burst 3 --sku EP1

az functionapp create -n $FX_NAME -g $RG_NAME -s $STORAGE_ACCOUNT_NAME --plan $FX_PLAN_NAME  --deployment-container-image-name $ACR_REGISTRY_NAME.azurecr.io/$CONTAINER_IMAGE_NAME:latest --docker-registry-server-password $ACR_PASSWORD --docker-registry-server-user $ACR_USER --functions-version 3 --os-type=Linux
```


# References

- Dev Setup - Azure CLI https://docs.microsoft.com/en-us/cli/azure/install-azure-cli-linux?pivots=apt
- Dev Setup - VS Code on Windows WSL - https://docs.microsoft.com/en-us/windows/wsl/tutorials/wsl-vscode
- Azure Function App CLI docs https://docs.microsoft.com/en-us/cli/azure/functionapp?view=azure-cli-latest
- Python Azure SDK Storage https://docs.microsoft.com/en-us/azure/developer/python/azure-sdk-example-storage-use?tabs=cmd
- Azure Function Deployments https://docs.microsoft.com/en-us/azure/azure-functions/functions-deployment-technologies
- Azure Functions on a Custom Container https://docs.microsoft.com/bs-latn-ba/Azure/azure-functions/functions-create-function-linux-custom-image?tabs=bash%2Cportal&pivots=programming-language-python
