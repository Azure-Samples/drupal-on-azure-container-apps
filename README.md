# Deploy Drupal on Azure Container Apps

Drupal is an open-source content management framework written in the PHP server-side scripting language. Drupal provides a backend framework for many enterprise websites, with standard features such as content authoring, reliable performance, and robust security.

The two code sets used by every Drupal site: the codebase and the database. The codebase is the set of files that make up Drupal itself, along with any themes, modules, or libraries you've added. The database is where Drupal stores most of its configuration and all of its content. These infrastructure requirements are satisfied by the follwoing Azure services

- [Azure Container Apps (ACA)](https://learn.microsoft.com/en-us/azure/container-apps/overview) for hosting Drupal application
- [Azure Database for MariaDB](https://learn.microsoft.com/en-us/azure/mariadb/overview) for hosting Drupal database
- [Azure Files](https://learn.microsoft.com/en-us/azure/storage/files/storage-files-introduction) for hosting Drupal files

![Solution Architecture](/solution-architecture.png)

## Setup

Install the latest version of the [Azure CLI](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli?view=azure-cli-latest).

- Sign in to the Azure CLI.

```bash
az login
```

* Set up environment variables used in various commands to follow (change according to your preference).

```bash
export RESOURCE_GROUP="my-drupal-apps-group"
export ENVIRONMENT_NAME="my-drupal-storage-environment"
export LOCATION="eastus"
```

* Ensure you have the latest version of the Container Apps Azure CLI extension.

```bash
az extension add -n containerapp --upgrade
```

* Register the Microsoft.App namespace.

```bash
az provider register -n Microsoft.App
```

* Register the Microsoft.OperationalInsights provider for the Azure Monitor Log Analytics workspace if you haven't used it before.

```bash
az provider register -n Microsoft.OperationalInsights
```

* Create a resource group.

```bash
az group create \
  --name $RESOURCE_GROUP \
  --location $LOCATION \
  --query "properties.provisioningState"
```

## Create ACA Environment

* Create ACA environment.

```bash
az containerapp env create \
  --name $ENVIRONMENT_NAME \
  --resource-group $RESOURCE_GROUP \
  --location "$LOCATION" \
  --query "properties.provisioningState"
```

```bash
export ENVIRONMENT_ID=$(az containerapp env show \
  --name $ENVIRONMENT_NAME \
  --resource-group $RESOURCE_GROUP \
  --query "id" \
  --output tsv)

echo $ENVIRONMENT_ID
```

## Set Up Azure Storage Account

* Define a storage account name (must be globally unique).

```bash
RAND=$(head /dev/urandom | tr -dc a-z0-9 | head -c10 ; echo '')
STORAGE_ACCOUNT_NAME="druplaca$RAND"

echo "Storage account name is" $STORAGE_ACCOUNT_NAME
```

* Create an Azure Storage account.

```bash
az storage account create \
  --resource-group $RESOURCE_GROUP \
  --name $STORAGE_ACCOUNT_NAME \
  --location "$LOCATION" \
  --kind StorageV2 \
  --sku Standard_LRS \
  --enable-large-file-share \
  --query provisioningState
```

* Define a file share name.

```bash
FILE_SHARE_NAME="mydrupalfileshare"
```

* Create a file share.

```bash
az storage share create --name $FILE_SHARE_NAME \
    --account-name $STORAGE_ACCOUNT_NAME --only-show-errors --output table
```

* Get the storage account key.

```bash
STORAGE_ACCOUNT_KEY=$(az storage account keys list \
  --resource-group $RESOURCE_GROUP \
  --account-name $STORAGE_ACCOUNT_NAME \
  --query "[0].value" \
  --output tsv)

echo "Storage Account Key is $STORAGE_ACCOUNT_KEY"
```

* Define the storage mount name.

```bash
export STORAGE_MOUNT_NAME="mydrupalstoragemount"
```

* Create a storage mount.

```bash
az containerapp env storage set \
  --access-mode ReadWrite \
  --azure-file-account-name $STORAGE_ACCOUNT_NAME \
  --azure-file-account-key $STORAGE_ACCOUNT_KEY \
  --azure-file-share-name $FILE_SHARE_NAME \
  --storage-name $STORAGE_MOUNT_NAME \
  --name $ENVIRONMENT_NAME \
  --resource-group $RESOURCE_GROUP \
  --output table
```

## Create Azure Database for MariaDB

* Drupal DB Settings

```bash
export DB_SERVER_NAME=drupal-db-srv-$RAND  # Must be globally unique, ie: 'drupal-db-srv-<unique>'
export DRUPAL_DB_HOST=$DB_SERVER_NAME.mariadb.database.azure.com
export DB_SERVER_SKU=GP_Gen5_2 # Azure Database for MariaDB SKU
export DRUPAL_DB_USER=myAdmin  # Cannot be 'admin'.
export DRUPAL_DB_PASSWORD=Zx3$RAND # Must include uppercase, lowercase, and numeric
export DRUPAL_DB_NAME=drupal_db
```

* Create a MariaDB Server

```bash
az mariadb server create --name $DB_SERVER_NAME \
    --location $LOCATION --resource-group $RESOURCE_GROUP \
    --sku-name $DB_SERVER_SKU --ssl-enforcement Disabled \
    --version 10.3 --admin-user $DRUPAL_DB_USER \
    --admin-password $DRUPAL_DB_PASSWORD --output none
```

* Enable Azure services (ie: Web App) to connect to the server.

```bash
az mariadb server firewall-rule create --name AllowAllWindowsAzureIps \
    --resource-group $RESOURCE_GROUP --server-name $DB_SERVER_NAME \
    --start-ip-address 0.0.0.0 --end-ip-address 0.0.0.0 --output none
```

* Create a blank DB for Drupal (Drupal's initialization process expects it to already exist and be empty.)

```bash
az mariadb db create --name $DRUPAL_DB_NAME --server-name $DB_SERVER_NAME \
    --resource-group $RESOURCE_GROUP --output none
```

## Create Azure Container App

* Define Azure Container App name.

```bash
export CONTAINER_APP_NAME="my-drupal-app"
```

* We will use a yaml configuration file [drupal-aca.yaml](/drupal-aca.yaml) to define container app. This file has placeholders for following fields that will be set by executing [update-yaml.sh](/update-yaml.sh) script file.

| Key | Placeholder | Description |
|-|-|-|
| `environmentId` | `ENVIRONMENT_ID` | The environment ID from the previous step. |
| `storageMountName` | `STORAGE_MOUNT_NAME` | The storage mount name from the previous step. |
| `storageMountPath` | `STORAGE_MOUNT_PATH` | The path to the storage mount. This is the path that Drupal will use to store files. |
| `drupalDbHost` | `DRUPAL_DB_HOST` | The MariaDB server host name from the previous step. |
| `drupalDbUser` | `DRUPAL_DB_USER` | The MariaDB server user name from the previous step. |
| `drupalDbPassword` | `DRUPAL_DB_PASSWORD` | The MariaDB server password from the previous step. |
| `drupalDbName` | `DRUPAL_DB_NAME` | The MariaDB server database name from the previous step. |
| `minReplicas` | `MIN_REPLICAS` | The minimum number of replicas for the container app. |
| `maxReplicas` | `MAX_REPLICAS` | The maximum number of replicas for the container app. |

* ACA allows you to create secrets that can be used in the yaml configuration file. We will create secrets for the following environment variables that are used in the yaml template file.
    * `DRUPAL_DB_HOST`
    * `DRUPAL_DB_USER`
    * `DRUPAL_DB_PASSWORD`
    * `DRUPAL_DB_NAME`

* ACA allows you to expose your container app to the public web, via an [External Ingress](https://learn.microsoft.com/en-us/azure/container-apps/ingress-overview).  When you enable ingress, you don't need to create an Azure Load Balancer, public IP address, or any other Azure resources to enable incoming HTTP requests or TCP traffic.

```yaml
ingress:
  external: true
```

* Execute the bash script to create a new yaml file `drupal-aca-DoNotCheckIn.yaml` with the updated values. Make sure not to check in this file to source control.

```bash
chmod +x update-yaml.sh
./update-yaml.sh
```

* Now we are ready to create a Container App to host Drupal.

```bash
az containerapp create -n $CONTAINER_APP_NAME -g $RESOURCE_GROUP \
    --environment $ENVIRONMENT_NAME \
    --yaml drupal-aca-DoNotCheckIn.yaml \
    --query properties.configuration.ingress.fqdn
```

## Access Drupal Site

The last command will return the FQDN of the container app. You can access the Drupal site by navigating to the FQDN in a browser. 

![Drupal Welcome Page](/drupal-welcome-page.png)

Click on the `Log in` link in the top right corner of the page. The default username is `user` and the password is `bitnami`.

![Drupal Logged In Page](/drupal-logged-in-page.png)

## Clean Up Resources

```bash
az group delete --name $RESOURCE_GROUP --yes --no-wait
