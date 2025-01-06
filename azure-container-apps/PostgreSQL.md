# Spring Music on Azure Container Apps

This guide provides instructions on how to deploy the Spring Music application to Azure Container Apps using PostgreSQL as the backing service. Spring Music is a sample application for using database services with the Spring Framework and Spring Boot.

You will learn how to:

- Containerize the Spring Music application using Azure Container Registry Buildpack.
- Run the application on Azure Container Apps.
- Set up a PostgreSQL flexible server as the backing service.
- Connect the PostgreSQL database with the Spring Music application.

## Table of Contents
- [Prerequisites](#prerequisites)
- [Set environment variables](#set-environment-variables)
- [Create backing service Azure PostgreSQL flexible server](#create-backing-service-azure-postgresql-flexible-server)
- [Connect backing service with Azure Container Apps](#connect-backing-service-with-azure-container-apps)
- [Update code with Azure dependencies](#update-code-with-azure-dependencies)
- [Containerize image](#containerize-image)
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
POSTGRES_SERVER_NAME=<your-postgres-server-name>
POSTGRES_DATABASE_NAME=spring-music
```

## Create backing service Azure PostgreSQL flexible server

Create a PostgreSQL flexible server and database.

```sh
az postgres flexible-server create --resource-group $RESOURCE_GROUP --name $POSTGRES_SERVER_NAME
az postgres flexible-server db create --database-name $POSTGRES_DATABASE_NAME --server $POSTGRES_SERVER_NAME --resource-group $RESOURCE_GROUP
```

## Connect backing service with Azure Container Apps

Create a managed identity for the PostgreSQL server that will be used to connect with the container app.

```sh
az identity create --name ${POSTGRES_SERVER_NAME}-rw-identity --resource-group $RESOURCE_GROUP
```

This command will create a user in the database and inject necessary database information as environment variables to the Azure Container App.

```sh
IDENTITY_CLIENT_ID=$(az identity show --name ${POSTGRES_SERVER_NAME}-rw-identity --resource-group $RESOURCE_GROUP --query 'clientId' -o tsv)
# POSTGRES_IDENTITY_ID=$(az identity show --name ${POSTGRES_SERVER_NAME}-rw-identity --resource-group $RESOURCE_GROUP --query 'id' -o tsv)
CONTAINER_APP_ID=$(az containerapp show --name $CONTAINER_APP_NAME --resource-group $RESOURCE_GROUP --query 'id' -o tsv)
POSTGRES_DATABASE_ID=$(az postgres flexible-server db show --database-name $POSTGRES_DATABASE_NAME --server $POSTGRES_SERVER_NAME --resource-group $RESOURCE_GROUP --query 'id' -o tsv)
SUBSCRIPTION_ID=$(az account show --query 'id' -o tsv)

az containerapp connection create postgres-flexible --connection postgres_connection \
    --source-id $CONTAINER_APP_ID \
    --target-id $POSTGRES_DATABASE_ID \
    --client-type springBoot \
    --user-identity \
        client-id=$IDENTITY_CLIENT_ID \
        subs-id=$SUBSCRIPTION_ID \
    -c $CONTAINER_APP_NAME
```

## Update code with Azure dependencies

Update your `build.gradle` file to include Azure dependencies. The `com.azure.spring:spring-cloud-azure-starter-jdbc-postgresql` dependencies will use Microsoft Entra authentication and PostgreSQL authentication to auth the application to Azure PostgreSQL server. See more details in [Use Spring Data JDBC with Azure Database for PostgreSQL](https://learn.microsoft.com/en-us/azure/developer/java/spring-framework/configure-spring-data-jdbc-with-azure-postgresql?tabs=passwordless%2Cservice-connector&pivots=postgresql-passwordless-flexible-server).

```diff
--- a/build.gradle
+++ b/build.gradle
@@ -20,6 +20,12 @@ ext {
     javaCfEnvVersion = '3.1.2'
 }

+dependencyManagement {
+    imports {
+        mavenBom "com.azure.spring:spring-cloud-azure-dependencies:5.19.0"
+    }
+}
+
 dependencies {
     // Spring Boot
     implementation "org.springframework.boot:spring-boot-starter-web"
```
```diff
--- a/build.gradle
+++ b/build.gradle
@@ -28,6 +34,7 @@ dependencies {
     implementation "org.springframework.boot:spring-boot-starter-data-mongodb"
     implementation "org.springframework.boot:spring-boot-starter-data-redis"
     implementation "org.springframework.boot:spring-boot-starter-validation"
+    implementation "com.azure.spring:spring-cloud-azure-starter-jdbc-postgresql"

     // Java CfEnv
     implementation "io.pivotal.cfenv:java-cfenv-boot:${javaCfEnvVersion}"
```

## Containerize image

Build the container image using Azure Container Registry.

```sh
# Build the container image using Azure Container Registry and the specified buildpack.
az acr pack build -r $ACR_NAME --pull -t spring-music-postgresql:latest --builder paketobuildpacks/builder-jammy-base .
```

## ## Update app's active profile

Create the Azure Container App. 
- This application will run on active profile `postgresql` with the setting environment variables.

```sh
az containerapp update \
    --name $CONTAINER_APP_NAME \
    --resource-group $RESOURCE_GROUP \
    --image $ACR_NAME.azurecr.io/spring-music-postgresql:latest \
    --set-env-vars SPRING_PROFILES_ACTIVE=postgresql
```

## Verification

Get the URL of the app and visit it through the browser.

```sh
az containerapp show --name $CONTAINER_APP_NAME --resource-group $RESOURCE_GROUP --query properties.configuration.ingress.fqdn -o tsv
```