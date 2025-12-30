# YellNo Azure Infrastructure

## Resource Naming Convention

```
Pattern: yellno-{service}-{environment}

Environments:
- dev
- staging
- prod

Examples:
- yellno-func-prod
- yellno-storage-dev
- yellno-speech-prod
```

---

## Resource Group

```
Name: yellno-{environment}-rg
Region: East US (or preferred region with Speech Services availability)
```

---

## Azure Services Required

### 1. Azure Functions

**Resource:** `yellno-func-{env}`

**Configuration:**
- Runtime: .NET 8 (isolated worker) or Node.js 20
- Plan: Consumption (scale to zero when idle)
- OS: Linux

**App Settings:**
```
AZURE_STORAGE_CONNECTION_STRING: (from storage account)
SPEECH_SERVICE_KEY: (from Speech Services)
SPEECH_SERVICE_REGION: eastus
GOOGLE_CLIENT_ID: (from Google Cloud Console)
FUNCTIONS_WORKER_RUNTIME: dotnet-isolated | node
```

**Functions to Deploy:**

| Function | Trigger | Route |
|----------|---------|-------|
| CreateFamily | HTTP | POST /api/family |
| GetFamily | HTTP | GET /api/family |
| UpdateFamily | HTTP | PATCH /api/family |
| DeleteFamily | HTTP | DELETE /api/family |
| AddMember | HTTP | POST /api/family/members |
| GetMembers | HTTP | GET /api/family/members |
| UpdateMember | HTTP | PATCH /api/family/members/{id} |
| DeleteMember | HTTP | DELETE /api/family/members/{id} |
| EnrollVoiceprint | HTTP | POST /api/voiceprint/enroll |
| IdentifyVoiceprint | HTTP | POST /api/voiceprint/identify |
| GetConfirmations | HTTP | GET /api/voiceprint/confirmations |
| RespondConfirmation | HTTP | POST /api/voiceprint/confirmations/{id} |
| Analyze | HTTP | POST /api/analyze |
| GetIncidents | HTTP | GET /api/incidents |
| RemoveIncident | HTTP | DELETE /api/incidents/{id} |
| GetStats | HTTP | GET /api/stats |
| RegisterDevice | HTTP | POST /api/devices |
| DeviceHeartbeat | HTTP | POST /api/devices/{id}/heartbeat |
| DeleteDevice | HTTP | DELETE /api/devices/{id} |
| CleanupDedupeWindow | Timer | 0 */1 * * * * (every minute) |
| CleanupExpiredConfirmations | Timer | 0 0 * * * * (every hour) |
| ComputeStatsSummaries | Timer | 0 0 0 * * * (daily at midnight) |

---

### 2. Azure Storage Account

**Resource:** `yellnostorage{env}` (no hyphens, lowercase)

**Configuration:**
- Performance: Standard
- Replication: LRS (Locally Redundant) for dev/staging, GRS for prod
- Account kind: StorageV2

**Table Storage Tables:**

| Table Name | Partition Strategy | Purpose |
|------------|-------------------|---------|
| families | OwnerId | Family accounts |
| members | FamilyId | Family members |
| incidents | FamilyId | Yelling incidents |
| dedupewindow | FamilyId | Short-lived dedupe hashes |
| pendingconfirmations | FamilyId | Voiceprint confirmations |
| devices | FamilyId | Registered devices |
| statssummaries | FamilyId_Interval | Precomputed stats |

**Blob Storage Containers:**

| Container | Purpose | Access |
|-----------|---------|--------|
| voiceprints | Managed by Azure Speaker Recognition | Private |

---

### 3. Azure Cognitive Services - Speech

**Resource:** `yellno-speech-{env}`

**Configuration:**
- Kind: SpeechServices
- SKU: S0 (Standard)
- Region: Must match Functions region

**Services Used:**

| Service | API | Purpose |
|---------|-----|---------|
| Speech-to-Text | REST/SDK | Transcription |
| Speaker Recognition | REST | Voiceprint enrollment/identification |

**Speaker Recognition Quotas (S0):**
- Enrollment: 10,000 profiles per subscription
- Identification: 10 candidates per request (sufficient for family size)

**Endpoints:**
```
Speech-to-Text: https://{region}.stt.speech.microsoft.com
Speaker Recognition: https://{region}.api.cognitive.microsoft.com/speaker
```

---

### 4. Application Insights

**Resource:** `yellno-insights-{env}`

**Configuration:**
- Workspace-based
- Linked to Functions app

**Monitoring:**
- Function execution metrics
- API response times
- Error rates
- Custom events (yelling detections, enrollments)

**Custom Events to Log:**
```
YellingDetected: familyId, memberId, confidence, mood
VoiceprintEnrolled: familyId, memberId
VoiceprintConfirmed: familyId, memberId
IncidentRemoved: familyId, incidentId
FamilyCreated: familyId
```

---

### 5. Azure Key Vault (Optional but Recommended for Prod)

**Resource:** `yellno-kv-{env}`

**Secrets:**
```
SpeechServiceKey
GoogleClientId
StorageConnectionString
```

**Access:**
- Functions app has managed identity
- Managed identity granted "Key Vault Secrets User" role

**Function App Reference:**
```
@Microsoft.KeyVault(VaultName=yellno-kv-prod;SecretName=SpeechServiceKey)
```

---

## Infrastructure Provisioning

### Azure CLI Commands

```bash
# Variables
ENV="dev"
REGION="eastus"
RG="yellno-${ENV}-rg"

# Resource Group
az group create --name $RG --location $REGION

# Storage Account
az storage account create \
  --name "yellnostorage${ENV}" \
  --resource-group $RG \
  --location $REGION \
  --sku Standard_LRS \
  --kind StorageV2

# Create Tables
CONN_STR=$(az storage account show-connection-string --name "yellnostorage${ENV}" --resource-group $RG --query connectionString -o tsv)

az storage table create --name families --connection-string $CONN_STR
az storage table create --name members --connection-string $CONN_STR
az storage table create --name incidents --connection-string $CONN_STR
az storage table create --name dedupewindow --connection-string $CONN_STR
az storage table create --name pendingconfirmations --connection-string $CONN_STR
az storage table create --name devices --connection-string $CONN_STR
az storage table create --name statssummaries --connection-string $CONN_STR

# Speech Services
az cognitiveservices account create \
  --name "yellno-speech-${ENV}" \
  --resource-group $RG \
  --kind SpeechServices \
  --sku S0 \
  --location $REGION \
  --yes

# Get Speech Key
SPEECH_KEY=$(az cognitiveservices account keys list --name "yellno-speech-${ENV}" --resource-group $RG --query key1 -o tsv)

# Application Insights
az monitor app-insights component create \
  --app "yellno-insights-${ENV}" \
  --location $REGION \
  --resource-group $RG \
  --kind web

INSIGHTS_KEY=$(az monitor app-insights component show --app "yellno-insights-${ENV}" --resource-group $RG --query instrumentationKey -o tsv)

# Function App
az functionapp create \
  --name "yellno-func-${ENV}" \
  --resource-group $RG \
  --storage-account "yellnostorage${ENV}" \
  --consumption-plan-location $REGION \
  --runtime dotnet-isolated \
  --runtime-version 8 \
  --functions-version 4 \
  --os-type Linux \
  --app-insights "yellno-insights-${ENV}" \
  --app-insights-key $INSIGHTS_KEY

# Configure Function App Settings
az functionapp config appsettings set \
  --name "yellno-func-${ENV}" \
  --resource-group $RG \
  --settings \
    "SPEECH_SERVICE_KEY=${SPEECH_KEY}" \
    "SPEECH_SERVICE_REGION=${REGION}" \
    "GOOGLE_CLIENT_ID=your-google-client-id"

# Key Vault (prod only)
az keyvault create \
  --name "yellno-kv-${ENV}" \
  --resource-group $RG \
  --location $REGION

# Enable managed identity on Function App
az functionapp identity assign \
  --name "yellno-func-${ENV}" \
  --resource-group $RG

# Grant Key Vault access to Function App
FUNC_IDENTITY=$(az functionapp identity show --name "yellno-func-${ENV}" --resource-group $RG --query principalId -o tsv)

az keyvault set-policy \
  --name "yellno-kv-${ENV}" \
  --object-id $FUNC_IDENTITY \
  --secret-permissions get list
```

---

## Bicep Template (Alternative)

```bicep
// main.bicep

param environment string = 'dev'
param location string = resourceGroup().location
param googleClientId string

var baseName = 'yellno'
var storageAccountName = '${baseName}storage${environment}'

resource storageAccount 'Microsoft.Storage/storageAccounts@2023-01-01' = {
  name: storageAccountName
  location: location
  sku: {
    name: environment == 'prod' ? 'Standard_GRS' : 'Standard_LRS'
  }
  kind: 'StorageV2'
}

resource speechService 'Microsoft.CognitiveServices/accounts@2023-05-01' = {
  name: '${baseName}-speech-${environment}'
  location: location
  kind: 'SpeechServices'
  sku: {
    name: 'S0'
  }
}

resource appInsights 'Microsoft.Insights/components@2020-02-02' = {
  name: '${baseName}-insights-${environment}'
  location: location
  kind: 'web'
  properties: {
    Application_Type: 'web'
  }
}

resource functionApp 'Microsoft.Web/sites@2023-01-01' = {
  name: '${baseName}-func-${environment}'
  location: location
  kind: 'functionapp,linux'
  properties: {
    siteConfig: {
      linuxFxVersion: 'DOTNET-ISOLATED|8.0'
      appSettings: [
        { name: 'AzureWebJobsStorage', value: storageAccount.properties.primaryEndpoints.blob }
        { name: 'FUNCTIONS_WORKER_RUNTIME', value: 'dotnet-isolated' }
        { name: 'SPEECH_SERVICE_KEY', value: speechService.listKeys().key1 }
        { name: 'SPEECH_SERVICE_REGION', value: location }
        { name: 'GOOGLE_CLIENT_ID', value: googleClientId }
        { name: 'APPINSIGHTS_INSTRUMENTATIONKEY', value: appInsights.properties.InstrumentationKey }
      ]
    }
  }
}

output functionAppName string = functionApp.name
output storageAccountName string = storageAccount.name
output speechServiceEndpoint string = speechService.properties.endpoint
```

---

## Cost Estimates (Monthly)

### Development Environment

| Service | Tier | Estimated Cost |
|---------|------|----------------|
| Azure Functions | Consumption | ~$0 (free tier covers dev) |
| Storage Account | LRS | ~$1-2 |
| Speech Services | S0 | ~$0-5 (low usage) |
| Application Insights | Basic | ~$0 (free tier) |
| **Total** | | **~$5/month** |

### Production Environment (1,000 active families)

| Service | Tier | Estimated Cost |
|---------|------|----------------|
| Azure Functions | Consumption | ~$20-50 |
| Storage Account | GRS | ~$10-20 |
| Speech Services | S0 | ~$100-300 |
| Application Insights | Basic | ~$10-20 |
| Key Vault | Standard | ~$1 |
| **Total** | | **~$150-400/month** |

**Cost Drivers:**
- Speech-to-Text: $1/hour of audio
- Speaker Recognition: $1/1,000 transactions
- Functions: $0.20/million executions
- Storage: Negligible at this scale

---

## Scaling Considerations

### Current Design (Consumption Plan)
- Auto-scales to zero
- Max 200 instances per function app
- Cold start latency: 1-3 seconds
- Suitable for: MVP, low-medium traffic

### Future Scaling Options

**Premium Plan (if cold starts become problematic):**
```
- Always-ready instances: 1
- Pre-warmed instances: 2
- Cost: ~$150/month base
```

**Dedicated Plan (high traffic):**
```
- App Service Plan B1+
- Predictable performance
- Cost: ~$50-200/month
```

**Speech Services Scaling:**
```
- S0 handles 100 concurrent requests
- For higher load: multiple cognitive service instances
- Or upgrade to commitment tier for discounts
```

---

## Security Checklist

- [ ] Enable HTTPS only on Function App
- [ ] Configure CORS to allow only app domains
- [ ] Use managed identity for Key Vault access
- [ ] Enable diagnostic logging
- [ ] Set up alerts for error rates > 1%
- [ ] Enable soft delete on storage account
- [ ] Restrict storage account network access (prod)
- [ ] Rotate Speech Service keys quarterly
- [ ] Review access policies quarterly

---

## Deployment Pipeline

### GitHub Actions Workflow

```yaml
# .github/workflows/deploy.yml

name: Deploy Azure Functions

on:
  push:
    branches: [main]
    paths:
      - 'api/**'

env:
  AZURE_FUNCTIONAPP_NAME: yellno-func-prod
  AZURE_FUNCTIONAPP_PACKAGE_PATH: 'api'
  DOTNET_VERSION: '8.0.x'

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup .NET
        uses: actions/setup-dotnet@v3
        with:
          dotnet-version: ${{ env.DOTNET_VERSION }}

      - name: Build
        run: dotnet build --configuration Release
        working-directory: ${{ env.AZURE_FUNCTIONAPP_PACKAGE_PATH }}

      - name: Publish
        run: dotnet publish --configuration Release --output ./output
        working-directory: ${{ env.AZURE_FUNCTIONAPP_PACKAGE_PATH }}

      - name: Deploy to Azure Functions
        uses: Azure/functions-action@v1
        with:
          app-name: ${{ env.AZURE_FUNCTIONAPP_NAME }}
          package: '${{ env.AZURE_FUNCTIONAPP_PACKAGE_PATH }}/output'
          publish-profile: ${{ secrets.AZURE_FUNCTIONAPP_PUBLISH_PROFILE }}
```

---

## Environment Promotion

```
dev → staging → prod

Promotion Checklist:
1. All tests passing
2. Manual QA on staging
3. Performance baseline met
4. Security scan clean
5. Rollback plan documented

Staging mirrors prod:
- Same SKUs
- Same configuration
- Separate data
```