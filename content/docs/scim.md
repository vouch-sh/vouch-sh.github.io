---
title: "Automate User Provisioning with SCIM"
linkTitle: "SCIM Provisioning"
description: "Sync users and groups from your identity provider to Vouch automatically using SCIM 2.0."
weight: 11
subtitle: "Automate user lifecycle management with your identity provider"
params:
  docsGroup: manage
---

Manually adding and removing users from Vouch when people join or leave your organization is error-prone and easy to forget. A missed offboarding means someone retains access to hardware-backed credentials they should no longer have.

SCIM (System for Cross-domain Identity Management) lets your identity provider -- Google Workspace, Okta, Azure AD, or OneLogin -- handle this automatically in real time. Vouch supports the **SCIM 2.0** protocol ([RFC 7644](https://datatracker.ietf.org/doc/html/rfc7644)) for automated user provisioning and de-provisioning. When SCIM is configured, your identity provider (IdP) can automatically:

- **Create** new Vouch user accounts when people join your organization.
- **Update** user attributes (name, email, role) when they change in your directory.
- **Deactivate** accounts instantly when someone leaves or changes roles.

This eliminates manual user management and ensures that credential access is always in sync with your corporate directory.

---

## How It Works

1. You generate a SCIM bearer token from the Vouch server.
2. You configure your identity provider to point at Vouch's SCIM 2.0 endpoint.
3. The IdP pushes user lifecycle events (create, update, deactivate) to Vouch in real time.
4. Vouch processes each event and updates its internal user directory accordingly.

Because SCIM is a standardized protocol, Vouch works with any identity provider that supports SCIM 2.0 -- including Google Workspace, Okta, Azure AD (Entra ID), and OneLogin.

---

## Step 1 -- Generate a SCIM Token

Before your identity provider can communicate with Vouch, you need to generate a bearer token that the IdP will use to authenticate its requests.

First, ensure you are logged in with an account that has **organization administrator** privileges:

```bash
vouch login
```

Then create a SCIM token:

```bash
curl -X POST {{< instance-url >}}/api/v1/org/scim-tokens \
  -b ~/.vouch/cookie.txt \
  -H "Content-Type: application/json" \
  -d '{"description": "Google Workspace SCIM", "expires_in_days": 365}'
```

The response includes the plaintext token:

```json
{
  "id": "scim_tok_abc123",
  "description": "Google Workspace SCIM",
  "token": "vouch_scim_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx",
  "expires_at": "2027-01-15T00:00:00Z",
  "created_at": "2026-01-15T12:00:00Z"
}
```

> **Security note:** Vouch stores only a cryptographic hash of the token. The plaintext value is shown **exactly once** in the creation response. Copy it immediately and store it securely -- you will not be able to retrieve it again. If you lose the token, revoke it and create a new one.

The `expires_in_days` field is **required** and must be an integer between **1** and **365**. Choose an expiration period that balances security with operational convenience. Most organizations use 365 days and rotate tokens annually.

---

## Step 2 -- Configure Your Identity Provider

Use the following values when configuring SCIM in your identity provider:

| Setting | Value |
|---|---|
| **SCIM Base URL** | `{{< instance-url >}}/scim/v2` |
| **Authentication Type** | Bearer Token |
| **Authorization Header** | `Bearer vouch_scim_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx` |

Replace the token value with the plaintext token you received in Step 1.

### Google Workspace

1. Open the [Google Admin Console](https://admin.google.com).
2. Navigate to **Apps > Web and mobile apps**.
3. Find or add the Vouch application.
4. Open the **Auto-provisioning** section.
5. Set the **SCIM Base URL** to `{{< instance-url >}}/scim/v2`.
6. Set the **Authentication Type** to **Bearer Token** and paste your SCIM token.
7. Click **Test Connection** to verify.
8. Enable auto-provisioning and configure the desired attribute mappings.
9. Click **Save**.

### Okta

1. Open the Okta Admin Dashboard.
2. Navigate to **Applications > Applications**.
3. Select or create the Vouch application.
4. Go to the **Provisioning** tab and click **Configure API Integration**.
5. Check **Enable API Integration**.
6. Set the **SCIM 2.0 Base URL** to `{{< instance-url >}}/scim/v2`.
7. Set the **API Token** to the SCIM bearer token from Step 1.
8. Click **Test API Credentials** to verify connectivity.
9. Click **Save**.
10. Under **Provisioning > To App**, enable the desired actions: **Create Users**, **Update User Attributes**, and **Deactivate Users**.

### Azure AD (Entra ID)

1. Open the [Azure Portal](https://portal.azure.com) and navigate to **Azure Active Directory > Enterprise Applications**.
2. Select or create the Vouch application.
3. Go to **Provisioning** and set the **Provisioning Mode** to **Automatic**.
4. Under **Admin Credentials**:
   - Set **Tenant URL** to `{{< instance-url >}}/scim/v2`.
   - Set **Secret Token** to the SCIM bearer token from Step 1.
5. Click **Test Connection** to verify that Azure can reach the Vouch SCIM endpoint.
6. Configure attribute mappings under **Mappings** to align Azure AD attributes with Vouch user fields.
7. Set **Provisioning Status** to **On**.
8. Click **Save** to begin provisioning.

### OneLogin

1. Open the OneLogin Admin Panel.
2. Navigate to **Applications > Applications**.
3. Select or create the Vouch application.
4. Go to the **Provisioning** tab.
5. Enable provisioning and set the **SCIM Base URL** to `{{< instance-url >}}/scim/v2`.
6. Set the **SCIM Bearer Token** to the token from Step 1.
7. Under **Provisioning Actions**, enable **Create user**, **Update user**, and **Delete user**.
8. Click **Save**.

---

## Step 3 -- Test the Integration

After configuring your identity provider, verify that the SCIM connection is working correctly by querying the Vouch SCIM endpoints directly.

### Check the Service Provider Configuration

This endpoint returns the SCIM capabilities supported by Vouch:

```bash
curl -s {{< instance-url >}}/scim/v2/ServiceProviderConfig \
  -H "Authorization: Bearer vouch_scim_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx" \
  | jq .
```

A successful response returns a JSON object describing the supported SCIM features, including authentication schemes, bulk support, and filtering capabilities.

### List Provisioned Users

Retrieve the list of users that have been provisioned through SCIM:

```bash
curl -s {{< instance-url >}}/scim/v2/Users \
  -H "Authorization: Bearer vouch_scim_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx" \
  | jq .
```

This returns a `ListResponse` containing all SCIM-managed users and their attributes.

### List Provisioned Groups

Retrieve the list of groups that have been provisioned through SCIM:

```bash
curl -s {{< instance-url >}}/scim/v2/Groups \
  -H "Authorization: Bearer vouch_scim_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx" \
  | jq .
```

This returns a `ListResponse` containing all SCIM-managed groups and their membership information.

---

## SCIM 2.0 Endpoints

Vouch implements the following SCIM 2.0 endpoints:

| Method | Endpoint | Description |
|---|---|---|
| `GET` | `/scim/v2/ServiceProviderConfig` | Returns the SCIM service provider configuration and supported features. |
| `GET` | `/scim/v2/ResourceTypes` | Lists the resource types (Users, Groups) supported by the server. |
| `GET` | `/scim/v2/Schemas` | Returns the full SCIM schemas supported by the server. |
| `GET` | `/scim/v2/Users` | Lists all provisioned users. Supports filtering and pagination. |
| `POST` | `/scim/v2/Users` | Creates a new user account. |
| `GET` | `/scim/v2/Users/{id}` | Retrieves a specific user by their SCIM ID. |
| `PUT` | `/scim/v2/Users/{id}` | Replaces all attributes of a specific user. |
| `PATCH` | `/scim/v2/Users/{id}` | Updates specific attributes of a user (partial update). |
| `DELETE` | `/scim/v2/Users/{id}` | Deactivates (soft-deletes) a user account. |
| `GET` | `/scim/v2/Groups` | Lists all provisioned groups. Supports filtering and pagination. |
| `POST` | `/scim/v2/Groups` | Creates a new group. |
| `GET` | `/scim/v2/Groups/{id}` | Retrieves a specific group by its SCIM ID. |
| `PUT` | `/scim/v2/Groups/{id}` | Replaces all attributes of a specific group. |
| `PATCH` | `/scim/v2/Groups/{id}` | Updates specific attributes of a group (partial update). |
| `DELETE` | `/scim/v2/Groups/{id}` | Deletes a group. |

All endpoints require a valid SCIM bearer token in the `Authorization` header. Filtering is supported on the `Users` and `Groups` list endpoints using the SCIM filter syntax (e.g., `?filter=userName eq "alice@example.com"`).

---

## Immediate De-provisioning

One of the most important benefits of SCIM integration is **immediate de-provisioning**. When an employee leaves your organization or changes roles:

1. Your identity provider sends a `PATCH` or `DELETE` request to the Vouch SCIM endpoint to deactivate the user.
2. Vouch immediately marks the user account as inactive.
3. All **active sessions** for that user are revoked instantly.
4. The user can no longer obtain new credentials. Previously issued short-lived credentials (SSH certificates, AWS STS credentials) will continue to function until their natural expiration (up to 8 hours). No new credentials can be issued after de-provisioning.

Because all Vouch credentials are short-lived (maximum 8 hours), the exposure window after de-provisioning is limited. Sessions are revoked immediately, and outstanding credentials expire on their own shortly after.

This is a significant security improvement over traditional provisioning workflows where revoking access requires manual steps across multiple systems. With Vouch and SCIM, de-provisioning is automated and the blast radius is minimized by the short credential lifetime.

---

## Managing SCIM Tokens

Organization administrators can list, create, and revoke SCIM tokens through the Vouch API.

### List All SCIM Tokens

Retrieve all active SCIM tokens for your organization:

```bash
curl -s {{< instance-url >}}/api/v1/org/scim-tokens \
  -b ~/.vouch/cookie.txt \
  | jq .
```

The response includes token metadata (ID, description, creation date, expiration date) but never the plaintext token value.

### Create a New Token

```bash
curl -X POST {{< instance-url >}}/api/v1/org/scim-tokens \
  -b ~/.vouch/cookie.txt \
  -H "Content-Type: application/json" \
  -d '{"description": "Okta SCIM Integration", "expires_in_days": 180}'
```

The response includes the plaintext token. Store it securely -- it will not be shown again.

### Revoke a Token

Revoke a SCIM token by its ID to immediately disable it:

```bash
curl -X DELETE {{< instance-url >}}/api/v1/org/scim-tokens/scim_tok_abc123 \
  -b ~/.vouch/cookie.txt
```

After revocation, any identity provider using this token will receive `401 Unauthorized` responses and provisioning will stop until a new token is configured.

---

## Rotating SCIM Tokens

To rotate a SCIM token without interrupting provisioning, follow this four-step process:

1. **Create a new token** with a descriptive name that indicates it is the replacement:

   ```bash
   curl -X POST {{< instance-url >}}/api/v1/org/scim-tokens \
     -b ~/.vouch/cookie.txt \
     -H "Content-Type: application/json" \
     -d '{"description": "Google Workspace SCIM (rotated 2026-02)", "expires_in_days": 365}'
   ```

2. **Update your identity provider** with the new token value. Follow the configuration steps for your IdP described in Step 2 above, replacing the old token with the new one.

3. **Test the new token** by triggering a sync from your identity provider or by querying the SCIM endpoint directly with the new token:

   ```bash
   curl -s {{< instance-url >}}/scim/v2/Users \
     -H "Authorization: Bearer vouch_scim_NEW_TOKEN_HERE" \
     | jq '.totalResults'
   ```

4. **Revoke the old token** once you have confirmed that the new token is working:

   ```bash
   curl -X DELETE {{< instance-url >}}/api/v1/org/scim-tokens/scim_tok_OLD_ID \
     -b ~/.vouch/cookie.txt
   ```

By creating the new token before revoking the old one, you ensure there is no window during which provisioning is interrupted.

---

## Troubleshooting

### 401 Unauthorized

The SCIM token is invalid, expired, or revoked.

- Verify the token has not expired by listing your active tokens.
- Ensure the `Authorization` header uses the format `Bearer <token>` with no extra whitespace or characters.
- If the token was recently created, confirm you copied the full plaintext value -- it is only shown once during creation.
- If the token has expired or been revoked, create a new token and update your identity provider configuration.

### 409 Conflict

A user or group with the same unique identifier already exists.

- This typically occurs when your identity provider attempts to create a user who has already been provisioned, either through a previous SCIM sync or through manual registration.
- Check whether the user already exists in Vouch by querying the Users endpoint with a filter:

  ```bash
  curl -s "{{< instance-url >}}/scim/v2/Users?filter=userName%20eq%20%22alice%40example.com%22" \
    -H "Authorization: Bearer vouch_scim_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx" \
    | jq .
  ```

- If the user exists, your IdP should use `PUT` or `PATCH` to update the existing record rather than `POST` to create a new one. Most identity providers handle this automatically after the initial conflict.

### Users Not Syncing

If users are not appearing in Vouch after configuring SCIM:

- **Verify the SCIM Base URL.** Ensure it is set to `{{< instance-url >}}/scim/v2` with no trailing slash.
- **Test connectivity.** Use the `ServiceProviderConfig` endpoint to confirm the IdP can reach Vouch:

  ```bash
  curl -s {{< instance-url >}}/scim/v2/ServiceProviderConfig \
    -H "Authorization: Bearer vouch_scim_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx" \
    | jq .
  ```

- **Check your IdP's provisioning logs.** Most identity providers maintain a log of SCIM operations and any errors encountered. Look for HTTP error codes or timeout messages.
- **Confirm provisioning is enabled.** In some IdPs (especially Okta and Azure AD), you must explicitly enable provisioning actions (Create, Update, Deactivate) after configuring the API connection.
- **Verify user assignment.** In many IdPs, users must be explicitly assigned to the Vouch application before they will be provisioned. Check that the intended users or groups are assigned.

### Attribute Mapping Issues

If user attributes (name, email, department) are not appearing correctly in Vouch:

- Review the attribute mappings in your identity provider's SCIM configuration.
- Vouch expects the standard SCIM 2.0 `User` schema attributes:
  - `userName` -- The user's email address (used as the unique identifier).
  - `name.givenName` -- The user's first name.
  - `name.familyName` -- The user's last name.
  - `emails` -- An array of email objects; the `primary` email is used for notifications.
  - `active` -- A boolean indicating whether the account is active.
- Ensure your IdP is mapping its directory attributes to these standard SCIM fields.
