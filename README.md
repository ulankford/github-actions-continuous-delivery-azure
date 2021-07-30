# Continuous Delivery with Azure

GitHub Actions is cloud agnostic, so any public cloud provider will work but in this example we will focus on Microsoft Azure.

Resources we will be working with:
- A web app is how we'll be deploying our application to Azure.
- A resource group is a collection of resources, like web apps and virtual machines (VMs).
- An App Service plan is what runs our web app and manages the billing (our app should run for free).

Through the power of GitHub Actions, we can create, configure, and destroy these resources through our workflow files

**Set up** Azure resources will run if the pull request contains a label with the name "spin up environment".
**Destroy** Azure resources will run if the pull request contains a label with the name "destroy environment".

## Setting up Azure resources

The first job in the spinup-destroy workflow sets up the Azure resources as follows:

- Logs into your Azure account with the azure/login action. The AZURE_CREDENTIALS secret you created earlier is used for authentication.
- Creates an Azure resource group by running az group create on the Azure CLI, which is pre-installed on the GitHub-hosted runner.
- Creates an App Service plan by running az appservice plan create on the Azure CLI.
- Creates a web app by running az webapp create on the Azure CLI.
- Configures the newly created web app to use GitHub Packages by using az webapp config on the Azure CLI. Azure can be configured to use its own Azure Container Registry, DockerHub, or a custom (private) registry. In this case, we'll configure GitHub Packages as a custom registry.


```
steps:
      - name: Checkout repository
        uses: actions/checkout@v2

      - name: Azure login
        uses: azure/login@v1
        with:
          creds: ${{ secrets.AZURE_CREDENTIALS }}

      - name: Create Azure resource group
        if: success()
        run: |
          az group create --location ${{env.AZURE_LOCATION}} --name ${{env.AZURE_RESOURCE_GROUP}} --subscription ${{secrets.AZURE_SUBSCRIPTION_ID}}
      - name: Create Azure app service plan
        if: success()
        run: |
          az appservice plan create --resource-group ${{env.AZURE_RESOURCE_GROUP}} --name ${{env.AZURE_APP_PLAN}} --is-linux --sku F1 --subscription ${{secrets.AZURE_SUBSCRIPTION_ID}}
      - name: Create webapp resource
        if: success()
        run: |
          az webapp create --resource-group ${{ env.AZURE_RESOURCE_GROUP }} --plan ${{ env.AZURE_APP_PLAN }} --name ${{ env.AZURE_WEBAPP_NAME }}  --deployment-container-image-name nginx --subscription ${{secrets.AZURE_SUBSCRIPTION_ID}}
      - name: Configure webapp to use GitHub Packages
        if: success()
        run: |
          az webapp config container set --docker-custom-image-name nginx --docker-registry-server-password ${{secrets.GITHUB_TOKEN}} --docker-registry-server-url https://docker.pkg.github.com --docker-registry-server-user ${{github.actor}} --name ${{ env.AZURE_WEBAPP_NAME }} --resource-group ${{ env.AZURE_RESOURCE_GROUP }} --subscription ${{secrets.AZURE_SUBSCRIPTION_ID}}
```

## Destroying Azure resources
The second job destroys Azure resources so that you do not use your free minutes or incur billing. The job works as follows:

- Logs into your Azure account with the azure/login action. The AZURE_CREDENTIALS secret you created earlier is used for authentication.
- Deletes the resource group we created earlier using az group delete on the Azure CLI.


```
steps:
      - name: Checkout repository
        uses: actions/checkout@v2

      - name: Azure login
        uses: azure/login@v1
        with:
          creds: ${{ secrets.AZURE_CREDENTIALS }}

      - name: Destroy Azure environment
        if: success()
        run: |
          az group delete --name ${{env.AZURE_RESOURCE_GROUP}} --subscription ${{secrets.AZURE_SUBSCRIPTION_ID}} --yes
 ```

## Conditionals

The specific setup or destroy job will run depending on the label attached to the pull request.

```
  jobs:
  setup-up-azure-resources:
    runs-on: ubuntu-latest

    if: contains(github.event.pull_request.labels.*.name, 'spin up environment')
```

```
  destroy-azure-resources:
    runs-on: ubuntu-latest

    if: contains(github.event.pull_request.labels.*.name, 'destroy environment')
```

