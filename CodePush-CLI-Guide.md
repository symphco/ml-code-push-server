# CodePush CLI Guide

This guide explains how to use the CodePush CLI with your Azure-hosted CodePush servers.

## Installation

To install the CodePush CLI:

```bash
npm install -g code-push-cli
```

## Configuration

Before using the CLI, you need to log in to your CodePush server:

```bash
# For development environment
code-push login --serverUrl https://codepush-devml.azurewebsites.net

# For production environment
code-push login --serverUrl https://codepush-prodml.azurewebsites.net
```

This will open a browser for authentication through Microsoft Entra ID.

## Common Commands

### Account Management

```bash
# Register new account (if needed)
code-push register --serverUrl https://codepush-devml.azurewebsites.net

# Check current logged in account
code-push whoami

# Logout
code-push logout
```

### App Management

```bash
# List all apps
code-push app ls

# Add a new app (create separate apps for iOS and Android)
code-push app add MyApp-iOS
code-push app add MyApp-Android

# Rename an app
code-push app rename CurrentName NewName

# Remove an app
code-push app rm AppName
```

### Deployment Management

Every app automatically gets "Staging" and "Production" deployments, but you can add more:

```bash
# List deployments for an app
code-push deployment ls MyApp-iOS

# Add a deployment
code-push deployment add MyApp-iOS QA

# Rename a deployment
code-push deployment rename MyApp-iOS QA Beta

# Remove a deployment
code-push deployment rm MyApp-iOS QA
```

### Releasing Updates

#### Standard Release

```bash
# Release an update
code-push release MyApp-iOS ./updates 1.0.0
```

Parameters:

- **App name**: Name of your app
- **Update contents**: Path to the update files or folder
- **Target binary version**: App store version this is for (can be a semver range)

Optional flags:

- `--deploymentName` or `-d`: Target deployment (defaults to Staging)
- `--description` or `-des`: Release notes
- `--mandatory` or `-m`: Force users to update
- `--rollout` or `-r`: Percentage of users to deploy to (e.g., "25%")
- `--disabled`: Prevent the update from being downloaded

#### React Native Release

```bash
# Release a React Native update
code-push release-react MyApp-iOS ios
```

Parameters:

- **App name**: Name of your app
- **Platform**: "ios" or "android"

Common options:

- `--deploymentName` or `-d`: Target deployment (defaults to Staging)
- `--description` or `-des`: Release notes
- `--mandatory` or `-m`: Force users to update
- `--targetBinaryVersion` or `-t`: App store version this is for
- `--development` or `--dev`: Generate an unminified bundle with warnings

### Managing Releases

```bash
# View release history
code-push deployment history MyApp-iOS Staging

# Promote a release from Staging to Production
code-push promote MyApp-iOS Staging Production

# Rollback a deployment to a previous release
code-push rollback MyApp-iOS Production
```

## Debugging

```bash
# View debug logs for a running app
code-push debug android
code-push debug ios
```

## Working with Collaborators

```bash
# List collaborators
code-push collaborator ls MyApp-iOS

# Add a collaborator
code-push collaborator add MyApp-iOS user@example.com

# Remove a collaborator
code-push collaborator rm MyApp-iOS user@example.com
```

## Additional Security with Code Signing

If using code signing:

```bash
# Release with a private key for signing
code-push release-react MyApp-iOS ios --privateKeyPath private.pem
```

## Example Workflow

A typical workflow with CodePush might look like:

1. Create your apps for each platform:

   ```bash
   code-push app add MyAwesomeApp-iOS
   code-push app add MyAwesomeApp-Android
   ```

2. Deploy an update to staging:

   ```bash
   code-push release-react MyAwesomeApp-iOS ios --deploymentName Staging
   ```

3. Test the update in the staging environment

4. Promote to production when ready:
   ```bash
   code-push promote MyAwesomeApp-iOS Staging Production
   ```

## Troubleshooting

- **Server errors**: Make sure your server URL is correct and server is running
- **Authentication issues**: Check your Entra ID configuration
- **Deployment errors**: Verify app name and deployment name
- **Release issues**: Check target binary version compatibility

For more information, refer to the [CLI documentation](../cli/README.md).
