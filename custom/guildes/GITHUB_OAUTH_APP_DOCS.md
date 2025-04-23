# GitHub OAuth Setup for CodePush Server

> **NOTE:** This document replaces the previous Microsoft Entra ID setup instructions, as the project has been switched to use GitHub OAuth for authentication.

This document provides guidance on setting up and verifying GitHub OAuth applications for authentication with CodePush Server.

## GitHub OAuth Configuration

### 1. Create a GitHub OAuth App

1. Go to https://github.com/settings/developers
2. Click on "New OAuth App"
3. Fill in the required information:
   - **Application name**: "CodePush Server" (or your preferred name)
   - **Homepage URL**: Your CodePush server URL
     - For Azure: `https://codepush-<your-suffix>.azurewebsites.net`
     - For local development: `http://localhost:3000`
   - **Authorization callback URL**:
     - For Azure: `https://codepush-<your-suffix>.azurewebsites.net/auth/callback/github`
     - For local development: `http://localhost:3000/auth/callback/github`
4. Click "Register application"
5. After registration, you'll see your **Client ID**
6. Click "Generate a new client secret" to create a **Client Secret**
7. Save both the Client ID and Client Secret securely

### 2. Configure Environment Variables

#### For Azure Deployments:

1. Go to Azure Portal → App Services → codepush-{your-suffix}
2. Navigate to Configuration → Application settings
3. Add or verify these settings:
   - `GITHUB_CLIENT_ID`: Your GitHub OAuth App's Client ID
   - `GITHUB_CLIENT_SECRET`: Your GitHub OAuth App's Client Secret
   - Remove Microsoft OAuth settings if not needed:
     - `MICROSOFT_CLIENT_ID`
     - `MICROSOFT_CLIENT_SECRET`
     - `MICROSOFT_TENANT_ID`

#### For Local Development:

Create or update your `.env` file in the `api` directory:

```
# GitHub OAuth
GITHUB_CLIENT_ID=your_github_client_id
GITHUB_CLIENT_SECRET=your_github_client_secret
SERVER_URL=http://localhost:3000
CORS_ORIGIN=http://localhost:3000

# Remove or comment out Microsoft OAuth settings
# MICROSOFT_CLIENT_ID=
# MICROSOFT_CLIENT_SECRET=
# MICROSOFT_TENANT_ID=
```

### 3. First-Time User Registration

To use your CodePush server with GitHub authentication, you must follow this sequence:

1. First register your account using the CLI:

   ```bash
   code-push-standalone register https://codepush-<your-suffix>.azurewebsites.net
   ```

   This will open a browser window where you'll authenticate with GitHub and create your account in the CodePush system.

2. After registration, you'll be automatically logged in to the CLI. If you need to log in again later:

   ```bash
   code-push-standalone login https://codepush-<your-suffix>.azurewebsites.net
   ```

3. Once registered and logged in via the CLI, you can access the web interface at your CodePush server URL.

### 4. Troubleshooting

#### Common Issues:

1. **Redirect URI Mismatch Error**: If you see "The redirect_uri is not associated with this application", ensure your callback URL in GitHub exactly matches what your CodePush server is using.

2. **Account Not Found Error**: If you see "Account not found. Have you registered with the CLI?" after GitHub authentication, it means you need to register first using the CLI command mentioned in section 3.

3. **Authentication Failed**: Verify your GitHub OAuth credentials are correctly set in your environment variables.

4. **Login Session Issues**: If CLI login fails, try checking your session status with:
   ```bash
   code-push-standalone whoami
   ```
   If needed, log out and try again:
   ```bash
   code-push-standalone logout
   ```

For more detailed instructions, refer to the main documentation in FRIENDLY_DOCS.md.
