# Keycloak SSO Setup with ERPNext

## Overview
This guide covers setting up Keycloak as an Identity Provider (IdP) for ERPNext using OpenID Connect (OIDC) for Single Sign-On (SSO).

---

## Part 1: Install Keycloak

### Option A: Docker Installation (Recommended)

```bash
# Pull Keycloak image
docker pull quay.io/keycloak/keycloak:23.0.4

# Start Keycloak container
docker run -d \
  --name keycloak \
  -p 8080:8080 \
  -e KEYCLOAK_ADMIN=admin \
  -e KEYCLOAK_ADMIN_PASSWORD=admin \
  quay.io/keycloak/keycloak:23.0.4 start-dev
```

### Option B: Standalone Installation

```bash
# Download Keycloak
wget https://github.com/keycloak/keycloak/releases/download/23.0.4/keycloak-23.0.4.zip

# Extract
unzip keycloak-23.0.4.zip
cd keycloak-23.0.4

# Start Keycloak
bin/kc.sh start-dev
```

### Access Keycloak Admin Console

Open browser: `http://localhost:8080`
- **Username**: admin
- **Password**: admin (or your configured password)

---

## Part 2: Configure Keycloak Realm

### 1. Create New Realm

1. Click on **Master** dropdown (top left)
2. Click **Create Realm**
3. Enter realm name: `erpnext-realm`
4. Click **Create**

### 2. Create ERPNext Client

1. Navigate to **Clients** → **Create Client**
2. Configure:
   - **Client Type**: OpenID Connect
   - **Client ID**: `erpnext`
   - **Name**: ERPNext
   - **Enabled**: ON
3. Click **Next**
4. Configure:
   - **Client authentication**: OFF (for public clients)
   - **Valid redirect URIs**: `http://erp.yourdomain.com/*`
   - **Web origins**: `http://erp.yourdomain.com`
5. Click **Save**

### 3. Configure Client Settings

Go to **Client Scopes** tab:
1. Add scope: `openid`
2. Add scope: `profile`
3. Add scope: `email`

Go to **Settings** tab:
1. **Access Type**: public
2. **Valid Redirect URIs**: `http://erp.yourdomain.com/*`
3. **Web Origins**: `http://erp.yourdomain.com`

### 4. Create Realm User

1. Navigate to **Users** → **Add User**
2. Fill in:
   - **Username**: erpnext-admin
   - **Email**: admin@yourdomain.com
   - **Email Verified**: ON
3. Click **Create**
4. Go to **Credentials** tab
5. Set password and click **Set Password**

### 5. Get Client Secret (if using confidential client)

If you configure the client as **confidential**:
1. Go to **Credentials** tab
2. Copy the **Client Secret**

---

## Part 3: Configure ERPNext for Keycloak SSO

### 1. Enable Social Login in ERPNext

1. Login to ERPNext as Administrator
2. Go to **Setup** → **Settings** → **Social Login Key**
3. Click **Add New**

### 2. Configure Social Login Settings

Fill in the following fields:

| Field | Value |
|-------|-------|
| **Provider Name** | keycloak |
| **Enable Social Login** | Yes |
| **Social Login Provider** | Custom |
| **Client ID** | erpnext |
| **Client Secret** | (from Keycloak credentials) |
| **Base URL** | `http://keycloak-server:8080/realms/erpnext-realm` |
| **Authorize URL** | `/protocol/openid-connect/auth` |
| **Access Token URL** | `/protocol/openid-connect/token` |
| **Redirect URL** | `/api/method/frappe.integrations.oauth2_logins.custom/keycloak` |
| **API Endpoint** | `http://keycloak-server:8080/realms/erpnext-realm/protocol/openid-connect/userinfo` |
| **Auth URL Data** | `{"response_type":"code","scope":"openid profile email"}` |
| **User ID Property** | preferred_username |

**Important**: The API Endpoint must be an absolute URL, not relative.

### 3. Generate API Secret (for confidential clients)

```bash
# SSH into ERPNext server
bench execute frappe.core.doctype.user.user.generate_keys --args ['Administrator']
```

### 4. Enable Social Login Button

1. Go to **Setup** → **Settings** → **Website Settings**
2. Under **Social Login**, add:
   - **Provider**: keycloak
   - **Icon**: (upload Keycloak icon)
3. Save

---

## Part 4: Test SSO Login

1. Open ERPNext login page
2. You should see the Keycloak login button
3. Click the button
4. Redirected to Keycloak login
5. Enter Keycloak credentials
6. Redirected back to ERPNext (logged in)

---

## Part 5: Advanced Configuration

### Role Mapping (OIDC Extended)

For advanced role mapping, install the OIDC Extended app:

```bash
bench get-app https://github.com/MohammedNoureldin/frappe-oidc-extended
bench --site erp.yourdomain.com install-app oidc_extended
```

### Configure Role Profiles

1. Go to **Setup** → **Roles and Permissions** → **Role Profile**
2. Create profiles for different user types
3. In Social Login Key, map Keycloak groups to Role Profiles

### Logout Configuration

To enable SSO logout from ERPNext:

1. Go to Social Login Key settings
2. Update **Auth URL Data**:
```json
{
  "response_type": "code",
  "scope": "openid profile email",
  "prompt": "login"
}
```

### Email Verification Issue

If you get "Email not verified" error:
1. Ensure user email is verified in Keycloak
2. Check API Endpoint is absolute URL
3. Verify user attributes in Keycloak

---

## Part 6: Troubleshooting

### Common Issues

1. **Login button not showing**: Check Social Login Key is enabled
2. **Redirect URI mismatch**: Verify redirect URLs match exactly
3. **Email not verified**: Check API Endpoint is absolute URL
4. **Session not persisting**: Check cookie settings

### Logs

```bash
# Check ERPNext logs
tail -f sites/erp.yourdomain.com/logs/frappe.log

# Check Keycloak logs (Docker)
docker logs keycloak
```

### Test OIDC Endpoints

```bash
# Test well-known endpoint
curl http://keycloak-server:8080/realms/erpnext-realm/.well-known/openid-configuration

# Test userinfo endpoint
curl -H "Authorization: Bearer YOUR_TOKEN" \
  http://keycloak-server:8080/realms/erpnext-realm/protocol/openid-connect/userinfo
```

---

## Security Considerations

1. Use HTTPS in production
2. Change default Keycloak admin password
3. Enable email verification in Keycloak
4. Use confidential client type for better security
5. Regularly rotate client secrets
6. Enable CSRF protection
7. Configure session timeout appropriately
