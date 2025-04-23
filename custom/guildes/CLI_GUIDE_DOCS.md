# CodePush CLI Guide

This guide explains how to use the local CodePush CLI with your self-hosted CodePush servers.

## Installation

To install and use the local CodePush CLI:

```bash
# Clone the CodePush Service repository
git clone <repository-url>

# Navigate to the project directory
cd ml-code-push-server

# Install dependencies
npm install

# Build the CLI
npm run build

# Install CLI globally
npm install -g
```

After installation, the CLI will be available as `code-push-standalone`.

## Configuration

Before using the CLI, you need to log in to your CodePush server:

```bash
# For development environment
code-push-standalone login https://codepush-devml.azurewebsites.net

# For production environment
code-push-standalone login https://codepush-prodml.azurewebsites.net
```

This will open a browser for authentication.

## Common Commands

### Account Management

```bash
# Register new account (if needed)
code-push-standalone register https://codepush-devml.azurewebsites.net

# Check current logged in account
code-push-standalone whoami

# Logout
code-push-standalone logout
```

### App Management

```bash
# List all apps
code-push-standalone app ls

# Add a new app (create separate apps for iOS and Android)
code-push-standalone app add MyApp-iOS
code-push-standalone app add MyApp-Android

# Rename an app
code-push-standalone app rename CurrentName NewName

# Remove an app
code-push-standalone app rm AppName
```

### Deployment Management

Every app automatically gets "Staging" and "Production" deployments, but you can add more:

```bash
# List deployments for an app
code-push-standalone deployment ls MyApp-iOS

# Add a deployment
code-push-standalone deployment add MyApp-iOS QA

# Rename a deployment
code-push-standalone deployment rename MyApp-iOS QA Beta

# Remove a deployment
code-push-standalone deployment rm MyApp-iOS QA
```

### Releasing Updates

#### Standard Release

```bash
# Release an update
code-push-standalone release MyApp-iOS ./updates 1.0.0
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
code-push-standalone release-react MyApp-iOS ios
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

Full command example:

```bash
# Comprehensive React Native release example with common options
code-push-standalone release-react MyAwesomeApp-iOS ios \
  --deploymentName Production \
  --description "Fixed login issues and improved performance" \
  --mandatory true \
  --targetBinaryVersion "~1.2.3" \
  --development false \
  --sourcemapOutput ./sourcemaps \
  --rollout 25 \
  --entryFile index.ios.js \
  --plistFile ./ios/MyAwesomeApp/Info.plist \
  --gradleFile ./android/app/build.gradle
```

This example:

- Releases to the "Production" deployment
- Includes descriptive release notes
- Makes the update mandatory for all users
- Targets app binary version 1.2.x (compatibility with semver range)
- Disables development mode for a minified production bundle
- Generates sourcemaps for debugging
- Rolls out to 25% of users initially
- Specifies custom entry file and build configuration files

### Managing Releases

```bash
# View release history
code-push-standalone deployment history MyApp-iOS Staging

# Promote a release from Staging to Production
code-push-standalone promote MyApp-iOS Staging Production

# Rollback a deployment to a previous release
code-push-standalone rollback MyApp-iOS Production
```

## Debugging

```bash
# View debug logs for a running app
code-push-standalone debug android
code-push-standalone debug ios
```

## Working with Collaborators

```bash
# List collaborators
code-push-standalone collaborator ls MyApp-iOS

# Add a collaborator
code-push-standalone collaborator add MyApp-iOS user@example.com

# Remove a collaborator
code-push-standalone collaborator rm MyApp-iOS user@example.com
```

## Additional Security with Code Signing

If using code signing:

```bash
# Release with a private key for signing
code-push-standalone release-react MyApp-iOS ios --privateKeyPath private.pem
```

## Example Workflow

A typical workflow with CodePush might look like:

1. Create your apps for each platform:

   ```bash
   code-push-standalone app add MyAwesomeApp-iOS
   code-push-standalone app add MyAwesomeApp-Android
   ```

2. Deploy an update to staging:

   ```bash
   code-push-standalone release-react MyAwesomeApp-iOS ios --deploymentName Staging
   ```

3. Test the update in the staging environment

4. Promote to production when ready:
   ```bash
   code-push-standalone promote MyAwesomeApp-iOS Staging Production
   ```

## Troubleshooting

- **Server errors**: Make sure your server URL is correct and server is running
- **Authentication issues**: Check your authentication configuration
- **Deployment errors**: Verify app name and deployment name
- **Release issues**: Check target binary version compatibility

For more information, refer to the [CLI documentation](../cli/README.md).
