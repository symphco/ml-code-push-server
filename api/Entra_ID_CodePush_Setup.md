# Microsoft Entra ID Setup for CodePush Server

This document provides guidance on setting up and verifying Microsoft Entra ID (formerly Azure AD) applications for authentication with CodePush Server.

## Existing Applications

You have two Entra ID applications already created:

| Environment | Application Name       | Application (Client) ID              | Created On |
| ----------- | ---------------------- | ------------------------------------ | ---------- |
| Development | CodePush Server (Dev)  | 5370e15a-e4cf-4ed5-9b68-95a2b7e398a2 | 4/3/2025   |
| Production  | CodePush Server (Prod) | 99ffcaab-bcaf-4862-be3f-eeaf62a67bbc | 4/3/2025   |

## Key Configuration Steps

### 1. Verify Redirect URIs

Ensure each application has the correct redirect URI configured:

- **Development**: `https://codepush-devml.azurewebsites.net/auth/callback/azure-ad`
- **Production**: `https://codepush-prodml.azurewebsites.net/auth/callback/azure-ad`

To check and update:

1. Go to Azure Portal → Microsoft Entra ID → App registrations
2. Select your application
3. Navigate to "Authentication"
4. Verify the redirect URI is correctly set
5. Under "Implicit grant and hybrid flows", ensure "Access tokens" and "ID tokens" are checked

### 2. Client Secrets

Verify you have active client secrets for both applications:

1. Go to Azure Portal → Microsoft Entra ID → App registrations
2. Select your application
3. Navigate to "Certificates & secrets"
4. Check that you have a valid, non-expired client secret
5. If needed, create a new secret by clicking "New client secret"
   - Add a description and choose an expiration period
   - Save the generated secret value immediately (you won't be able to see it again)

### 3. Account Type Verification

Confirm both applications are configured for the appropriate account type:

1. Go to Azure Portal → Microsoft Entra ID → App registrations
2. Select your application
3. On the "Overview" page, check "Supported account types"
4. If using single tenant, note your tenant ID from the Overview page

### 4. Environment Variables in CodePush Web Apps

Ensure your CodePush servers have the correct environment variables:

#### Development:

1. Go to Azure Portal → App Services → codepush-devml
2. Navigate to Configuration → Application settings
3. Verify or add these settings:
   - `MICROSOFT_CLIENT_ID`: `5370e15a-e4cf-4ed5-9b68-95a2b7e398a2`
   - `MICROSOFT_CLIENT_SECRET`: [Your saved client secret value]
   - `MICROSOFT_TENANT_ID`: [Only needed for single tenant setups]

#### Production:

1. Go to Azure Portal → App Services → codepush-prodml
2. Navigate to Configuration → Application settings
3. Verify or add these settings:
   - `MICROSOFT_CLIENT_ID`: `99ffcaab-bcaf-4862-be3f-eeaf62a67bbc`
   - `MICROSOFT_CLIENT_SECRET`: [Your saved client secret value]
   - `MICROSOFT_TENANT_ID`: [Only needed for single tenant setups]

## Troubleshooting

If authentication is not working:

1. **Check logs in your CodePush server**: Look for errors related to authentication
2. **Verify redirect URIs**: Ensure they exactly match your CodePush server URLs
3. **Check client secrets**: Make sure they are not expired
4. **Confirm environment variables**: Ensure all required variables are set correctly
5. **Test authentication flow**: Try logging in and check for any error messages

## Creating New Applications (If Needed)

If you need to create new Entra ID applications:

1. Go to Azure Portal → Microsoft Entra ID → App registrations → New registration
2. Name your application (e.g., "CodePush Server")
3. Choose the appropriate account type
4. Set the redirect URI to your CodePush server callback URL
5. Click "Register"
6. Create a client secret as described above
7. Configure the necessary settings in your CodePush server

## Reference

For more information, refer to:

- [CodePush Server Documentation](https://github.com/microsoft/react-native-code-push)
- [Microsoft Entra ID Documentation](https://learn.microsoft.com/en-us/entra/identity-platform/)
