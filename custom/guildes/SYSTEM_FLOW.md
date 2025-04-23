# Complete CodePush System Architecture & Workflow

## System Components Overview

CodePush consists of three main components:

1. **CodePush Server**: Backend service (Node.js) that manages app updates
2. **CodePush CLI**: Command-line tool for interacting with the server
3. **CodePush SDK**: Client library embedded in React Native apps

Let's explore how these components work together in detail:

## 1. Infrastructure and Storage

### Azure Storage

CodePush uses Azure's cloud storage services to manage both the actual update files and their metadata:

- **Blob Storage**:

  - Stores the actual update packages (JavaScript bundles, images, and assets)
  - Handles large binary data efficiently
  - Each update package has a unique blob ID
  - Also stores package history as JSON (limited to 50 most recent releases per deployment)

- **Table Storage**:

  - NoSQL database that stores structured metadata using a hierarchical key system
  - Uses partition/row keys for efficient lookups
  - Key tables include:
    - **Accounts table**: User information (id, email, name, authentication details)
    - **Apps table**: App records (id, name, owner, collaborators)
    - **Deployments table**: Environment configurations (id, name, key, latest package reference)
    - **Packages table**: Release metadata (version, hash, description, rollout settings)
    - **Access Keys table**: CLI authentication tokens

- **Storage Relationships**:
  - Each Account can have multiple Apps
  - Each App can have multiple Deployments
  - Each Deployment can have multiple Packages (as history)
  - Complex relationships encoded in hierarchical keys (e.g., "appId_123_deploymentId\*\_456")
  - Shortcut keys for quick access (by email, deployment key, etc.)

### Redis (Optional)

- Used only for metrics functionality
- Tracks download statistics and active deployments
- Enables analytics on update adoption rates

## 2. Authentication Flow

### OAuth App Registration (One-time Setup)

1. Developer creates OAuth app in GitHub/Microsoft
2. Developer configures CodePush server with OAuth credentials:
   - `GITHUB_CLIENT_ID` and `GITHUB_CLIENT_SECRET`, or
   - `MICROSOFT_CLIENT_ID` and `MICROSOFT_CLIENT_SECRET`

### User Registration & Authentication

1. New user runs: `code-push-standalone register https://your-server-url`
2. CLI makes API call to `/auth/login` endpoint
3. Server generates unique state token and authorization URL
4. Browser opens to OAuth provider (GitHub/Microsoft)
5. User authenticates with OAuth provider
6. Provider redirects to callback URL with authorization code
7. Server exchanges code for OAuth access token via provider's API
8. Server creates user record in Azure Table Storage
9. Server generates CodePush-specific JWT access token
10. Browser sends token back to CLI via local callback server
11. CLI stores token in local config file (~/.code-push.json)

### Subsequent CLI Authentication

1. User runs: `code-push-standalone login https://your-server-url`
2. Same OAuth flow occurs but user record already exists
3. All subsequent CLI commands include JWT token in API requests

## 3. App & Deployment Management

### Creating Apps

1. User runs: `code-push-standalone app add MyApp-iOS ios`
2. CLI makes POST request to `/apps` endpoint with app name and platform
3. Server:
   - Generates unique app identifier
   - Creates app record in Azure Table Storage
   - Creates default deployments (Staging/Production)
   - Generates deployment keys for each environment

### Managing Deployments

1. User runs: `code-push-standalone deployment add MyApp-iOS QA`
2. CLI makes POST request to `/apps/MyApp-iOS/deployments`
3. Server:
   - Creates new deployment in Azure Table Storage
   - Generates unique deployment key
   - Associates deployment with parent app

## 4. Release Management

### Publishing an Update

1. Developer makes code changes to React Native app
2. Developer runs: `code-push-standalone release-react MyApp-iOS ios`
3. CLI:

   - Bundles app's JavaScript and assets (using Metro bundler)
   - Creates zip package with manifest.json
   - Makes POST request to `/apps/MyApp-iOS/deployments/Staging/releases`
   - Uploads package data in multipart/form-data request

4. Server:
   - Validates request and user permissions
   - Generates unique release identifier
   - Uploads package to Azure Blob Storage
   - Creates release record in Table Storage with:
     - Target binary version
     - Release notes
     - Package hash
     - Upload time
     - Mandatory flag
     - Rollout percentage
   - Updates deployment's latest release reference

### Promoting Updates

1. Developer runs: `code-push-standalone promote MyApp-iOS Staging Production`
2. CLI makes POST request to `/apps/MyApp-iOS/deployments/Staging/promote` with target
3. Server:
   - Validates request
   - Copies release metadata from Staging to Production
   - Updates Production deployment's latest release reference
   - Re-uses same package blob (no duplication)

### Rollback

1. Developer runs: `code-push-standalone rollback MyApp-iOS Production`
2. CLI makes POST request to `/apps/MyApp-iOS/deployments/Production/rollback`
3. Server:
   - Finds previous release
   - Updates deployment to point to that release

## 5. Client Update Process

### SDK Integration

1. Developer adds react-native-code-push to app
2. Configures deployment keys in:
   - iOS: Info.plist
   - Android: strings.xml
3. Configures custom server URL:
   - iOS: `CodePushServerURL` in Info.plist
   - Android: `CodePushServerUrl` in strings.xml
4. Wraps app's root component with CodePush HOC

### Update Check Flow

1. React Native app launches or triggers check
2. CodePush SDK:

   - Retrieves deployment key and server URL from app config
   - Makes GET request to `/deployments/{deploymentKey}/updates/check` with:
     - App version
     - Package hash (if any previous update)
     - Locale information

3. Server:

   - Finds deployment by key
   - Checks if update is available based on:
     - Target binary version compatibility
     - Package hash comparison
     - Rollout percentage
   - Returns update metadata if available

4. CodePush SDK:
   - If update available, makes GET request to download package from blob storage
   - Verifies package hash
   - Unzips package to local storage
   - Applies update based on configured install mode:
     - `IMMEDIATE`: Applies immediately
     - `ON_NEXT_RESTART`: Saves for next app launch
     - `ON_NEXT_RESUME`: Applies when app resumes from background

### Metrics Collection (if enabled)

1. SDK reports download success/failure
2. Server stores metrics in Redis
3. Metrics data accessible via CLI: `code-push-standalone deployment histogram`

## 6. Security Mechanisms

1. **JWT Authentication**: All API calls require valid JWT token
2. **API Permission Checks**: Server validates user permissions for each operation
3. **Package Hashing**: Updates verified by SHA256 hash
4. **Optional Code Signing**: Updates can be cryptographically signed and verified

## 7. Data Model and Entity Relationships

### Account

- **Purpose**: Represents a user of the CodePush system
- **Key Fields**:
  - `id`: Unique identifier
  - `email`: User's email (unique)
  - `name`: User's name
  - `createdTime`: When account was created
  - Authentication IDs: `gitHubId`, `microsoftId`, or `azureAdId`
- **Relationships**:
  - One Account can own many Apps
  - One Account can have many AccessKeys

### App

- **Purpose**: Represents a mobile application registered with CodePush
- **Key Fields**:
  - `id`: Unique identifier
  - `name`: App name
  - `createdTime`: When app was created
  - `collaborators`: Map of email addresses to collaborator properties
- **Relationships**:
  - Belongs to an Account (ownership)
  - One App can have many Deployments
  - One App can have many Collaborators

### Deployment

- **Purpose**: Represents an environment (Staging, Production, etc.)
- **Key Fields**:
  - `id`: Unique identifier
  - `name`: Deployment name (e.g., "Staging", "Production")
  - `key`: Unique deployment key used in app configuration
  - `createdTime`: When deployment was created
  - `package`: Optional reference to the latest release package
- **Relationships**:
  - Belongs to an App (parent-child)
  - One Deployment can have many Packages (release history)

### Understanding CodePush Deployments vs. Server Environments

It's important to understand the distinction between CodePush "deployments" and traditional server environments:

1. **Server Environments** (Dev/Staging/Production servers):

   - Separate physical or virtual infrastructure
   - Different URLs (e.g., `dev-server.com`, `staging-server.com`, `prod-server.com`)
   - Typically used for different stages of backend development

2. **CodePush Deployments** (Staging/Production/etc.):
   - Logical environments within a single CodePush server
   - Share the same server infrastructure and URL
   - Control how app updates are distributed to different user segments

#### Why CodePush Uses This Model

CodePush deployments provide a release pipeline for app updates, allowing you to:

- Test updates with internal users before general release
- Gradually roll out changes to production users
- Maintain separate channels for beta testers and regular users
- Quickly roll back problematic updates

#### How CodePush Deployments Work Together

A typical workflow using multiple deployments:

1. **Development**:

   - Developers make changes to the app's JavaScript and assets

2. **Staging Release**:

   - Updates are pushed to the "Staging" deployment
   - Internal testers verify the update using the same app binary but with a different deployment key
   - Bugs are found and fixed before reaching external users

3. **Production Release**:

   - Once verified, updates are promoted from Staging to Production
   - This copies the release metadata but keeps the same package blob
   - End users with the Production deployment key receive the update

4. **Emergency Rollback**:
   - If issues appear in Production, rollback to a previous release
   - No need to rebuild or resubmit the app to app stores

This multi-deployment model on a single CodePush server gives you the flexibility of multiple environments without the overhead of managing separate infrastructure for each release channel.

### Package

- **Purpose**: Represents a specific release/update
- **Key Fields**:
  - `appVersion`: Compatible app version
  - `blobUrl`: URL to the actual update package
  - `manifestBlobUrl`: URL to the manifest file
  - `packageHash`: Hash of the package content
  - `description`: Release notes
  - `isDisabled`: Whether the update is disabled
  - `isMandatory`: Whether the update is mandatory
  - `size`: Size of the package
  - `uploadTime`: When the package was uploaded
  - `label`: Auto-generated version
  - `releaseMethod`: How package was created (Upload, Promote, Rollback)
  - `originalDeployment`: Source deployment if promoted
  - `originalLabel`: Original label if promoted or rolled back
  - `rollout`: Percentage of users receiving the update
- **Relationships**:
  - Belongs to a Deployment (parent-child)
  - Referenced by Deployment's `package` field (latest release)

### Collaborator

- **Purpose**: Links an Account to an App with permissions
- **Structure**:
  ```
  collaborators: {
    "user@example.com": {
      accountId: "accountId",
      permission: "Owner" | "Collaborator",
      isCurrentAccount: boolean
    }
  }
  ```
- **Relationships**:
  - Stored as a map within the App object
  - Links an Account to an App with specific permissions

### AccessKey

- **Purpose**: Used for CLI authentication
- **Key Fields**:
  - `id`: Unique identifier
  - `name`: Key name
  - `friendlyName`: Human-readable name
  - `createdBy`: Email of creator
  - `createdTime`: When key was created
  - `expires`: Expiration timestamp
  - `isSession`: Whether it's a temporary session key
- **Relationships**:
  - Belongs to an Account (parent-child)

### Storage Implementation

- Azure Table Storage uses a hierarchical key structure to represent relationships
- Partition/row keys encode entity relationships (e.g., "appId_123_deploymentId\*\_456")
- Shortcut keys for efficient lookups (by email, deployment key, etc.)
- Package history stored as JSON in blob storage (limited to 50 most recent releases)

### Accessing Azure Table Storage in the Portal

To view and inspect your CodePush data in the Azure portal:

1. **Log in to Azure Portal**:

   - Open your browser and navigate to: https://portal.azure.com
   - Sign in with your Azure credentials
   - Note: If you can't click the link above, copy and paste the URL into your browser

2. **Find Your Storage Account**:

   - In the left menu, click on "Storage accounts"
   - Locate and select the storage account used by your CodePush server (typically named "codepushstorage{suffix}")

3. **Access Table Service**:

   - In the storage account menu, scroll down to the "Data storage" section
   - Click on "Tables"

4. **View Table Data**:

   - You'll see a table named "storagev2" (the main CodePush table)
   - Click on the table name to open it
   - Use the "Query" button to view table entities

5. **Running Queries**:

   - To view all records: Leave the query empty and click "Run"
   - To filter by a specific partition key:
     ```
     PartitionKey eq 'accountId 12345'
     ```
   - To find a specific app:
     ```
     PartitionKey eq 'appId 67890'
     ```
   - To view deployments for an app:
     ```
     PartitionKey eq 'appId 67890' and RowKey ge 'appId 67890 deploymentId'
     ```

6. **Understanding Table Structure**:

   - Each entity has a PartitionKey and RowKey that encode relationships
   - Entity data is stored as properties
   - Hierarchical keys use space and asterisk delimiters
   - The asterisk (\*) is used to mark leaf nodes in hierarchical keys

7. **Storage Explorer Alternative**:
   - For a more user-friendly experience, use Azure Storage Explorer
   - Download from: https://azure.microsoft.com/features/storage-explorer/
   - Copy and paste this URL into your browser to download
   - Connect to your storage account and navigate to Tables → storagev2

This direct access to Table Storage can be useful for debugging, manual data inspection, or recovering from corrupted states when needed.

## Complete End-to-End Example

1. **Initial Setup**:

   - Developer deploys CodePush server to Azure
   - Sets up GitHub OAuth app with callback URL
   - Configures server with OAuth credentials
   - Registers account: `code-push-standalone register https://my-codepush.azurewebsites.net`

2. **App Creation**:

   - Creates apps:
     ```
     code-push-standalone app add MyApp-iOS ios
     code-push-standalone app add MyApp-Android android
     ```
   - Server creates records in Azure Tables and generates deployment keys

3. **SDK Integration**:

   - Adds CodePush SDK to React Native app
   - Configures custom server URL and deployment keys
   - Wraps app with CodePush HOC

4. **Update Workflow**:

   - Makes changes to app code
   - Releases update to Staging:
     ```
     code-push-standalone release-react MyApp-iOS ios -d Staging
     ```
   - CLI bundles app and uploads to server
   - Server stores bundle in Blob Storage and metadata in Tables
   - Tests update on test devices
   - Promotes to Production:
     ```
     code-push-standalone promote MyApp-iOS Staging Production
     ```

5. **User Update Experience**:
   - End user opens app
   - SDK checks for updates from server
   - Server determines if update is available
   - SDK downloads, verifies, and applies update
   - App restarts with new code without App Store submission

This comprehensive workflow allows for rapid iteration and deployment of JavaScript and asset changes, bypassing the traditional app store review process while maintaining a robust, secure update mechanism.
