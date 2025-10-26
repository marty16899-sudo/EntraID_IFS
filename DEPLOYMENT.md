# Azure Deployment Guide

This guide covers deploying the Access Package API to Azure.

## Deployment Options

### Option 1: Azure App Service (Recommended)

#### Prerequisites
- Azure CLI installed
- Azure subscription

#### Steps

1. **Login to Azure**
```bash
az login
```

2. **Create Resource Group**
```bash
az group create --name access-package-api-rg --location eastus
```

3. **Create App Service Plan**
```bash
az appservice plan create \
  --name access-package-api-plan \
  --resource-group access-package-api-rg \
  --sku B1 \
  --is-linux
```

4. **Create Web App**
```bash
az webapp create \
  --name access-package-api \
  --resource-group access-package-api-rg \
  --plan access-package-api-plan \
  --runtime "NODE:18-lts"
```

5. **Configure Environment Variables**
```bash
az webapp config appsettings set \
  --resource-group access-package-api-rg \
  --name access-package-api \
  --settings \
    TENANT_ID="your-tenant-id" \
    CLIENT_ID="your-client-id" \
    CLIENT_SECRET="your-client-secret" \
    NODE_ENV="production"
```

6. **Deploy Code**
```bash
# From your project directory
az webapp deployment source config-zip \
  --resource-group access-package-api-rg \
  --name access-package-api \
  --src ./access-package-api.zip
```

Or use GitHub Actions for CI/CD (see GitHub Actions section below)

7. **Enable HTTPS Only**
```bash
az webapp update \
  --resource-group access-package-api-rg \
  --name access-package-api \
  --https-only true
```

### Option 2: Azure Functions

Azure Functions is great for serverless deployment with automatic scaling.

1. **Install Azure Functions Core Tools**
```bash
npm install -g azure-functions-core-tools@4
```

2. **Convert to Functions** (requires code refactoring)
- Each route becomes a separate function
- Use HTTP triggers
- Configure function.json for each endpoint

3. **Deploy**
```bash
func azure functionapp publish <YOUR_FUNCTION_APP_NAME>
```

### Option 3: Azure Container Instances

For Docker-based deployment:

1. **Create Dockerfile** (in project root):
```dockerfile
FROM node:18-alpine
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production
COPY . .
EXPOSE 3001
CMD ["node", "server.js"]
```

2. **Build and Push to Azure Container Registry**
```bash
az acr create --resource-group access-package-api-rg --name accesspackageacr --sku Basic
az acr build --registry accesspackageacr --image access-package-api:latest .
```

3. **Deploy to Container Instances**
```bash
az container create \
  --resource-group access-package-api-rg \
  --name access-package-api \
  --image accesspackageacr.azurecr.io/access-package-api:latest \
  --dns-name-label access-package-api \
  --ports 3001
```

## GitHub Actions CI/CD

Create `.github/workflows/azure-deploy.yml`:

```yaml
name: Deploy to Azure App Service

on:
  push:
    branches: [ main ]

env:
  AZURE_WEBAPP_NAME: access-package-api
  NODE_VERSION: '18.x'

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v3
    
    - name: Set up Node.js
      uses: actions/setup-node@v3
      with:
        node-version: ${{ env.NODE_VERSION }}
        
    - name: npm install and build
      run: |
        npm ci
        
    - name: Deploy to Azure Web App
      uses: azure/webapps-deploy@v2
      with:
        app-name: ${{ env.AZURE_WEBAPP_NAME }}
        publish-profile: ${{ secrets.AZURE_WEBAPP_PUBLISH_PROFILE }}
        package: .
```

## Post-Deployment Configuration

### 1. Update Azure App Registration

Add your deployed API URL to:
- Redirect URIs
- CORS allowed origins

### 2. Configure CORS in Azure

```bash
az webapp cors add \
  --resource-group access-package-api-rg \
  --name access-package-api \
  --allowed-origins "https://teams.microsoft.com"
```

### 3. Enable Application Insights (Monitoring)

```bash
az monitor app-insights component create \
  --app access-package-api-insights \
  --location eastus \
  --resource-group access-package-api-rg \
  --application-type web

# Link to App Service
az webapp config appsettings set \
  --resource-group access-package-api-rg \
  --name access-package-api \
  --settings APPINSIGHTS_INSTRUMENTATIONKEY="your-key"
```

### 4. Set Up Custom Domain (Optional)

```bash
az webapp config hostname add \
  --webapp-name access-package-api \
  --resource-group access-package-api-rg \
  --hostname api.yourdomain.com
```

## Health Checks

Test your deployment:

```bash
# Health check
curl https://access-package-api.azurewebsites.net/health

# Auth health
curl https://access-package-api.azurewebsites.net/api/auth/health
```

## Monitoring and Logs

### View Logs
```bash
az webapp log tail \
  --resource-group access-package-api-rg \
  --name access-package-api
```

### Stream Logs
```bash
az webapp log config \
  --resource-group access-package-api-rg \
  --name access-package-api \
  --application-logging filesystem \
  --level information

az webapp log tail \
  --resource-group access-package-api-rg \
  --name access-package-api
```

## Security Best Practices

1. **Use Azure Key Vault for Secrets**
```bash
# Create Key Vault
az keyvault create \
  --name access-package-kv \
  --resource-group access-package-api-rg \
  --location eastus

# Add secrets
az keyvault secret set \
  --vault-name access-package-kv \
  --name ClientSecret \
  --value "your-client-secret"

# Reference in App Service
az webapp config appsettings set \
  --resource-group access-package-api-rg \
  --name access-package-api \
  --settings CLIENT_SECRET="@Microsoft.KeyVault(SecretUri=https://access-package-kv.vault.azure.net/secrets/ClientSecret/)"
```

2. **Enable Managed Identity**
```bash
az webapp identity assign \
  --resource-group access-package-api-rg \
  --name access-package-api
```

3. **Configure IP Restrictions** (if needed)
```bash
az webapp config access-restriction add \
  --resource-group access-package-api-rg \
  --name access-package-api \
  --rule-name allow-corporate \
  --action Allow \
  --ip-address 203.0.113.0/24 \
  --priority 100
```

## Scaling

### Manual Scaling
```bash
az appservice plan update \
  --name access-package-api-plan \
  --resource-group access-package-api-rg \
  --sku P1V2
```

### Auto-Scaling
```bash
az monitor autoscale create \
  --resource-group access-package-api-rg \
  --resource access-package-api \
  --resource-type Microsoft.Web/serverfarms \
  --name autoscale-access-package \
  --min-count 1 \
  --max-count 5 \
  --count 1
```

## Troubleshooting

### Check Deployment Status
```bash
az webapp deployment list \
  --resource-group access-package-api-rg \
  --name access-package-api
```

### SSH into Container
```bash
az webapp ssh \
  --resource-group access-package-api-rg \
  --name access-package-api
```

### Common Issues

1. **502 Bad Gateway**: Check application logs for startup errors
2. **Authentication fails**: Verify environment variables are set correctly
3. **CORS errors**: Add Teams origin to CORS configuration
4. **Slow performance**: Consider scaling up or enabling Application Insights

## Cost Estimation

- **Basic App Service (B1)**: ~$13/month
- **Standard App Service (S1)**: ~$70/month
- **Application Insights**: Pay-as-you-go (usually < $5/month for small apps)
- **Key Vault**: ~$0.03 per 10,000 operations

## Cleanup

To remove all resources:
```bash
az group delete --name access-package-api-rg --yes --no-wait
```
