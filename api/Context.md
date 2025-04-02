# CodePush Server Azure Deployment Context

## Project Overview

- **Repository**: https://github.com/symphco/ml-code-push-server.git
- **Environments**:
  - Development (codepush-devml)
  - Production (pending deployment)

## Infrastructure Architecture

- **Azure Structure**:
  - Single subscription with multiple resource groups
  - Separate resource groups for dev and prod environments
  - Each environment contains App Service, App Service Plan, and Storage Account

## Deployment Status

### Completed Actions

- Created resource groups:

  ```bash
  az group create --name codepush-dev-rg --location eastus
  az group create --name codepush-prod-rg --location eastus
  ```

- Deployed infrastructure with bicep:

  ```bash
  az deployment group create --resource-group codepush-dev-rg --template-file ./codepush-infrastructure.bicep --parameters project_suffix=devml az_location=eastus microsoft_client_id=<client-id> microsoft_client_secret=<client-secret>
  ```

- Setup Microsoft OAuth applications in Azure AD with redirect URIs:

  - Dev: `https://codepush-devml.azurewebsites.net/auth/callback/microsoft` and `https://codepush-devml.azurewebsites.net/auth/callback/azure-ad`
  - Prod: Equivalent URLs with prod suffix

- Configured Git deployment:

  ```bash
  # Add GitHub authentication token
  az webapp deployment source update-token --git-token <github-PAT> --token-type github

  # Configure deployment source
  az webapp deployment source config --resource-group codepush-dev-rg --name codepush-devml --repo-url https://github.com/symphco/ml-code-push-server.git --branch main

  # Set project folder location
  az webapp config appsettings set --resource-group codepush-dev-rg --name codepush-devml --settings PROJECT=api
  ```

### Resources Created

- Dev Environment:
  - Resource Group: codepush-dev-rg
  - App Service: codepush-devml
  - Storage Account: with unique name based on project suffix

## Technical Requirements & Constraints

### Azure Naming Limitations

- Project suffix:
  - Only letters allowed (no numbers or special characters)
  - Maximum 15 characters
- Storage account names must be globally unique across all Azure

### Authentication

- Using Microsoft OAuth (work and personal accounts)
- GitHub authentication is optional alternative

### Environment Variables

- Set directly in Azure App Service Configuration
- No need for local `.env` file

### Git Repository Access

- Only HTTPS URLs supported (not SSH)
- Private repositories require GitHub Personal Access Token with "repo" scope

## Common Issues & Solutions

1. **Storage Account Name Conflicts**:

   - Error: "The storage account named codepushstorageXXX is already taken"
   - Solution: Use more unique suffix

2. **GitHub Authentication Issues**:

   - Error: "Cannot find SourceControlToken with name GitHub"
   - Solution: Add GitHub PAT token using `az webapp deployment source update-token`

3. **Git URL Format**:
   - Problem: SSH format not supported
   - Solution: Use HTTPS format for GitHub repositories

## Client Integration

### Android Configuration

Add to `strings.xml`:

```xml
<string moduleConfig="true" name="CodePushServerUrl">https://codepush-devml.azurewebsites.net</string>
```

### iOS Configuration

Add to `Info.plist`:

```xml
<key>CodePushServerURL</key>
<string>https://codepush-devml.azurewebsites.net</string>
```

## Monitoring & Management

- Deployment status: Azure Portal > App Service > Deployment Center
- Manual deployment: Use "Sync" button in Deployment Center
- Environment variables: Azure Portal > App Service > Configuration > Application settings

## Next Steps

1. Verify dev deployment is working correctly
2. Complete production environment deployment if needed
3. Test client integration with mobile apps
4. Establish CI/CD workflow for ongoing development
