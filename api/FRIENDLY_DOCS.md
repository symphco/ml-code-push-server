# Setting Up and Deploying CodePush Server to Azure: A Beginner's Guide

This guide will walk you through the entire process of setting up the CodePush Server locally and deploying it to Azure, explaining how all the configuration files work together.

## Table of Contents

1. [What is CodePush?](#what-is-codepush)
2. [Repository Structure Overview](#repository-structure-overview)
3. [Setting Up Locally](#setting-up-locally)
4. [Deploying to Azure](#deploying-to-azure)
5. [GitHub Actions Workflows](#github-actions-workflows)
6. [Authentication with OAuth Apps](#authentication-with-oauth-apps)
7. [Security Considerations](#security-considerations)
8. [Metrics](#metrics)
9. [Troubleshooting](#troubleshooting)
10. [Configuring React Native Apps to Use Your Server](#configuring-react-native-apps-to-use-your-server)
11. [Quick Start: Step by Step Commands](#quick-start-step-by-step-commands)

## What is CodePush?

CodePush is a service that allows React Native developers to deploy mobile app updates directly to users' devices without going through the app store review process. The CodePush Server is the backend service that powers this functionality, allowing you to host your own CodePush service.

## Repository Structure Overview

The key files we'll be working with include:

- **codepush-infrastructure.bicep**: Defines the Azure infrastructure needed for deployment
- **README.md**: Contains basic setup instructions
- **.github/workflows/development_codepush-devml.yml**: GitHub Actions workflow for automatic deployment
- **.github/workflows/codeql.yml**: GitHub Actions workflow for code security scanning
- **.env.example**: Template for environment variables

## Setting Up Locally

### Prerequisites

1. **Node.js**: Make sure you have Node.js 18 LTS installed
2. **Azure CLI**: Install the Azure Command-Line Interface
3. **Azurite**: For local Azure Storage emulation
4. **OAuth Apps**: You'll need GitHub and/or Microsoft OAuth applications for user authentication

### Step 1: Clone the Repository

```bash
git clone <repository-url>
cd ml-code-push-server
```

### Step 2: Set Up Environment Variables

```bash
cd api
cp .env.example .env
```

Now edit the `.env` file to set your environment variables. For local development, make sure to set:

- `EMULATED=true` (to use Azurite instead of real Azure Storage)
- OAuth credentials for authentication (GitHub/Microsoft)

### Step 3: Install Dependencies

```bash
npm install
```

### Step 4: Start Azurite

Follow the Azurite [installation instructions](https://learn.microsoft.com/en-us/azure/storage/common/storage-use-azurite) and start it:

```bash
# Example using npm
npm install -g azurite
azurite --silent --location azurite --debug azurite/debug.log
```

### Step 5: Build and Run the Server

```bash
npm run build
npm run start:env
```

By default, the server runs on HTTP (localhost:3000). For HTTPS:

1. Create a `certs` directory with `cert.key` and `cert.crt` files
2. Set `HTTPS=true` in your `.env` file
3. The server will run on localhost:8443

## Authentication with OAuth Apps

### Why OAuth Apps are Required for CodePush

OAuth Apps are **essential** for your CodePush server deployment because:

1. **User Authentication**: They provide the authentication system for your CodePush server, allowing users to log in with existing GitHub or Microsoft accounts.

2. **Access Control**: They determine who can access your CodePush server to deploy updates to mobile applications.

3. **Security**: They eliminate the need to build your own authentication system or store user credentials.

4. **Deployment Management**: They control who has permission to create and manage deployments for different applications.

Without at least one OAuth provider configured, your CodePush server would have no way to authenticate users, making it impossible to use the system for deploying updates.

### Setting Up a GitHub OAuth App

1. Go to https://github.com/settings/developers
2. Click "New OAuth App"
3. Fill in required details:
   - Application name: "CodePush Server" (or any name you prefer)
   - Homepage URL: Your CodePush server URL (e.g., `https://codepush-YOUR-SUFFIX.azurewebsites.net` or `http://localhost:3000` for local)
   - Authorization callback URL: `{server-url}/auth/callback/github` (this must be exact)
4. Click "Register application" to get your Client ID
5. Generate a new Client Secret
6. Save both values for your CodePush server configuration

### Setting Up a Microsoft OAuth App

1. Register an app at https://portal.azure.com → Azure Active Directory → App registrations → New registration
2. Name your application (e.g., "CodePush Server")
3. Choose the account type based on your needs:
   - For both personal and work accounts: "Accounts in any organizational directory and personal Microsoft accounts"
   - For work accounts only: Select appropriate tenant option
   - For personal accounts only: "Personal Microsoft accounts only"
4. Set redirect URIs:
   - For personal accounts: `{server-url}/auth/callback/microsoft`
   - For work accounts: `{server-url}/auth/callback/azure-ad`
5. After registration, copy the Application (client) ID
6. Create a client secret: Certificates & secrets → New client secret
7. Copy and save both values for your CodePush server configuration

### Adding OAuth Credentials to Your CodePush Server

#### For Local Development:

Add to your `.env` file:

```
# GitHub OAuth
GITHUB_CLIENT_ID=your_github_client_id
GITHUB_CLIENT_SECRET=your_github_client_secret

# Microsoft OAuth
MICROSOFT_CLIENT_ID=your_microsoft_client_id
MICROSOFT_CLIENT_SECRET=your_microsoft_client_secret
# Only needed for single tenant work account setup
MICROSOFT_TENANT_ID=your_tenant_id
```

#### For Azure Deployment:

Include these parameters when running the bicep deployment or add them in the Azure Portal under your Web App → Configuration → Application settings.

## Deploying to Azure

### Step 1: Login to Azure

```bash
az login
az account set --subscription <subscription-id>
```

### Step 2: Create a Resource Group

```bash
az group create --name <resource-group-name> --location eastus
```

### Step 3: Understanding the Bicep Template

The `codepush-infrastructure.bicep` file defines all the resources needed to run CodePush:

- **App Service Plan**: Hosts your application (named: codepush-asp-{suffix})
- **Storage Account**: Stores your CodePush packages (named: codepushstorage{suffix})
- **Web App**: The actual CodePush server (named: codepush-{suffix})

Important parameters:

- `project_suffix`: Used in naming all resources (letters only, max 15 chars)
- `az_location`: Azure region (default: eastus)
- OAuth credentials for GitHub and Microsoft authentication
- `logging`: Boolean parameter to enable/disable logging (default: true)

### Step 4: Deploy the Infrastructure

```bash
az deployment group create \
  --resource-group <resource-group-name> \
  --template-file ./codepush-infrastructure.bicep \
  --parameters project_suffix=<suffix> \
  az_location=eastus \
  github_client_id=<github-client-id> \
  github_client_secret=<github-client-secret> \
  microsoft_client_id=<microsoft-client-id> \
  microsoft_client_secret=<microsoft-client-secret>
```

Notes:

- Choose a unique suffix (letters only, max 15 chars)
- OAuth parameters are optional but recommended for authentication
- **Important**: At least one OAuth provider (GitHub or Microsoft) must be configured for the system to work properly
- If using Microsoft single tenant work accounts, you'll need to add the `MICROSOFT_TENANT_ID` parameter after deployment
- Your server URL will be: https://codepush-{suffix}.azurewebsites.net

### Step 5: Manual Deployment to Azure Web App

After creating the infrastructure, you can deploy your application manually:

1. Build your application: `npm run build`
2. Zip the contents: `zip -r release.zip ./*`
3. Deploy using the Azure CLI:

```bash
az webapp deployment source config-zip --resource-group <resource-group-name> --name codepush-<suffix> --src release.zip
```

## GitHub Actions Workflows

You can set up GitHub Actions workflows for automated deployment and security scanning of your CodePush server:

### Automated Deployment

A deployment workflow can automatically deploy your CodePush server to Azure when changes are pushed to your repository:

1. **Typical Components**:
   - Triggers on pushes to a specific branch or manual workflow dispatch
   - Build job that checks out code, sets up Node.js, installs dependencies, builds, and tests
   - Deploy job that deploys built artifacts to Azure Web App

To set up automated deployment:

1. Create a workflow file in `.github/workflows/` directory of your repository
2. Create an Azure Web App publish profile
3. Add the publish profile as a GitHub secret
4. Configure the workflow to use your Azure Web App name and resource group

### Security Scanning

You can also set up a security scanning workflow that performs analysis on your codebase:

1. **Typical Components**:
   - Runs on pushes to main branch, pull requests, or on a schedule
   - Sets up and runs code analysis tools like CodeQL
   - Reports security findings

These workflows will help automate your deployment process and ensure code security.

## Security Considerations

1. The default Azure Blob Storage is accessible to all subscription users - consider adjusting access settings
2. Always use HTTPS in production (enabled by default on Azure App Service)
3. Protect your OAuth credentials
4. Run security scans regularly to check for vulnerabilities

## Metrics

If you want to monitor release activity via the CLI:

1. Redis is required for metrics functionality
2. Follow the [official Redis installation guide](https://redis.io/docs/latest/operate/oss_and_stack/install/install-redis/)
3. TLS is required for Redis - follow the [official Redis TLS setup guide](https://redis.io/docs/latest/operate/oss_and_stack/management/security/encryption/#running-manually)
4. Set the necessary environment variables for Redis in your configuration

## Troubleshooting

### Common Issues:

1. **Deployment Fails**: Check that your project suffix meets naming requirements (letters only, max 15 chars)
2. **Authentication Issues**: Verify OAuth callback URLs match your server URL exactly
3. **Storage Errors**: Ensure Azurite is running for local development, or check storage account access keys
4. **Node.js Version**: Make sure you're using Node.js 18 LTS as specified

For more detailed troubleshooting, check the logs in your Azure Web App or local development environment.

## Configuring React Native Apps to Use Your Server

To make your React Native apps use your CodePush server:

### Android

In `strings.xml`, add:

```xml
<string moduleConfig="true" name="CodePushServerUrl">https://codepush-YOUR-SUFFIX.azurewebsites.net</string>
```

### iOS

In `Info.plist`, add:

```xml
<key>CodePushServerURL</key>
<string>https://codepush-YOUR-SUFFIX.azurewebsites.net</string>
```

## Quick Start: Step by Step Commands

Here's a complete step-by-step command guide to set up and deploy CodePush Server:

### Local Development Setup

```bash
# 1. Clone the repository
git clone <repository-url>
cd ml-code-push-server

# 2. Navigate to the API directory
cd api

# 3. Set up environment variables
cp .env.example .env

# 4. Install dependencies
npm install

# 5. Install Azurite for local storage emulation
npm install -g azurite

# 6. Start Azurite in a separate terminal
azurite --silent --location azurite --debug azurite/debug.log

# 7. Build the CodePush server
npm run build

# 8. Start the server with environment variables
npm run start:env
```

### Azure Deployment

```bash
# 1. Login to Azure
az login

# 2. Set your subscription
az account set --subscription "<subscription-id>"

# 3. Create a resource group
az group create --name "codepush-rg" --location "eastus"

# 4. Deploy the infrastructure
az deployment group create \
  --resource-group "codepush-rg" \
  --template-file ./codepush-infrastructure.bicep \
  --parameters project_suffix="<your-suffix>" \
  az_location="eastus" \
  github_client_id="<github-client-id>" \
  github_client_secret="<github-client-secret>" \
  microsoft_client_id="<microsoft-client-id>" \
  microsoft_client_secret="<microsoft-client-secret>"

# 5. Build the application for deployment
npm run build

# 6. Create a deployment package
zip -r release.zip ./*

# 7. Deploy the package to the Azure Web App
az webapp deployment source config-zip \
  --resource-group "codepush-rg" \
  --name "codepush-<your-suffix>" \
  --src release.zip
```

### Using the CLI with Your CodePush Server

```bash
# Install the CodePush CLI
npm install -g code-push-cli

# Login to your CodePush server
code-push login --serverUrl https://codepush-<your-suffix>.azurewebsites.net

# Register your app
code-push app add <app-name> <platform>

# Deploy an update
code-push release-react <app-name> <platform> --development
```

Remember to replace placeholder values (`<your-suffix>`, `<subscription-id>`, etc.) with your actual values. Also, make sure you have set up at least one OAuth provider (GitHub or Microsoft) before deployment.
