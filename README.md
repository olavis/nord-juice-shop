# Nord Juice Shop

*Nord Juice Shop* is a collection of scripts that aims to facilitate automation of the deployment of MultiJuicer and CTFd, which may be used to host CTF events or internal training on real-world web application security issues using OWASP JuiceShop.

The main script, [`manage-multijuicer.sh`](./manage-multijuicer.sh), is a wrapper around MultiJuicer and CTFd (see [Acknowledgements](#acknowledgements)), and as such must be deployed on a Kubernetes cluster. The repository also contains a script for creating a new Kubernetes cluster in Azure Kubernetes Service (AKS) (see [Creating a Kubernetes cluster in Azure](#creating-a-kubernetes-cluster-in-azure)). For a full example of going from zero to deployment, see [Full example with deployment on Azure](#full-example-with-deployment-on-azure).

We have been using this solution in our internal workshops and events, with the aim of providing IT personnel with hands-on practical experience on web application security, in a *safe* environment.

## Prerequisites

### Packages & Services
- [A Kubernetes cluster](https://kubernetes.io/) (or [an Azure subscription](#creating-a-kubernetes-cluster-in-azure))
- [Helm](https://helm.sh/docs/intro/install/)
- [kubectl](https://kubernetes.io/docs/tasks/tools/#kubectl)

### Environment variables
```sh
# Copy the example environment variable file .env
cp .env .env.local
```
Modify the values in the file to your liking. Required variables are marked as `(required)`:
- `CTF_KEY`: A key used to generate challenge flags. Should be rotated between CTF-events to ensure unique flags.
- `COOKIE_SECRET`: Secret used for the cookie.
- `CTFD_SECRET_KEY`: Secret key used by the CTFd instance.
- `JUICE_FQDN`: The domain name at which the setup will be reachable. A DNS record must exist and be publicly reachable for TLS certificate acquisition and routing to work.

Optionally specify the max. number of JuiceShop instances that can be created by setting the variabel `MAX_INSTANCES`, or any other variables you wish to modify.

> :exclamation: Make sure to *source* the environment variable file you just created before proceeding!

## Running

### Deploying the CTF services
*Deployment of the CTF services (i.e. MultiJuicer and CTFd) is done using the script [`manage-multijuicer.sh`](./manage-multijuicer.sh)*

#### Prerequisites
- Access to a Kubernetes cluster, and having activated that context in your `kubectl` config

```bash
./manage-multijuicer.sh -h
Usage: ./manage-multijuicer.sh COMMAND

    Commands:
        up      Deploy the MultiJuicer and CTFd services in the Kubernetes cluster
        down    Remove the MultiJuicer and CTFd services from the Kubernetes cluster

```

```bash
# Deploy the services
./manage-multijuicer.sh up
# The MultiJuicer service should now be available at `JUICE_FQDN`,
# while CTFd should be available at `JUICE_FQDN/ctfd`

# Shut down the services
./manage-multijuicer.sh down
```

### Creating a Kubernetes cluster in Azure
*Creation of a Kubernetes cluster in Azure Kubernetes Service (AKS) is done using the script [`manage-azure-deployment.sh`](./manage-azure-deployment.sh). The script may also be used to create a Resource Group and a Key Vault.*

#### Prerequisites

##### Packages & Services
- [Azure CLI](https://learn.microsoft.com/en-us/cli/azure/)

##### Authenticate against Azure
```bash
# Log in to Azure CLI with your account
# A new tab will open in your browser, asking you to authenticate
az login

# Set the subscription in which you wish to deploy the services
az account set -s <subscription_id | subscription_name>
```

##### Environment variables
Modify the environment variables in the file you copied in [Environment variables](#environment-variables). Required variables are marked as `(required)`:
- `AZURE_DNS_NAME`: The hostname used for the NGINX ingress service. Must be unique within a zone.
- `AZURE_SUBSCRIPTION_ID`: The subscription ID of your Azure subscription.
- `AZURE_RESOURCE_GROUP`: A pre-existing resource group. The script may create it for you, if you set `MANAGE_RG=1`.

Specify which services the script should manage (in addition to the cluster):
- `MANAGE_RG`: If not specified, you must make sure that a resource group with the name `AZURE_RESOURCE_GROUP` exists.
- `MANAGE_KEYVAULT`

> :warning: Make sure that the resource group `AZURE_RESOURCE_GROUP` exists prior to running, or set `MANAGE_RG=1`.

> :exclamation: Make sure to *source* the environment variable file you just created before proceeding!

```bash
./manage-azure-deployment.sh -h
Usage: ./manage-azure-deployment.sh COMMAND

    Commands:
        new     Deploy a brand new cluster
        up      Spin the cluster back up, scaling up the resources
        down    Scale down the cluster to save resources (keeps the AKS resource itself intact)
        wipe    Removes the cluster
        wipe-all        Removes the cluster, resource group, and key vault.
        config  Run post-deployment configurations, including creating the DNS record in Azure
        password        Retrieve the admin password for the multi-juicer instance

```

```bash
# Creating a new cluster
./manage-azure-deployment.sh new

# 'Turn on' a cluster previously taken down with the 'down' command (see below).
./manage-azure-deployment.sh up

# Shut down the cluster, but keep the AKS resource intact - intended for when you plan to spin it back up within a short time.
./manage-azure-deployment.sh down

# Shut down the cluster and remove the AKS resource.
./manage-azure-deployment.sh wipe

# Shut down the cluster and remove the AKS resource as well as the resource group and key vault.
./manage-azure-deployment.sh wipe-all

# Run post-deployment configurations. Must be run after running `manage-multijuicer.sh up` to configure the DNS record and more.
./manage-azure-deployment.sh config

# Retrieve the password for the admin-user in the MultiJuicer instance
./manage-azure-deployment.sh password
```

#### Full example with deployment on Azure
You may use the domain name provided by Azure DNS to reach your host. If you wish to do so, make sure to set `JUICE_FQDN` to `<AZURE_DNS_NAME>.<AZURE_LOCATION>.cloudapp.azure.com` (e.g. `juice1337.norwayeast.cloudapp.azure.com`) **prior** to running the scripts.
```bash
# Firstly, create a new Kubernetes cluster in Azure Kubernetes Service
./manage-azure-deployment.sh new

# Next, we will deploy the services in the Kubernetes cluster
./manage-multijuicer.sh up

# Finally, run the post-deployment configuration to finalize the deployment
./manage-azure-deployment.sh config

# Once the event is done, you may remove the cluster and all services by running
./manage-multijuicer.sh down
./manage-azure-deployment.sh wipe
```

### Creating a Service Principal in Azure
*Creation of a Service Principal (App Registration) in Azure Active Directory (AAD) is done using the script [`service-principal.sh`](./service-principal.sh).*

#### Prerequisites

##### Packages & Services
- [Azure CLI](https://learn.microsoft.com/en-us/cli/azure/)

##### Authenticate against Azure
```bash
# Log in to Azure CLI with your account
# A new tab will open in your browser, asking you to authenticate
az login

# Set the subscription in which you wish to deploy the services
az account set -s <subscription_id | subscription_name>
```

##### Environment variables
Modify the environment variables in the file you copied in [Environment variables](#environment-variables). Required variables are marked as `(required)`:
- `AZURE_SUBSCRIPTION_ID`: The subscription ID of your Azure subscription.
- `AZURE_RESOURCE_GROUP`: Name of the resource group to use.
- `AZURE_SERVICE_PRINCIPAL_NAME`: Name of the service principal.
- `AZURE_AD_APP_ADMIN_GROUP`: Name of the Azure AD group allowed to administrate the Service Principal.

Optionally specify the Github repository in which the code exists, to allow the Service Principal access to the repository:
- `GIT_REPO`: E.g. `bouvet/nord-juice-shop`

> :warning: Make sure that the resource group `AZURE_RESOURCE_GROUP` exists prior to running.

> :exclamation: Make sure to *source* the environment variable file you just created before proceeding!

```bash
./service-principal.sh
Usage: ./service-principal.sh COMMAND

    Commands:
        new     Create a new service principal
        wipe    Delete the service principal

```

## Acknowledgements
This project provides scripts for automating as much as possible in terms of deploying the [`MultiJuicer`](https://github.com/juice-shop/multi-juicer/) and [`CTFd`](https://ctfd.io/) services in a Kubernetes cluster.

Please see the respective websites for more information:
- CTFd:
    - https://ctfd.io/
    - https://github.com/CTFd/CTFd
- OWASP JuiceShop:
    - https://owasp.org/www-project-juice-shop/
    - https://github.com/juice-shop/juice-shop
- MultiJuicer:
    - https://github.com/juice-shop/multi-juicer
