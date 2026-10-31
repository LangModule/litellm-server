# Microsoft Entra SSO for LiteLLM_DEV

This guide records the Microsoft Entra setup used for this project's LiteLLM dashboard. Users sign in with their Microsoft work accounts, and Entra app roles determine their LiteLLM permissions.

## 1. Register the application

1. Open the [Microsoft Entra admin center](https://entra.microsoft.com/).
2. Go to **Entra ID → App registrations → New registration**.
3. Enter **LiteLLM_DEV** as the application name.
4. Select **Accounts in this organizational directory only** (single tenant).
5. Under **Redirect URI**, select **Web** and enter the callback URL for the deployment.
6. Click **Register**.

For local development:

```text
http://localhost:4000/sso/callback
```

For the current ngrok tunnel:

```text
https://arlo-photoreceptive-subfrontally.ngrok-free.dev/sso/callback
```

Additional Web redirect URIs can be added under **App registrations → LiteLLM_DEV → Authentication**. The callback must match the browser-facing LiteLLM base URL followed by `/sso/callback`.

## 2. Create a client secret

1. Open **App registrations → LiteLLM_DEV → Certificates & secrets**.
2. Under **Client secrets**, click **New client secret**.
3. Enter a description, such as `LiteLLM development SSO`, and choose an expiration according to your organization's policy.
4. Click **Add**.
5. Immediately copy the secret's **Value**, not its **Secret ID**. The value is only displayed when the secret is created.
6. Save the value as `MICROSOFT_CLIENT_SECRET` in the local `.env` file and record its expiration so it can be replaced before it expires.

Do not put the secret value in this document or commit it to Git.

## 3. Configure API permissions

1. Open **App registrations → LiteLLM_DEV → API permissions**.
2. Check whether **Microsoft Graph → User.Read → Delegated** is already present.
3. If it is missing, choose **Add a permission → Microsoft Graph → Delegated permissions**.
4. Select **User.Read**, then click **Add permissions**.
5. If your tenant's consent policy requires administrator approval, have an authorized administrator grant consent.

`User.Read` allows the application to read the signed-in user's profile. Use **Delegated permissions** for this login flow. Broad directory, mail, and application permissions are not needed for basic dashboard sign-in.

## 4. Create the four app roles

Open **App registrations → LiteLLM_DEV → App roles → Create app role**. Create each role below, select **Users/Groups** for **Allowed member types**, and leave **Enable this app role** checked.

The **Value** must match the LiteLLM role exactly, including underscores and lowercase letters.

| Display name | Value | Description |
| --- | --- | --- |
| Proxy Admin | `proxy_admin` | Administer the LiteLLM platform, including users, teams, models, keys, and settings. |
| Proxy Admin Viewer | `proxy_admin_viewer` | View platform-wide keys and spend without administrator write access. |
| Internal User | `internal_user` | Manage own keys and view own spend, subject to configured permissions. |
| Internal User Viewer | `internal_user_viewer` | View own keys and spend without managing them. |

Click **Apply** after creating each role.

## 5. Require assignment to the enterprise application

User access is assigned under **Enterprise applications**, not **App registrations**.

1. Go to **Entra ID → Enterprise applications → All applications**.
2. Select **LiteLLM_DEV**. If there are duplicate names, match its Application ID to the app registration.
3. Open **Properties**.
4. Set **Assignment required?** to **Yes**.
5. Confirm **Enabled for users to sign-in?** is **Yes**, then save.

This restricts sign-in to assigned users or members of assigned groups.

## 6. Assign people and roles

1. Open **Enterprise applications → LiteLLM_DEV → Users and groups**.
2. Click **Add user/group**.
3. Under **Users and groups**, select the people who should have access.
4. Under **Select a role**, choose the appropriate app role.
5. Click **Assign**.

Use `proxy_admin` for platform administrators and `internal_user` for normal developers who need to manage their own keys. Use viewer roles for read-only access. For someone already assigned to the app, select their existing assignment and use **Edit assignment** to change its role.

Group-based application assignment requires Microsoft Entra ID P1 or P2. Individual users can be assigned directly for this setup.

LiteLLM reads the Entra app role during SSO sign-in. That role takes precedence over the user's existing LiteLLM database role. After changing a role, sign out of LiteLLM and sign in again to refresh the session.

## 7. Configure and verify LiteLLM

Our Compose file passes the Microsoft settings from `.env` to the LiteLLM container. `config/config.yaml` disables environment-credential dashboard login and password login when SSO is configured.

For the current tunnel, set:

```dotenv
PROXY_BASE_URL=https://arlo-photoreceptive-subfrontally.ngrok-free.dev
```

The base URL must not include `/ui` or `/sso/callback`. If the tunnel hostname changes, update both `PROXY_BASE_URL` and the Web redirect URI in Entra.

Apply environment changes from the project root:

```bash
docker compose config --quiet
docker compose up -d
```

Keep Docker and ngrok running, then open:

```text
https://arlo-photoreceptive-subfrontally.ngrok-free.dev/ui
```

If ngrok displays its free-plan warning, click **Visit Site**. Sign in with Microsoft and check the role in the LiteLLM profile menu. The administrator account should show **Proxy Admin** after its Entra assignment is applied and it signs in again.

Microsoft SSO login through this tunnel has been tested successfully. The subsequent Proxy Admin assignment still needs to be verified in a fresh session.

## References

- [LiteLLM Microsoft SSO setup](https://docs.litellm.ai/docs/proxy/admin_ui_sso)
- [LiteLLM SSO role mapping](https://docs.litellm.ai/docs/proxy/access_management/sso_permissions)
- [LiteLLM user roles](https://docs.litellm.ai/docs/proxy/self_serve)
- [Microsoft application assignment](https://learn.microsoft.com/en-us/entra/identity/enterprise-apps/assign-user-or-group-access-portal)

## Values to collect at the end

| Value | Where to find it | `.env` variable |
| --- | --- | --- |
| Application (client) ID | App registrations → LiteLLM_DEV → Overview | `MICROSOFT_CLIENT_ID` |
| Directory (tenant) ID | App registrations → LiteLLM_DEV → Overview | `MICROSOFT_TENANT` |
| Client secret **Value** | Certificates & secrets → Client secrets; copy when created | `MICROSOFT_CLIENT_SECRET` |
| Browser-facing LiteLLM base URL | The local address or active ngrok HTTPS URL | `PROXY_BASE_URL` |

Store the values in `.env`, which is ignored by Git:

```dotenv
MICROSOFT_CLIENT_ID=<application-client-id>
MICROSOFT_TENANT=<directory-tenant-id>
MICROSOFT_CLIENT_SECRET=<client-secret-value>
PROXY_BASE_URL=https://arlo-photoreceptive-subfrontally.ngrok-free.dev
```

Also record the client secret's expiration date. Client and tenant IDs identify the application and directory; the client secret value is confidential.
