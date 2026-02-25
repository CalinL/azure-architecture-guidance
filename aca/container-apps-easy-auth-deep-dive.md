# Azure Container Apps Easy Auth: Complete Architecture and Token Store Deep Dive

## Table of Contents

- [Overview](#overview)
- [Feature Architecture](#feature-architecture)
- [Authentication Flow Step by Step](#authentication-flow-step-by-step)
- [What Your Application Receives](#what-your-application-receives)
- [Token Store](#token-store)
- [Token Store with SAS URL (Stable API)](#token-store-with-sas-url-stable-api)
- [Token Store with Managed Identity (Preview API)](#token-store-with-managed-identity-preview-api)
- [Token Expiration Reference](#token-expiration-reference)
- [Managed Identity vs Easy Auth](#managed-identity-vs-easy-auth)
- [Decision Guide](#decision-guide)
- [Security Best Practices](#security-best-practices)
- [References](#references)

---

## Overview

Azure Container Apps provides built-in authentication and authorization features, commonly referred to as "Easy Auth." This platform-level capability secures external ingress-enabled container apps with minimal or no application code. Container Apps uses the same authentication and authorization system as Azure App Service, with some implementation differences documented in this guide.

Easy Auth supports multiple identity providers out of the box:

| Provider                    | Sign-in Endpoint             |
|-----------------------------|------------------------------|
| Microsoft Entra ID          | `/.auth/login/aad`           |
| Facebook                    | `/.auth/login/facebook`      |
| GitHub                      | `/.auth/login/github`        |
| Google                      | `/.auth/login/google`        |
| X (Twitter)                 | `/.auth/login/x`             |
| Custom OpenID Connect       | `/.auth/login/<providerName>`|

> [!IMPORTANT]
> Easy Auth requires HTTPS. Ensure `allowInsecure` is disabled on your container app's ingress configuration.

---

## Feature Architecture

When you enable Easy Auth, Azure Container Apps deploys an **authentication sidecar container** alongside your application container on every replica. This sidecar follows the [Ambassador pattern](https://learn.microsoft.com/azure/architecture/patterns/ambassador) and intercepts all incoming HTTP requests before they reach your application.

```text
Internet --> Ingress --> [ Auth Sidecar Container ] --> [ Your App Container ]
                         |                        |
                         | Uses:                  |
                         | - App Registration      |
                         |   (Client ID + Secret)  |
                         | - Identity Provider      |
                         |   configuration          |
                         |                          |
                         | Performs:                 |
                         | - OAuth redirects         |
                         | - Token validation        |
                         | - Session management      |
                         | - Header injection        |
```

The sidecar runs in a separate container, isolated from your application code. Because it does not run in-process, no direct integration with specific language frameworks is possible. Instead, the sidecar injects identity information into HTTP request headers that your application reads.

The sidecar handles:

1. Authenticating users and clients with the configured identity providers.
2. Validating OAuth tokens received from the identity provider.
3. Managing the authenticated session (issuing and validating session cookies).
4. Injecting identity information into HTTP request headers forwarded to your app.

### Core Requirement: App Registration

To use Easy Auth with Microsoft Entra ID, you create an **app registration** in Entra ID. This provides:

| Component     | Purpose                                                                                       |
|---------------|-----------------------------------------------------------------------------------------------|
| Client ID     | Identifies your container app to the identity provider during OAuth flows.                     |
| Client Secret | Authenticates your container app when exchanging authorization codes for tokens with Entra ID. |
| Redirect URI  | The callback URL (`/.auth/login/aad/callback`) where Entra ID sends users after sign-in.      |

The app registration and client secret are what make the OAuth flow work. They are the core configuration of Easy Auth.

---

## Authentication Flow Step by Step

Easy Auth supports two flows: **server-directed** (browser apps) and **client-directed** (API/mobile apps). The server-directed flow is most common and works as follows:

### Server-Directed Flow (Browser)

```text
1. Browser --> GET https://myapp.azurecontainerapps.io/dashboard

2. Auth Sidecar intercepts: no session cookie found, user is unauthenticated

3. Sidecar redirects browser:
   --> https://login.microsoftonline.com/{tenant}/oauth2/v2.0/authorize
       ?client_id=<CLIENT_ID>
       &redirect_uri=https://myapp.../.auth/login/aad/callback
       &scope=openid profile email [offline_access]
       &response_type=code

4. User signs in at Microsoft's login page

5. Entra ID redirects browser:
   --> https://myapp.../.auth/login/aad/callback?code=AUTHORIZATION_CODE

6. Sidecar receives the auth code and exchanges it with Entra ID:
   POST https://login.microsoftonline.com/{tenant}/oauth2/v2.0/token
   Body: client_id + client_secret + authorization_code

   Entra ID returns: { id_token, access_token, refresh_token }

7. Sidecar validates the tokens

8. IF token store is enabled:
     Sidecar writes provider tokens to Blob Storage

9. Sidecar creates an encrypted session cookie (AppServiceAuthSession)
   and redirects browser back to /dashboard

10. Browser --> GET /dashboard (with session cookie)

11. Sidecar validates session cookie, extracts user claims,
    injects headers, and forwards request to your app container

12. Your app reads the headers and serves the page
```

### Client-Directed Flow (API/Mobile)

In this flow, the client application signs in the user directly using the provider's SDK, then submits the resulting token to the sidecar for validation:

```text
POST https://myapp.azurecontainerapps.io/.auth/login/aad
Content-Type: application/json

{"id_token":"<token>","access_token":"<token>"}
```

If validation succeeds, the API returns a session token:

```json
{
    "authenticationToken": "...",
    "user": {
        "userId": "sid:..."
    }
}
```

Subsequent requests include this token in the `X-ZUMO-AUTH` header:

```text
GET https://myapp.azurecontainerapps.io/api/products/1
X-ZUMO-AUTH: <authenticationToken_value>
```

### Authorization Behavior

You configure how the sidecar handles unauthenticated requests:

| Setting                        | Behavior                                                                |
|--------------------------------|-------------------------------------------------------------------------|
| Allow unauthenticated access   | Passes requests through; your app decides what to do with anonymous users. |
| Require authentication         | Rejects unauthenticated requests with redirect (302), 401, or 403.     |

---

## What Your Application Receives

On every authenticated request, the sidecar injects identity claims as HTTP headers. Your application does not need any authentication SDK or library to read these.

### Always Available (No Token Store Required)

| Header                        | Contains                                        |
|-------------------------------|-------------------------------------------------|
| `X-MS-CLIENT-PRINCIPAL-NAME`  | User's display name or email.                   |
| `X-MS-CLIENT-PRINCIPAL-ID`    | User's object ID from the identity provider.    |
| `X-MS-CLIENT-PRINCIPAL`       | Base64-encoded JSON with all claims.            |

These headers are sufficient for most applications that only need to identify who the user is, gate access to authenticated users, or read claims for role-based decisions.

### Available Only with Token Store Enabled

| Header                            | Contains                                                |
|-----------------------------------|---------------------------------------------------------|
| `X-MS-TOKEN-AAD-ID-TOKEN`        | The Entra ID token containing user identity claims.     |
| `X-MS-TOKEN-AAD-ACCESS-TOKEN`    | The OAuth access token for calling downstream APIs.     |
| `X-MS-TOKEN-AAD-REFRESH-TOKEN`   | The refresh token for obtaining new access tokens.      |
| `X-MS-TOKEN-AAD-EXPIRES-ON`      | The expiration timestamp of the access token.           |

You can also retrieve tokens by calling `GET /.auth/me` from your application code.

> [!NOTE]
> The `/.auth/me` endpoint and provider-specific token headers require the token store to be enabled. Without the token store, you receive only user identity claims in the `X-MS-CLIENT-PRINCIPAL-*` headers.

---

## Token Store

The token store is an **optional, preview feature** that provides persistent storage for the OAuth tokens (ID tokens, access tokens, and refresh tokens) that the identity provider issues during the authentication flow.

### Why Does the Token Store Exist?

Most applications only need to know who the user is. The `X-MS-CLIENT-PRINCIPAL-*` headers provide this without any token store configuration.

However, some applications need the **raw provider tokens** to call downstream APIs on behalf of the user:

* Calling Microsoft Graph API to read the user's calendar or email.
* Posting to the user's Facebook timeline.
* Accessing Google Drive files belonging to the user.

For these scenarios, you need the provider's access token, not the user's identity claims alone.

### Why Blob Storage?

On Azure App Service, the token store is built in. App Service runs on persistent VMs with a file system where the platform stores tokens automatically.

Azure Container Apps is different. Containers are ephemeral: they scale to zero, get destroyed, and recreate on different nodes. There is **no persistent local file system** across replicas. The auth sidecar needs an external, persistent location to store and retrieve provider tokens across container restarts and scale events.

Azure Blob Storage serves as that external persistent store.

### Token Store Architecture

```text
                                    +-------------------------+
                                    |   Azure Blob Storage    |
                                    |   (Token Store)         |
                                    |   - ID tokens           |
                                    |   - Access tokens       |
                                    |   - Refresh tokens      |
                                    +------------^------------+
                                                 |
                                                 | Connection via
                                                 | SAS URL or Managed Identity
                                                 |
Internet --> Ingress --> [ Auth Sidecar ] --------+---> [ Your App Container ]
                         Uses: Client ID                Gets headers:
                         + Client Secret                X-MS-TOKEN-AAD-ACCESS-TOKEN
                         for OAuth flow                 X-MS-TOKEN-AAD-REFRESH-TOKEN
                                                        X-MS-TOKEN-AAD-ID-TOKEN
```

Only the associated user of each authenticated session can access their cached tokens.

---

## Token Store with SAS URL (Stable API)

This is the only method available in the **current stable API** (`2025-01-01` and earlier).

### How It Works

You generate a SAS (Shared Access Signature) URL for a private Blob Storage container. The SAS URL provides time-limited, permission-scoped access to the blob container without requiring the sidecar to authenticate via Entra ID.

### Setup Steps

1. Create an Azure Storage account with a **private** blob container.
2. Generate a SAS URL for the container with **read**, **write**, and **delete** permissions.
3. Save the SAS URL as a **secret** in your container app.
4. Enable the token store and associate it with the secret.

```bash
az containerapp auth update \
  --resource-group <RESOURCE_GROUP_NAME> \
  --name <CONTAINER_APP_NAME> \
  --sas-url-secret-name <SAS_SECRET_NAME> \
  --token-store true
```

### Bicep (Stable API)

```bicep
resource authConfig 'Microsoft.App/containerApps/authConfigs@2025-01-01' = {
  parent: containerApp
  name: 'current'
  properties: {
    platform: {
      enabled: true
    }
    identityProviders: {
      azureActiveDirectory: {
        enabled: true
        registration: {
          clientId: '<APP_REGISTRATION_CLIENT_ID>'
          clientSecretSettingName: '<SECRET_NAME>'
          openIdIssuer: 'https://login.microsoftonline.com/<TENANT_ID>/v2.0'
        }
      }
    }
    login: {
      tokenStore: {
        enabled: true
        azureBlobStorage: {
          sasUrlSettingName: '<SAS_SECRET_NAME>'
        }
      }
    }
  }
}
```

### ARM Template Schema (Stable)

In the `2025-01-01` stable API, the `BlobStorageTokenStore` type has only one property and it is **required**:

```json
"BlobStorageTokenStore": {
    "required": ["sasUrlSettingName"],
    "properties": {
        "sasUrlSettingName": {
            "description": "The name of the app secrets containing the SAS URL of the blob storage containing the tokens.",
            "type": "string"
        }
    }
}
```

> [!WARNING]
> The SAS URL has an expiration date that you set when generating it. If the SAS expires, the token store stops working. You must track SAS expiration dates and rotate the SAS URL before it expires, updating the Container Apps secret accordingly.

### SAS URL Expiration Workarounds

* Set a long SAS expiration (for example, 2 to 5 years).
* Use an Azure Automation runbook or Azure Function on a schedule to regenerate the SAS and update the Container Apps secret before expiration.
* Set calendar reminders to rotate manually if automation is not feasible.

---

## Token Store with Managed Identity (Preview API)

Starting with the `2025-02-02-preview` API version, the `BlobStorageTokenStore` type supports **Managed Identity** as an alternative to SAS URLs.

> [!CAUTION]
> This capability is available only in the **preview** API (`2025-02-02-preview`). Preview APIs can have breaking changes before reaching general availability. Use in production at your own risk and evaluate your organization's tolerance for preview features.

### How It Works

Instead of a SAS URL, you provide:

* The **blob container URI** (plain HTTPS URL, no SAS token).
* A **User-Assigned Managed Identity** reference (either `clientId` or `managedIdentityResourceId`).

The auth sidecar uses the managed identity to authenticate to Blob Storage via Entra ID and RBAC. No secrets to rotate, no expiration to track.

### ARM Template Schema (Preview)

In the `2025-02-02-preview` API, the `BlobStorageTokenStore` type looks like this:

```json
"BlobStorageTokenStore": {
    "properties": {
        "sasUrlSettingName": {
            "description": "...Should not be used along with blobContainerUri.",
            "type": "string"
        },
        "blobContainerUri": {
            "description": "The URI of the blob storage containing the tokens. Should not be used along with sasUrlSettingName.",
            "type": "string"
        },
        "clientId": {
            "description": "The Client ID of a User-Assigned Managed Identity. Should not be used along with managedIdentityResourceId.",
            "type": "string"
        },
        "managedIdentityResourceId": {
            "description": "The Resource ID of a User-Assigned Managed Identity. Should not be used along with clientId.",
            "type": "string"
        }
    }
}
```

> [!IMPORTANT]
> The properties are mutually exclusive:
>
> * Use `sasUrlSettingName` **or** `blobContainerUri`, not both.
> * Use `clientId` **or** `managedIdentityResourceId`, not both.
> * `sasUrlSettingName` is no longer required in the preview schema, making the managed identity approach possible.

### Setup Steps

1. Create a **User-Assigned Managed Identity** and assign it to your Container App.
2. Assign the **Storage Blob Data Contributor** role to the managed identity, scoped to the blob container.
3. Configure the token store using `blobContainerUri` and either `managedIdentityResourceId` or `clientId`.

### Bicep (Preview API, Using managedIdentityResourceId)

```bicep
resource authConfig 'Microsoft.App/containerApps/authConfigs@2025-02-02-preview' = {
  parent: containerApp
  name: 'current'
  properties: {
    platform: {
      enabled: true
    }
    identityProviders: {
      azureActiveDirectory: {
        enabled: true
        registration: {
          clientId: '<APP_REGISTRATION_CLIENT_ID>'
          clientSecretSettingName: '<SECRET_NAME>'
          openIdIssuer: 'https://login.microsoftonline.com/<TENANT_ID>/v2.0'
        }
      }
    }
    login: {
      tokenStore: {
        enabled: true
        azureBlobStorage: {
          blobContainerUri: 'https://<STORAGE_ACCOUNT>.blob.core.windows.net/<CONTAINER>'
          managedIdentityResourceId: '/subscriptions/<SUB_ID>/resourceGroups/<RG>/providers/Microsoft.ManagedIdentity/userAssignedIdentities/<MI_NAME>'
        }
      }
    }
  }
}
```

### Bicep (Preview API, Using clientId)

```bicep
resource authConfig 'Microsoft.App/containerApps/authConfigs@2025-02-02-preview' = {
  parent: containerApp
  name: 'current'
  properties: {
    platform: {
      enabled: true
    }
    identityProviders: {
      azureActiveDirectory: {
        enabled: true
        registration: {
          clientId: '<APP_REGISTRATION_CLIENT_ID>'
          clientSecretSettingName: '<SECRET_NAME>'
          openIdIssuer: 'https://login.microsoftonline.com/<TENANT_ID>/v2.0'
        }
      }
    }
    login: {
      tokenStore: {
        enabled: true
        azureBlobStorage: {
          blobContainerUri: 'https://<STORAGE_ACCOUNT>.blob.core.windows.net/<CONTAINER>'
          clientId: '<USER_ASSIGNED_MI_CLIENT_ID>'
        }
      }
    }
  }
}
```

### Official Example Payloads

The Azure REST API specs repository includes two official example payloads for the managed identity approach:

* [AuthConfigs_BlobStorageTokenStore_CreateOrUpdate.json](https://github.com/Azure/azure-rest-api-specs/blob/main/specification/app/resource-manager/Microsoft.App/ContainerApps/preview/2025-02-02-preview/examples/AuthConfigs_BlobStorageTokenStore_CreateOrUpdate.json) uses `managedIdentityResourceId`.
* [AuthConfigs_BlobStorageTokenStore_ClientId_CreateOrUpdate.json](https://github.com/Azure/azure-rest-api-specs/blob/main/specification/app/resource-manager/Microsoft.App/ContainerApps/preview/2025-02-02-preview/examples/AuthConfigs_BlobStorageTokenStore_ClientId_CreateOrUpdate.json) uses `clientId`.

---

## Token Expiration Reference

### Session and Provider Tokens

| Token / Credential                  | Expires?  | Default Lifetime                             | Your Responsibility                                                                                  |
|-------------------------------------|-----------|----------------------------------------------|------------------------------------------------------------------------------------------------------|
| Easy Auth session cookie            | Yes       | 8 hours, with a 72-hour grace period.        | Call `GET /.auth/refresh` to extend within the grace window. After 72 hours, the user must re-sign in. |
| Provider access token (Entra ID)    | Yes       | Approximately 1 hour (provider-specific).    | Call `GET /.auth/refresh` to trigger automatic refresh via the stored refresh token.                  |
| Provider refresh token (Entra ID)   | Yes       | Configurable via Entra ID policy (days to months). | Configure the provider to issue them (add `offline_access` scope for Entra ID).                      |
| Facebook long-lived token           | Yes       | 60 days. No refresh tokens available.        | User must re-authenticate after expiration.                                                          |
| X (Twitter) access token            | No        | Does not expire.                             | No action needed.                                                                                    |
| Google access token                 | Yes       | Approximately 1 hour.                        | Append `access_type=offline` to get refresh tokens. Call `/.auth/refresh` to refresh.                |
| App Registration client secret      | Yes       | You set the duration (6 months, 1 year, 2 years). | Rotate the secret before expiration and update the Container Apps auth configuration.                |

### Token Store Connection Credentials

| Connection Method       | Expires?  | Your Responsibility                                                                |
|-------------------------|-----------|------------------------------------------------------------------------------------|
| SAS URL (stable API)    | Yes       | Track the expiration date and rotate the SAS URL before it expires.                |
| Managed Identity (preview API) | No  | No credentials to manage. The platform handles authentication to Blob Storage.     |

### Configuring Entra ID to Issue Refresh Tokens

For the sidecar to refresh expired access tokens, Entra ID must issue refresh tokens. Add the `offline_access` scope to the login parameters:

```json
"login": {
    "loginParameters": ["scope=openid profile email offline_access"]
}
```

### Session Token Grace Period

The 72-hour grace period applies only to the Easy Auth session, not to the provider access tokens. You can extend or shorten this grace period via the `tokenRefreshExtensionHours` property in the auth configuration.

In Bicep or ARM, set this under `login.tokenStore`:

```bicep
login: {
  tokenStore: {
    tokenRefreshExtensionHours: 48 // default is 72
  }
}
```

> [!NOTE]
> On Azure App Service, you can set this via the CLI with `az webapp auth update --token-refresh-extension-hours <HOURS>`. For Azure Container Apps, configure this property through ARM/Bicep or the REST API.

> [!WARNING]
> Extending the session grace period beyond 72 hours has security implications. If a session token is leaked or stolen, a longer window increases the risk of unauthorized access.

---

## Managed Identity vs Easy Auth

These are two separate authentication mechanisms that serve different purposes. They are not interchangeable.

| Aspect             | Easy Auth                                                         | Managed Identity                                                                |
|--------------------|-------------------------------------------------------------------|---------------------------------------------------------------------------------|
| Purpose            | Authenticate **end users** (humans) to your Container App.       | Authenticate **the app itself** to other Azure services.                        |
| Token source       | External identity provider (Entra ID, Google, etc.) via OAuth.   | Azure platform's internal identity endpoint (`IDENTITY_ENDPOINT`).              |
| Token storage      | Token store (Blob Storage) for provider tokens; session cookies for sessions. | No persistent storage. Tokens are fetched on demand from the platform's local REST endpoint. |
| Token expiration   | Session: 8h + 72h grace. Access: approximately 1h (provider-specific). | Access tokens: approximately 1h. The platform handles refresh automatically.    |
| Credential management | Client secret or certificates for the app registration.        | None. No credentials exist. The platform manages everything.                    |
| Use case examples  | User sign-in pages, protecting web UIs, calling APIs on behalf of users. | Connecting to Key Vault, Storage, SQL Database, Container Registry from your app code. |

### How Managed Identity Works in Container Apps

A container app with a managed identity exposes a local REST endpoint via two environment variables:

| Variable           | Purpose                                                        |
|--------------------|----------------------------------------------------------------|
| `IDENTITY_ENDPOINT`| Local URL where your app requests tokens.                      |
| `IDENTITY_HEADER`  | A header value used to mitigate server-side request forgery.   |

Your app requests tokens by calling this endpoint:

```http
GET http://${IDENTITY_ENDPOINT}?resource=https://vault.azure.net&api-version=2019-08-01
x-identity-header: <IDENTITY_HEADER_VALUE>
```

The platform returns an access token:

```json
{
    "access_token": "eyJ0eXAi...",
    "expires_on": "1586984735",
    "resource": "https://vault.azure.net",
    "token_type": "Bearer",
    "client_id": "aaaaaaaaa-0000-1111-2222-bbbbbbbbbbbb"
}
```

> [!NOTE]
> Managed Identity tokens are not stored in the Easy Auth token store. They are completely independent. After updating RBAC role assignments for a managed identity, the backend may cache old permissions for up to 24 hours.

---

## Decision Guide

### Do You Need the Token Store?

```text
Does your app need to call downstream APIs
(Graph, Facebook, Google) on behalf of the user?
    |
    +-- No --> Token store NOT needed.
    |          Use Easy Auth for authentication only.
    |          Read X-MS-CLIENT-PRINCIPAL-* headers.
    |
    +-- Yes --> Token store IS needed.
               Choose connection method:
                   |
                   +-- Production (stable API) --> SAS URL
                   |   Track expiration, rotate before it expires.
                   |
                   +-- Can use preview API --> Managed Identity
                       No expiration, no secrets to manage.
```

### Do You Need Easy Auth at All?

```text
Does your app need to authenticate end users?
    |
    +-- No --> Easy Auth NOT needed.
    |          Use Managed Identity for service-to-service auth.
    |
    +-- Yes --> Do you want platform-managed auth with no code?
                    |
                    +-- Yes --> Use Easy Auth.
                    |
                    +-- No --> Implement auth in your app code
                               using MSAL or another auth library.
```

### Token Store Connection Method Comparison

| Criteria                      | SAS URL (Stable)                   | Managed Identity (Preview)                  |
|-------------------------------|-------------------------------------|---------------------------------------------|
| API version                   | `2025-01-01` and earlier (GA).      | `2025-02-02-preview` (Preview).             |
| Credentials to manage         | SAS URL stored as a secret.         | None.                                       |
| Expiration                    | SAS URL expires on a date you set.  | No expiration.                              |
| Rotation required             | Yes, before SAS expires.            | No.                                         |
| RBAC support                  | No. SAS provides delegated access.  | Yes. Assign Storage Blob Data Contributor.  |
| Production readiness          | GA, production-ready.               | Preview, may have breaking changes.         |
| Portal documentation support  | Fully documented.                   | Not yet documented in portal walkthrough.   |

---

## Security Best Practices

1. Set `allowInsecure` to `false` on ingress to enforce HTTPS for all authentication traffic.
2. Use a client secret with a certificate alternative (`clientSecretCertificateThumbprint`) for app registrations in high-security environments.
3. Rotate app registration client secrets before they expire. Set calendar reminders or automate rotation.
4. If using SAS URLs, set expiration dates and automate rotation using Azure Automation or Functions.
5. When the managed identity option reaches GA, migrate from SAS URLs to eliminate credential expiration concerns.
6. Configure the `offline_access` scope for Entra ID to ensure refresh tokens are issued.
7. Do not extend the session token grace period beyond 72 hours unless your security posture supports it.
8. Restrict authentication to specific users by [configuring the application in Microsoft Entra ID](https://learn.microsoft.com/azure/active-directory/develop/howto-restrict-your-app-to-a-set-of-users) rather than allowing all tenant users.
9. Use Managed Identity (not Easy Auth) for service-to-service authentication to other Azure services. Do not store connection strings or credentials in code.

---

## References

All claims in this document are validated against the following sources:

### Azure REST API Swagger Specifications (Primary Source of Truth)

| API Version             | Spec File                                                                                                                                                                                                             | Key Finding                                                 |
|-------------------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------|
| `2025-01-01` (Stable)   | [AuthConfigs.json](https://github.com/Azure/azure-rest-api-specs/blob/main/specification/app/resource-manager/Microsoft.App/ContainerApps/stable/2025-01-01/AuthConfigs.json)                                         | `BlobStorageTokenStore` requires `sasUrlSettingName` only.  |
| `2025-02-02-preview`    | [AuthConfigs.json](https://github.com/Azure/azure-rest-api-specs/blob/main/specification/app/resource-manager/Microsoft.App/ContainerApps/preview/2025-02-02-preview/AuthConfigs.json)                                | `BlobStorageTokenStore` adds `blobContainerUri`, `clientId`, `managedIdentityResourceId`. |
| `2025-02-02-preview`    | [BlobStorageTokenStore MI Example](https://github.com/Azure/azure-rest-api-specs/blob/main/specification/app/resource-manager/Microsoft.App/ContainerApps/preview/2025-02-02-preview/examples/AuthConfigs_BlobStorageTokenStore_CreateOrUpdate.json)      | Official example using `managedIdentityResourceId`.         |
| `2025-02-02-preview`    | [BlobStorageTokenStore ClientId Example](https://github.com/Azure/azure-rest-api-specs/blob/main/specification/app/resource-manager/Microsoft.App/ContainerApps/preview/2025-02-02-preview/examples/AuthConfigs_BlobStorageTokenStore_ClientId_CreateOrUpdate.json) | Official example using `clientId`.                          |

### Microsoft Learn Documentation

| Topic                                              | URL                                                                                                     |
|----------------------------------------------------|---------------------------------------------------------------------------------------------------------|
| Authentication and authorization in Container Apps | <https://learn.microsoft.com/azure/container-apps/authentication>                                       |
| Enable an authentication token store               | <https://learn.microsoft.com/azure/container-apps/token-store>                                          |
| Managed identities in Container Apps               | <https://learn.microsoft.com/azure/container-apps/managed-identity>                                     |
| Security overview in Container Apps                | <https://learn.microsoft.com/azure/container-apps/security>                                             |
| Secure your Container Apps deployment              | <https://learn.microsoft.com/azure/container-apps/secure-deployment>                                    |
| Manage OAuth tokens (App Service, shared system)   | <https://learn.microsoft.com/azure/app-service/configure-authentication-oauth-tokens>                   |
| App Service authentication overview                | <https://learn.microsoft.com/azure/app-service/overview-authentication-authorization>                   |
| Enable auth with Microsoft Entra ID                | <https://learn.microsoft.com/azure/container-apps/authentication-entra>                                 |
| ARM template reference (authConfigs)               | <https://learn.microsoft.com/azure/templates/microsoft.app/containerapps/authconfigs>                   |

### Key Caveats

> [!IMPORTANT]
> The ARM template reference page on learn.microsoft.com shows properties from the **latest** API version (including preview) without clearly labeling which API version introduced each property. Always cross-reference with the [Swagger specs on GitHub](https://github.com/Azure/azure-rest-api-specs/tree/main/specification/app/resource-manager/Microsoft.App/ContainerApps) when validating API feature availability per version.

> [!NOTE]
> The [token store documentation](https://learn.microsoft.com/azure/container-apps/token-store) currently covers only the SAS URL method. The managed identity approach is defined in the `2025-02-02-preview` API spec but has not been documented in the portal walkthrough as of this writing.
