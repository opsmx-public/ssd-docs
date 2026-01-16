# Google SAML Application – Creation & Configuration Guide

This document provides **step-by-step instructions** to create and configure a **Custom Google SAML application** for an application like **SSD Demo Server**. It covers:
- Creating a new Custom SAML app
- Configuring Service Provider (SP) details
- Downloading and using IdP metadata
- Attribute mapping (user attributes)
- Enabling user access
- Group-based access control (recommended)

---

## Prerequisites

Before starting, ensure you have:
- **Google Workspace Admin** privileges
- Application **ACS URL** and **Entity ID** from the Service Provider (SSD)
- Application domain accessible over HTTPS

---

## Step 1: Navigate to Web & Mobile Apps

1. Login to **Google Admin Console**
2. Go to:
   
   **Apps → Web and mobile apps**
3. You will see the list of existing applications

---

## Step 2: Add a New Custom SAML App

1. Click **Add app**
2. Select **Add custom SAML app**

---

## Step 3: App Details

Fill in the basic application information:

- **App name**: `ssd-demoserver`
- **Description**: `ssd.ssd-demo.opsmx.net`
- **App icon**: (Optional – upload SSD logo)

Click **CONTINUE**

---

## Step 4: Google Identity Provider (IdP) Details

You will now see Google IdP details.

### Option 1 (Recommended): Download Metadata

1. Click **DOWNLOAD METADATA**
2. File downloaded: `GoogleIDPMetadata.xml`

> This metadata file will be required on the application (SSD) side

Click **CONTINUE**

### Option 2 (Alternative): Manual Configuration

You can also manually copy:
- **SSO URL**
- **Entity ID**
- **Certificate**

---

## Step 5: Service Provider Details

Configure details provided by the SSD application.

### Required Fields

- **ACS URL**  
  `https://ssd.ssd-demo.opsmx.net/saml/acs`

- **Entity ID**  
  `https://ssd.ssd-demo.opsmx.net/saml/metadata`

- **Signed response**: ✅ Enabled

### Name ID Configuration

- **Name ID format**: `EMAIL`
- **Name ID**: `Basic Information → Primary email`

Click **CONTINUE**

---

## Step 6: Attribute Mapping (User Attributes)

Map Google user attributes to application attributes.

### Default Mappings

| Google Directory Attribute | App Attribute |
|----------------------------|--------------|
| Basic Information → First name | firstName |
| Basic Information → Last name  | lastName  |

Click **ADD MAPPING** if additional attributes are needed.

> These attributes are commonly used by SSD for user profile creation

Click **FINISH**

---

## Step 7: Enable User Access

By default, newly created apps are **OFF for everyone**.

1. Open the newly created app (`ssd-demoserver`)
2. Go to **User access**

### Option A: Enable for Everyone (Not Recommended for Prod)

- Set **User access** → **ON for everyone**

### Option B: Enable for Selected Groups (Recommended)

- Select **ON for selected groups**
- Choose the required Google Groups

Examples:
- `ssd-admins` → Admin users
- `ssd-users` → Regular users

Click **SAVE**

---

## Step 8: Group-Based Access Control (Recommended)

### Create Google Groups

Go to:
**Directory → Groups → Create Group**

Examples:
- `ssd-admins@company.com`
- `ssd-users@company.com`

Add users to respective groups.

### Use Groups for Access Control

- Assign **ssd-admins** group to app with Admin privileges
- Assign **ssd-users** group for normal access

> Group-based access avoids enabling the app for all users and improves security

---

## Step 9: (Optional) Group Attribute Mapping

If SSD supports group claims:

1. Go to **Attribute mapping**
2. Add mapping:
   - Google attribute: `Groups`
   - App attribute: `groups`

This allows SSD to:
- Identify admin vs non-admin users
- Apply RBAC internally

---

## Step 10: Validate SAML Login

1. Open the app details page
2. Click **TEST SAML LOGIN**
3. Login with a user assigned to the app
4. Verify successful redirection to SSD

---

## Summary

You have successfully:
- Created a Custom Google SAML application
- Configured SP details (ACS & Entity ID)
- Downloaded and shared IdP metadata
- Mapped user attributes
- Enabled user access using groups

This setup is suitable for **Dev, Demo, and Production** environments.

---

## Recommended Best Practices

- Always prefer **group-based access** over enabling for everyone
- Rotate certificates before expiry
- Use separate SAML apps per environment (dev/demo/prod)
- Keep metadata files versioned

---

**Document Owner:** OpsMx DevOps Team  
**Application:** SSD Demo Server

---

## Appendix A: GitHub‑Friendly Snippets

### Test SAML Login

```text
Admin Console → Apps → Web and mobile apps → ssd-demoserver → TEST SAML LOGIN
```

### Typical SSD-Side Kubernetes Commands (Reference)

```bash
# Rename downloaded metadata
mv GoogleIDPMetadata.xml metadata.xml

# Create secret with IdP metadata
kubectl create secret generic saml-files \
  --from-file=metadata.xml \
  -n <NAMESPACE>

# Restart Gate after configuration
kubectl rollout restart deploy ssd-gate -n <NAMESPACE>
```

> Note: Keep this document as a `.md` file in GitHub for best rendering (tables, fenced blocks, headings).

