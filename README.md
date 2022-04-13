# Azure DevOps and Terraform

You need:

-   .NET
-   git
-   GiHub account
-   Docker Desktop (windows)
-   DockerHub account
-   Azure CLI
-   Terraform CLI (and Azure Terraform VS Code Microsoft extension, HashiCorp Terraform VS Code extension)

## ASP .NET 6 time

Create a folder `terraform-sample`.

Open windows terminal inside the new folder and run `dotnet new webapi -minimal -n weatherapi -o .`

Open VS Code and accept to add the assets for build and debug (will add folder `.vscode`).

In `Program.cs` comment `app.UseHttpsRedirection();`

> Note: feel free to try it. This is a crash intro to Azure DevOps and Terraform and we will avoid likely conflicts.

Run the app `dotnet run` to check every works nicely.

## Docker and DockerHub time

Add a `Dockerfile`to your project with the content below:

```
# https://hub.docker.com/_/microsoft-dotnet
FROM mcr.microsoft.com/dotnet/sdk:6.0 AS build
WORKDIR /app

# copy csproj and restore as distinct layers
COPY *.csproj .
RUN dotnet restore

# copy everything else and build app
COPY . .
RUN dotnet publish -c release -o out

# final stage/image
FROM mcr.microsoft.com/dotnet/aspnet:6.0
WORKDIR /app
EXPOSE 80
COPY --from=build /app/out .
ENTRYPOINT ["dotnet", "weatherapi.dll"]
```

Create docker image with `docker build -t xgrois/weatherapi .`

Run a docker container using that image to test everything behaves as expected. Use `docker run --rm -p 8080:80 xgrois/weatherapi`

Push docker image to DockerHub `docker push xgrois/weatherapi`

## Git and GitHub time

Add `.gitignore` file with the Visual Studio template.

Now, create the local repo and then the remote in GitHub.

## Terraform time

Create a new file `main.tf` with below content:

```
provider "azurerm" {
    features {

    }
}

resource "azurerm_resource_group" "tf_test" {
  name = "tf-test-rg"
  location = "West Europe"
}
```

Now run in the terminal `terraform init`, then `terraform plan` and then `terraform apply`.

> Note: you need to be login with Azure CLI (`az login`).

You should see in Azure Portal that a new resource group (_tf-test-rg_) has been created!

Now add container instance in Azure:

```
...
resource "azurerm_resource_group" "tf_test" {
  name = "tf-test-rg"
  location = "West Europe"
}

resource "azurerm_container_group" "tfcg_test" {
  name = "weatherapi"
  location = azurerm_resource_group.tf_test.location
  resource_group_name = azurerm_resource_group.tf_test.name

  ip_address_type = "Public"
  dns_name_label = "xgroisweatherapi"
  os_type = "Linux"

  container {
    name = "weatherapi"
    image = "xgrois/weatherapi"
    cpu = "1"
    memory = "1"
    ports {
      port = 80
      protocol = "TCP"
    }
  }
}
```

and then `terraform plan` and then `terraform apply`.

You will see the new weatherapi container instance in Azure portal.

You can try the API in your browser:

`http://xgroisweatherapi.westeurope.azurecontainer.io/weatherforecast`

> Note: HTTPS will not work.

## Time to automate all previous steps

We need the next Service Principal Environment Variables (**DO NOT SHARE THESE NEVER**):

-   ARM_CLIENT_ID
-   ARM_CLIENT_SECRET
-   ARM_TENANT_ID
-   ARM_SUBSCRIPTION_ID

Go to Azure Active Directory -> Register and application -> New

-   Name: Terraform
-   Single tenant
    Register

Copy in a secure place:

-   Application (client) ID
-   Directory (tenant) ID

Now go to Certificates & Secrets

-   Create a new client secret (you can type any name you want, e.g. Terraform Client Secrets)
-   Copy in a safe place the generated Value field

Go Azure Home and copy your Subscription ID, also in a safe place.

Go to your subscription and then IAM section -> Add Role Assignement and select:

-   Contributor
-   User, group, or service principal
-   Select members, your app registration (e.g. "Terraform")

This will add Terraform as a Contributor for your Azure Subscription.

## Azure DevOps time

Go to your Azure DevOps account.

Add a new project.

Go to Project Settings -> Service connections -> Create service connection and select Docker Registry, and then Docker Hub.
Set your ID and Pass, verify, and add some Service connection name (e.g., xgrois Docker Hub).
Mark "Grant access permission to all pipelines" and Save.

> Note: now Azure DevOps has full access to your DockerHub account.

Add a new service connection. Azure Service Manager -> Service pricipal -> Subscription with Service connection name e.g., xgrois Azure Resource Manager.
Mark "grant access permission to all pipelines" and Save.

> Note: now Azure DevOps has full access to deploy on your Azure Cloud

### Pipelines

> Note: Pipelines are equivalent to GitHub Actions

Pipelines will (every time you push a change in you GitHub repo):

-   Take the repo in GitHub you want
-   Perform any action (or sequential actions) with it. e.g. in our case:
    -   Create a Docker image and publish it to DockerHub
    -   Pick that image and perform a deployment to Azure cloud following Terraform instructions

Go to Pipelines -> Create Pipeline -> GitHub (give permissions when asked).

Select your repo (accept permissions).

Select Docker -> Validate and configure. In the YAML, click "Show assistant" and type docker, select Docker. In options, set your DockerHub as container registry,
type your docker repo name `xgrois/weatherapi` and leave other defaults. Add.
Change existing task to the new. This is how it should look like:

```
# Build a Docker image
# https://docs.microsoft.com/azure/devops/pipelines/languages/docker

trigger:
- master

resources:
- repo: self

variables:
  tag: '$(Build.BuildId)'

stages:
- stage: Build
  displayName: Build image
  jobs:
  - job: Build
    displayName: Build
    pool:
      vmImage: ubuntu-latest
    steps:
    # Docker
    - task: Docker@2
      inputs:
        containerRegistry: 'xgrois Docker Hub'
        repository: 'xgrois/weatherapi'
        command: 'buildAndPush'
        Dockerfile: '**/Dockerfile'
        tags: |
          $(tag)
```

Save and run.
