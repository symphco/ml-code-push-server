# Testing Your CodePush Server and CLI Setup

This guide provides practical steps to test your CodePush server and CLI setup to ensure everything is working correctly. It follows a progressive testing approach, starting from basic connectivity and advancing to complete end-to-end testing with a React Native app.

## Prerequisites

- Your CodePush server is running (either locally or on Azure)
- You have the CodePush CLI installed: `npm install -g code-push-cli`
- Microsoft Entra ID applications are configured for authentication

## 1. Basic Connectivity Tests

### 1.1. Server Availability

First, let's verify your server is accessible:

```bash
# For production environment
curl https://codepush-prodml.azurewebsites.net/

# For development environment
curl https://codepush-devml.azurewebsites.net/
```

You should see a response indicating the server is running. If not, check that your server is deployed correctly.

### 1.2. CLI Authentication

Test authentication with the CLI:

```bash
# For production
code-push login --serverUrl https://codepush-prodml.azurewebsites.net

# For development
code-push login --serverUrl https://codepush-devml.azurewebsites.net
```

This should open a browser for authentication with Microsoft Entra ID. After authenticating, you'll be asked to paste the access token into the CLI.

### 1.3. Verify Authentication

Check that you're properly authenticated:

```bash
code-push whoami
```

This should display your email address and indicate which authentication provider you used.

## 2. App Management Tests

### 2.1. Create Test Apps

Create test applications for each platform:

```bash
# Create a test iOS app
code-push app add TestApp-iOS ios

# Create a test Android app
code-push app add TestApp-Android android
```

### 2.2. List Apps

Verify your apps were created:

```bash
code-push app ls
```

You should see the apps you just created, along with their deployment keys.

### 2.3. Inspect Deployments

Check the deployments for one of your apps:

```bash
code-push deployment ls TestApp-iOS
```

You should see "Staging" and "Production" deployments automatically created for your app.

### 2.4. Create a Custom Deployment

Test creating a custom deployment:

```bash
code-push deployment add TestApp-iOS Development
```

Then verify it was created:

```bash
code-push deployment ls TestApp-iOS
```

## 3. Release Management Tests

### 3.1. Create a Dummy Release Package

Create a simple file to use as a test update:

```bash
mkdir -p TestRelease
echo '{"version": "1.0.0", "test": true}' > TestRelease/app.json
```

### 3.2. Release an Update

Release this package to your test app:

```bash
code-push release TestApp-iOS ./TestRelease 1.0.0 --description "Test release"
```

### 3.3. View Release History

Check that your release is in the deployment history:

```bash
code-push deployment history TestApp-iOS Staging
```

You should see your test release in the list.

### 3.4. Promote an Update

Test the promotion workflow:

```bash
code-push promote TestApp-iOS Staging Production --description "Promoted test release"
```

### 3.5. Verify the Promotion

Verify that the release exists in both deployments:

```bash
code-push deployment history TestApp-iOS Production
```

### 3.6. Rollback Test

Test the rollback functionality (assuming you have multiple releases):

```bash
# Release a second update
echo '{"version": "1.0.1", "test": true}' > TestRelease/app.json
code-push release TestApp-iOS ./TestRelease 1.0.0 --description "Second test release"

# Rollback to the previous release
code-push rollback TestApp-iOS Staging
```

Verify the rollback worked:

```bash
code-push deployment history TestApp-iOS Staging
```

## 4. Full End-to-End Testing with a React Native App

For complete testing, you'll need a React Native app with the CodePush SDK integrated.

### 4.1. Create a Simple React Native App

```bash
npx react-native init CodePushTestApp
cd CodePushTestApp
```

### 4.2. Add the CodePush SDK

```bash
npm install react-native-code-push --save
```

### 4.3. Configure the SDK to Use Your Server

#### For iOS (in `ios/CodePushTestApp/Info.plist`):

Add within the `<dict>` tags:

```xml
<key>CodePushServerURL</key>
<string>https://codepush-prodml.azurewebsites.net</string>
```

#### For Android (in `android/app/src/main/res/values/strings.xml`):

Add:

```xml
<string moduleConfig="true" name="CodePushServerUrl">https://codepush-prodml.azurewebsites.net</string>
```

### 4.4. Update Your App Code

Modify `App.js` to integrate CodePush:

```javascript
import React from "react";
import { View, Text, Button, Alert } from "react-native";
import codePush from "react-native-code-push";

const codePushOptions = {
  checkFrequency: codePush.CheckFrequency.ON_APP_START,
  installMode: codePush.InstallMode.IMMEDIATE,
};

function App() {
  const checkForUpdates = () => {
    codePush.sync(
      {
        updateDialog: true,
        installMode: codePush.InstallMode.IMMEDIATE,
      },
      (status) => {
        switch (status) {
          case codePush.SyncStatus.CHECKING_FOR_UPDATE:
            Alert.alert("Checking for updates");
            break;
          case codePush.SyncStatus.DOWNLOADING_PACKAGE:
            Alert.alert("Downloading package");
            break;
          case codePush.SyncStatus.INSTALLING_UPDATE:
            Alert.alert("Installing update");
            break;
          case codePush.SyncStatus.UP_TO_DATE:
            Alert.alert("App is up to date");
            break;
          case codePush.SyncStatus.UPDATE_INSTALLED:
            Alert.alert("Update installed");
            break;
          case codePush.SyncStatus.UNKNOWN_ERROR:
            Alert.alert("An unknown error occurred");
            break;
        }
      }
    );
  };

  return (
    <View style={{ flex: 1, justifyContent: "center", alignItems: "center" }}>
      <Text>CodePush Test App - Version 1</Text>
      <Button title="Check for updates" onPress={checkForUpdates} />
    </View>
  );
}

export default codePush(codePushOptions)(App);
```

### 4.5. Test the Full Workflow

1. Build and run your app on a device or simulator
2. Make a change to your app (e.g., change "Version 1" to "Version 2" in App.js)
3. Release the update:

```bash
code-push release-react TestApp-iOS ios -d Staging --description "Updated to Version 2"
```

4. In the app, press "Check for updates"
5. Verify that the app downloads and applies the update

## 5. Cleanup (Optional)

When you're done testing, you can clean up:

```bash
# Remove test apps
code-push app rm TestApp-iOS
code-push app rm TestApp-Android

# Logout from the CLI
code-push logout
```

## Troubleshooting

If you encounter issues during testing:

1. **Authentication failures**: Check your Entra ID app configuration and ensure the redirect URIs are correct
2. **Server connection issues**: Verify the server URL and that the server is running
3. **Release errors**: Check the CLI output for specific error messages
4. **App update issues**: Use the debug command to view logs:
   ```bash
   code-push debug ios  # or android
   ```

For more detailed troubleshooting, refer to the server logs in Azure App Service or your local development environment.
