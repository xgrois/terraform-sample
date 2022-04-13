# Azure DevOps and Terraform

[DevOps Pipeline to build](Capture.JPG)

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

> Note: take care of exe file when using terraform CLI since GitHub will complain about the size. You can add it to `.gitignore`.

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

variable "imagebuild" {
  type = string
  default = ""
  description = "docker image tag"
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
    image = "xgrois/weatherapi:${var.imagebuild}"
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

> Note: you can safely use terraform CLI from your desktop computer since you are already logged in Azure (`az login`)

> Note: however, if you want to include Terraform in Azure DevOps, you will need the credentials in this section,
> so Azure DevOps can use terraform and terraform can deploy to your Azure cloud.

> Note: you can safely use credentials in your DevOps account, but take care of them and not use them anywhere else. Do not add those in code or documentation.
> Otherwise, a bad person (yes, a bad person) might see them in your GitHub repo, and use your Azure cloud account for some self-satisfaction

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

#### 1 Give access to your GitHub repo

Go to Pipelines -> Create Pipeline -> GitHub (give permissions when asked).

Select your repo (accept permissions).

#### 2 Define the 1st action: Create a Docker image and push it to your DockerHub

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

> Note: this will add this YAML file to your GitHub repo. AzureDevOps will automatically run the pipeline. So, if everything is OK, you will see a new docker image in your DockerHub

> Note: every new commit and push to GitHub for this repo will trigger the pipeline. You can see that viewing the tags of the docker image in DockerHub. Now, you start to see the power of pipelines that can automate a lot of things everytime you update your code base.

#### 2 Define the 2nd action: Use Terraform to deploy the container in Azure cloud

Terraform can be used in Azure DevOps to deploy your stuff to Azure cloud.
However, Terraform will need appropriate credentials to do that.
Here is how we configure them.

In Pipelines section, go to Library and add a Variable group:

-   Name: Terraform Service Principal Vars (any name you like)
-   Description: any you want
-   Allow access to all pipelines (yes)
-   Link secrets from an Azure
-   Add a new variable (you can add the padlock to all if you want):
    -   Name: ARM_CLIENT_ID
    -   Value: your mega secret code
-   Add a new variable:
    -   Name: ARM_CLIENT_SECRET
    -   Value: your mega secret code
-   Add a new variable:
    -   Name: ARM_TENANT_ID
    -   Value: your mega secret code
-   Add a new variable:
    -   Name: ARM_SUBSCRIPTION_ID
    -   Value: your mega secret code

Each time Azure DevOps runs the Pipeline, a new context for Terraform is created. This means that the file `terraform.tfstate` will not persist.
Since this file is "the git file for Terraform" to keep tracking terra-code to azure-resources, we need a way to persist the file.
We will do what Terraform suggests to do so, what it is called the "backend".

Go to Azure cloud:

-   Add a new Resource Group
    -   Name: what you want, e.g. "tf_storage_rg"
    -   Location: your typical one
    -   Review and Create

Create a Storage Account (blob)

-   Use the previous rg
-   Use any name you want (must be unique globally...), xgroistfstorageacc
-   Use typical location
-   Use LRS for replication (the most basic one)
-   Use Cool as Access Tier
-   Leave defaults for other

In the new Blob service, go to Containers, and create a new one:

-   Name: anything you like, tfstate
-   Private

Now, in `main.tf` file add next below provider entry:

```
...
terraform {
  backend "azurerm" {
    resource_group_name = "tf_storage_rg"
    storage_account_name = "xgroistfstorageacc"
    container_name = "tfstate"
    key = "terraform.tfstate"
  }
}
...
```

Note how Terraform understands that it will need to take the `terraform.state` file from that azure location and now creating a new one.

Now, let's add Terraform to our DevOps pipeline. Update your `azure-pipelines.yml` file as below:

```
trigger:
    - master

resources:
    - repo: self

variables:
    tag: "$(Build.BuildId)"

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
                      containerRegistry: "xgrois Docker Hub"
                      repository: "xgrois/weatherapi"
                      command: "buildAndPush"
                      Dockerfile: "**/Dockerfile"
                      tags: |
                          $(tag)
    - stage: Provision
      displayName: Deploy to Azure using Terraform
      dependsOn: Build
      jobs:
          - job: Provision
            displayName: Provision Container Instance
            pool:
                vmImage: ubuntu-latest
            variables:
                - group: Terraform Service Principal Vars
            steps:
                - script: |
                      set -e

                      terraform init -input=false
                      terraform apply -input=false -auto-approve
                  name: runTerraform
                  displayName: Run Terraform
                  env:
                      ARM_CLIENT_ID: $(ARM_CLIENT_ID)
                      ARM_CLIENT_SECRET: $(ARM_CLIENT_SECRET)
                      ARM_TENANT_ID: $(ARM_TENANT_ID)
                      ARM_SUBSCRIPTION_ID: $(ARM_SUBSCRIPTION_ID)
                      TF_VAR_imagebuild: $(tag)
```

## Try it from "scratch"

Delete DockerHub repo with the weatherapi image.

Go to Azure cloud and delete (you can do terraform destroy BUT in case you have any issue):

-   Resource Group: tf-test-rg
-   Container Instance: weatherapi

In your project's directory, remove all terraform files except `main.tf` since we are now initiating all again.
Make sure it looks like below:

```
provider "azurerm" {
    features {

    }
}

terraform {
  backend "azurerm" {
    resource_group_name = "tf_storage_rg"
    storage_account_name = "xgroistfstorageacc"
    container_name = "tfstate"
    key = "terraform.tfstate"
  }
}

resource "azurerm_resource_group" "tf_test" {
  name = "tf-test-rg"
  location = "West Europe"
}

variable "imagebuild" {
  type = string
  default = ""
  description = "docker image tag"
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
    image = "xgrois/weatherapi:${var.imagebuild}"
    cpu = "1"
    memory = "1"
    ports {
      port = 80
      protocol = "TCP"
    }
  }
}
```

Make sure the `azure-pipeline.yml` looks like:

```
trigger:
    - master

resources:
    - repo: self

variables:
    tag: "$(Build.BuildId)"

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
                      containerRegistry: "xgrois Docker Hub"
                      repository: "xgrois/weatherapi"
                      command: "buildAndPush"
                      Dockerfile: "**/Dockerfile"
                      tags: |
                          $(tag)
    - stage: Provision
      displayName: Deploy to Azure using Terraform
      dependsOn: Build
      jobs:
          - job: Provision
            displayName: Provision Container Instance
            pool:
                vmImage: ubuntu-latest
            variables:
                - group: Terraform Service Principal Vars
            steps:
                - script: |
                      set -e

                      terraform init -input=false
                      terraform apply -input=false -auto-approve
                  name: runTerraform
                  displayName: Run Terraform
                  env:
                      ARM_CLIENT_ID: $(ARM_CLIENT_ID)
                      ARM_CLIENT_SECRET: $(ARM_CLIENT_SECRET)
                      ARM_TENANT_ID: $(ARM_TENANT_ID)
                      ARM_SUBSCRIPTION_ID: $(ARM_SUBSCRIPTION_ID)
                      TF_VAR_imagebuild: $(tag)
```

Now, commit and push to Git/GitHub and the Azure DevOps Pipeline will do all the magic.

> Note: the first time you will see an error when initiating the Terraform stage. Azure DevOps will ask you for giving Terra access to the secret variables group (with Azure credentials) to deploy on Azure cloud. Give it permissions and the pipeline process will continue.

Once pipeline has succeeded, you can query the public API endpoint for weatherforecast and should be operative.
Note that any code changes in your API will trigger AzureDevOps pipeline again, so in few minutes your real API will be redeployed and updated!

## Destroy

To destroy Azure resources, in your project's terminal:

```
terraform destroy
```

> Note: if you commit new code to GitHub, the pipeline will create the cloud resources again.
