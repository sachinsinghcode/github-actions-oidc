# Github Actions OIDC
Sample Application to be deployed using github Actions OIDC to Azure Web Apps

## Pre-Requisites
We'll be taking example of a NodeJS Application Being Deployed to Azure Web App (App Service Plan based) to learn difference between publish profile based deployment and OIDC(App Registration and Service Principal in Azure) Based Deployments

Keep this file with any *filename* inside this path below for your NodeJs Application = /.github/workflows/*filename*.yaml

### For Publish Profile Deployments(Not OIDC and not secure)

The Values __AZURE_WEBAPP_NAME__ and __AZURE_WEBAPP_PUBLISH_PROFILE__ are to be fetched from azure portal and stored either in repository secrets or environent secrets

### Publish Profile Pipeline Yaml
```
name: CI-CD Pipeline Credentials Based

on:
  push:
    branches:
      - master
    tags:
      - 'v*'
  pull_request:
    branches:
      - master

env:
  NODE_VERSION: '22'

jobs:

  ci:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout Code
        uses: actions/checkout@v4

      - name: Setup Node
        uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}

      - name: Install Dependencies
        run: npm install

      - name: Run Tests
        run: npm test

      - name: Build App
        run: npm run build


  deploy:
    needs: ci
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/master'
    environment: Azure   

    steps:
      - name: Checkout Code
        uses: actions/checkout@v4

      - name: Setup Node
        uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}

      - name: Install Dependencies
        run: npm install

      - name: Build
        run: npm run build


      - name: Deploy to Azure Web App
        uses: azure/webapps-deploy@v3
        with:
          app-name: ${{ secrets.AZURE_WEBAPP_NAME }}
          publish-profile: ${{ secrets.AZURE_WEBAPP_PUBLISH_PROFILE }}
          package: .
```


### For OIDC Deployments

##### Step 1: Create Azure AD App (Service Principal)

Run:

`az ad app create --display-name "github-actions-oidc"`

Then:

`az ad sp create --id <APP_ID>`

#### Step 2: Assign permissions

Give it access to your Web App (or resource group):

```
az role assignment create \
  --assignee <APP_ID> \
  --role contributor \
  --scope /subscriptions/<SUBSCRIPTION_ID>/resourceGroups/<RESOURCE_GROUP>
```

#### Step 3: Add Federated Credential (MOST IMPORTANT)

This is what enables OIDC.

Run:

```
az ad app federated-credential create \
  --id <APP_ID> \
  --parameters '{
    "name": "github-actions",
    "issuer": "https://token.actions.githubusercontent.com",
    "subject": "repo:<GITHUB_ORG>/<REPO_NAME>:ref:refs/heads/main",
    "audiences": ["api://AzureADTokenExchange"]
  }'
```

In case of escape character issue in json use
```
az ad app federated-credential create \
  --id <APP_ID> \
  --parameters '{
    \"name\":\"github-actions\",
    \"issuer\":\"https://token.actions.githubusercontent.com\",
    \"subject\":\"repo:<GITHUB_ORG>/<REPO_NAME>:ref:refs/heads/main\",
    \"audiences\":[\"api://AzureADTokenExchange\"]
    }'
```


What this means

`repo:ORG/REPO:ref:refs/heads/main`

Meaning Only workflows from:

That repo -> That branch (main) -> can deploy

for environment based use subject as 
`repo:ORG/REPO:environment:azure`

for tag based usec subject as
`repo:ORG/REPO:refs/tags/v*`

#### Step 4: Add GitHub Secrets (only IDs, not secrets)

In GitHub → Secrets:

AZURE_CLIENT_ID → App (client) ID


AZURE_TENANT_ID → Tenant ID

AZURE_SUBSCRIPTION_ID → Subscription ID

No passwords needed

### OIDC Pipeline Yaml

```
name: CI-CD Pipeline OIDC

on:

  push:
    branches:
      - master
    tags:
      - 'v*'
  pull_request:
    branches:
      - master

env:
  NODE_VERSION: '22'

jobs:
  ci:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout Code
        uses: actions/checkout@v4

      - name: Setup Node
        uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}

      - name: Install Dependencies
        run: npm install

      - name: Run Tests
        run: npm test

      - name: Build App
        run: npm run build

  deploy:
    needs: ci
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/master'
    environment: Azure  

    permissions:
      id-token: write 
      contents: read 

    steps:
      - name: Checkout Code
        uses: actions/checkout@v4

      - name: Setup Node
        uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}

      - name: Install Dependencies
        run: npm install

      - name: Build
        run: npm run build

      - name: Azure Login (OIDC)
        uses: azure/login@v2
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

      - name: Deploy to Azure Web App
        uses: azure/webapps-deploy@v3
        with:
          app-name: ${{ secrets.AZURE_WEBAPP_NAME }}
          package: .
      
```
