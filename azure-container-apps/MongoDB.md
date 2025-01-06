# Spring Music on Azure Container Apps

This guide provides instructions on how to deploy the Spring Music application to Azure Container Apps using MongoDB as the backing service. Spring Music is a sample application for using database services with the Spring Framework and Spring Boot.

You will learn how to:

- Containerize the Spring Music application using Azure Container Registry Buildpack.
- Run the application on Azure Container Apps.
- Set up a MongoDB database as the backing service.
- Connect the MongoDB database with the Spring Music application.

## Table of Contents
- [Prerequisites](#prerequisites)
- [Set environment variables](#set-environment-variables)
- [Create backing service Azure Cosmos DB for MongoDB](#create-backing-service-azure-cosmos-db-for-mongodb)
- [Connect backing service with Azure Container Apps](#connect-backing-service-with-azure-container-apps)
- [Update app's active profile](#update-apps-active-profile)
- [Verification](#verification)

## Prerequisites

Before you begin, ensure you follow [README](./README.md) to deploy the application with an in-memory database to Azure Container Apps.

## Set environment variables

Set the necessary environment variables for your Azure resources.

```sh
RESOURCE_GROUP=<your-resource-group-name>
CONTAINER_APP_ENV=<your-global-unique-container-app-environment-name>
ACR_NAME=<your-global-unique-container-registry-name>
LOCATION=eastus
CONTAINER_APP_NAME=spring-music
COSMOS_DB_ACCOUNT_NAME=<your-cosmos-db-account-name>
COSMOS_DB_DATABASE_NAME=spring-music
```

## Create backing service Azure Cosmos DB for MongoDB

Create an Azure Cosmos DB account for MongoDB and a database.

```sh
az cosmosdb create --name $COSMOS_DB_ACCOUNT_NAME --resource-group $RESOURCE_GROUP --kind MongoDB
az cosmosdb mongodb database create --account-name $COSMOS_DB_ACCOUNT_NAME --name $COSMOS_DB_DATABASE_NAME --resource-group $RESOURCE_GROUP
```

## Connect backing service with Azure Container Apps

This command will create a user in the database and inject necessary database information as environment variables to the Azure Container App.

```sh
CONTAINER_APP_ID=$(az containerapp show --name $CONTAINER_APP_NAME --resource-group $RESOURCE_GROUP --query 'id' -o tsv)
COSMOS_DB_DATABASE_ID=$(az cosmosdb mongodb database show --account-name $COSMOS_DB_ACCOUNT_NAME --name $COSMOS_DB_DATABASE_NAME --resource-group $RESOURCE_GROUP --query 'id' -o tsv)

az containerapp connection create cosmos-mongo --connection cosmosdb_connection \
    --source-id $CONTAINER_APP_ID \
    --target-id $COSMOS_DB_DATABASE_ID \
    --client-type springBoot \
    -c $CONTAINER_APP_NAME \
    --secret
```

## Update app's active profile

Create the Azure Container App. 
- This application will run on active profile `mongodb` with the setting environment variables.

```sh
az containerapp update \
    --name $CONTAINER_APP_NAME \
    --resource-group $RESOURCE_GROUP \
    --set-env-vars SPRING_PROFILES_ACTIVE=mongodb
```

## Verification

Get the URL of the app and visit it through the browser.

```sh
az containerapp show --name $CONTAINER_APP_NAME --resource-group $RESOURCE_GROUP --query properties.configuration.ingress.fqdn -o tsv
```
