# Spring Music on Azure Container Apps

This guide provides instructions on how to deploy the Spring Music application to Azure Container Apps using an in-memory database. Spring Music is a sample application for using database services with the Spring Framework and Spring Boot.

You will learn how to:

- Containerize the Spring Music application using Azure Container Registry Buildpack.
- Run the application on Azure Container Apps with the in-memory database.

## Table of Contents
- [Prerequisites](#prerequisites)
- [Set environment variables](#set-environment-variables)
- [Create resource group](#create-resource-group)
- [Create Azure Container Environment](#create-azure-container-environment)
- [Create Azure Container Registry](#create-azure-container-registry)
- [Containerize image](#containerize-image)
- [Create Azure Container Apps](#create-azure-container-apps)
- [Verification](#verification)
- [Next Steps](#next-steps)

## Prerequisites

Before you begin, ensure you have the following:

- An Azure account.
- [Azure CLI](https://learn.microsoft.com/en-us/cli/azure/) installed and logged in.

## Set environment variables

Set the necessary environment variables for your Azure resources.

```sh
RESOURCE_GROUP=<your-resource-group-name>
CONTAINER_APP_ENV=<your-global-unique-container-app-environment-name>
ACR_NAME=<your-global-unique-container-registry-name>
LOCATION=eastus
CONTAINER_APP_NAME=spring-music
```

## Create resource group

Create a resource group to organize your Azure resources.

```sh
az group create --name $RESOURCE_GROUP --location $LOCATION
```

## Create Azure Container Environment

Create an Azure Container Apps environment.

```sh
az containerapp env create --name $CONTAINER_APP_ENV --resource-group $RESOURCE_GROUP --location $LOCATION
```

## Create Azure Container Registry

Create an Azure Container Registry to store your container images.

```sh
az acr create --resource-group $RESOURCE_GROUP --name $ACR_NAME --sku Basic --location $LOCATION
```

Create a managed identity for your container app to access the container registry. The managed identity will be granted AcrPull access and used by the Azure Container Apps to pull images from the Container Registry.

```sh
az identity create --name ${ACR_NAME}-identity --resource-group $RESOURCE_GROUP

# Assign the AcrPull role to the managed identity, allowing it to pull images from the Azure Container Registry.
IDENTITY_CLIENT_ID=$(az identity show --name ${ACR_NAME}-identity --resource-group $RESOURCE_GROUP --query 'clientId' -o tsv)
ACR_ID=$(az acr show --name $ACR_NAME --resource-group $RESOURCE_GROUP --query 'id' -o tsv)
az role assignment create --assignee $IDENTITY_CLIENT_ID --role AcrPull --scope $ACR_ID
```

## Containerize image

Edit `project.toml` to set Buildpack configurations. The added command specifies the Java version to 17.

```diff
--- a/project.toml
+++ b/project.toml
@@ -14,3 +14,7 @@ exclude = [
 [[build.env]]
 name = "BP_SPRING_CLOUD_BINDINGS_DISABLED"
 value = "true"
+
+[[build.env]]
+name = "BP_JVM_VERSION"
+value = "17.*"
```

Build the container image using Azure Container Registry. The build will happen on a cloud agent and finally store the image in the given Azure Container Registry. See more details in [Build and push an image from an app using a Cloud Native Buildpack](https://learn.microsoft.com/en-us/azure/container-registry/container-registry-tasks-pack-build).

```sh
# Build the container image using Azure Container Registry and the specified buildpack.
az acr pack build -r $ACR_NAME --pull -t spring-music:latest --builder paketobuildpacks/builder-jammy-base .
```

## Create Azure Container Apps

Create the Azure Container App. 
- This command specifies the resource consumption for the application. Check [Containers in Azure Container Apps](https://learn.microsoft.com/en-us/azure/container-apps/containers) for how many resources can be used by the application with the configuration. 
- This application will run with the image built from the previous step. It uses the managed identity that has AcrPull privilege to pull the ACR image. 
- The application will also be exposed to the Internet with `--ingress external` configuration.

```sh
ACR_IDENTITY_ID=$(az identity show --name ${ACR_NAME}-identity --resource-group $RESOURCE_GROUP --query 'id' -o tsv)
az containerapp create \
    --name $CONTAINER_APP_NAME \
    --resource-group $RESOURCE_GROUP \
    --environment $CONTAINER_APP_ENV \
    --target-port 8080 \
    --ingress 'external' \
    --cpu 1 \
    --memory 2Gi \
    --image $ACR_NAME.azurecr.io/spring-music:latest \
    --registry-identity $ACR_IDENTITY_ID \
    --registry-server $ACR_NAME.azurecr.io
```

## Verification

Get the URL of the app and visit it through the browser.

```sh
az containerapp show --name $CONTAINER_APP_NAME --resource-group $RESOURCE_GROUP --query properties.configuration.ingress.fqdn -o tsv
```

## Next Steps

To connect the Spring Music application with persistent storage, follow the guides below:

- [Using MongoDB as the backing service](./MongoDB.md)
- [Using MySQL as the backing service](./MySQL.md)
- [Using PostgreSQL as the backing service](./PostgreSQL.md)
- [Using Redis as the backing service](./redis.md) (Coming soon)

