# Continuous Delivery with Azure

GitHub Actions is cloud agnostic, so any public cloud provider will work. 

A web app is how we'll be deploying our application to Azure.
A resource group is a collection of resources, like web apps and virtual machines (VMs).
An App Service plan is what runs our web app and manages the billing (our app should run for free).

Through the power of GitHub Actions, we can create, configure, and destroy these resources through our workflow files

**Set up** Azure resources will run if the pull request contains a label with the name "spin up environment".
**Destroy** Azure resources will run if the pull request contains a label with the name "destroy environment".

Setting up Azure resources

The first job sets up the Azure resources as follows:

Logs into your Azure account with the azure/login action. The AZURE_CREDENTIALS secret you created earlier is used for authentication.
Creates an Azure resource group by running az group create on the Azure CLI, which is pre-installed on the GitHub-hosted runner.
Creates an App Service plan by running az appservice plan create on the Azure CLI.
Creates a web app by running az webapp create on the Azure CLI.
Configures the newly created web app to use GitHub Packages by using az webapp config on the Azure CLI. Azure can be configured to use its own Azure Container Registry, DockerHub, or a custom (private) registry. In this case, we'll configure GitHub Packages as a custom registry.

Destroying Azure resources
The second job destroys Azure resources so that you do not use your free minutes or incur billing. The job works as follows:

Logs into your Azure account with the azure/login action. The AZURE_CREDENTIALS secret you created earlier is used for authentication.
Deletes the resource group we created earlier using az group delete on the Azure CLI.

```
  destroy-azure-resources:
    runs-on: ubuntu-latest

    if: contains(github.event.pull_request.labels.*.name, 'destroy environment')
```

```
  jobs:
  setup-up-azure-resources:
    runs-on: ubuntu-latest

    if: contains(github.event.pull_request.labels.*.name, 'spin up environment')
```

```
name: Production deployment

on: 
  push:
    branches:
      - main

env:
  DOCKER_IMAGE_NAME: ulankford-azure-ttt # Must not exist as a package associated with a different repo!
  IMAGE_REGISTRY_URL: docker.pkg.github.com
  #################################################
  ### USER PROVIDED VALUES ARE REQUIRED BELOW   ###
  #################################################
  #################################################
  ### REPLACE USERNAME WITH GH USERNAME         ###
  AZURE_WEBAPP_NAME: ulankford-ttt-app
  #################################################

jobs:
  build:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v1
      - name: npm install and build webpack
        run: |
          npm install
          npm run build
      - uses: actions/upload-artifact@main
        with:
          name: webpack artifacts
          path: public/

  Build-Docker-Image:
    runs-on: ubuntu-latest
    needs: build
    name: Build image and store in GitHub Packages
    steps:
      - name: Checkout
        uses: actions/checkout@v1

      - name: Download built artifact
        uses: actions/download-artifact@main
        with:
          name: webpack artifacts
          path: public

      - name: create image and store in Packages
        uses: mattdavis0351/actions/docker-gpr@1.3.0
        with:
          repo-token: ${{secrets.GITHUB_TOKEN}}
          image-name: ${{env.DOCKER_IMAGE_NAME}}

  Deploy-to-Azure:
    runs-on: ubuntu-latest
    needs: Build-Docker-Image
    name: Deploy app container to Azure
    steps:
      - name: "Login via Azure CLI"
        uses: azure/login@v1
        with:
          creds: ${{ secrets.AZURE_CREDENTIALS }}

      - uses: azure/docker-login@v1
        with:
          login-server: ${{env.IMAGE_REGISTRY_URL}}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Deploy web app container
        uses: azure/webapps-deploy@v2
        with:
          app-name: ${{env.AZURE_WEBAPP_NAME}}
          images: ${{env.IMAGE_REGISTRY_URL}}/${{ github.repository }}/${{env.DOCKER_IMAGE_NAME}}:${{ github.sha }}

      - name: Azure logout
        run: |
          az logout

```
