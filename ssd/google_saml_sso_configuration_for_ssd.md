# Google SAML SSO Configuration for SSD

This document describes how to configure **Google SAML Single Sign-On (SSO)** for an **OpsMx SSD** instance, including required Kubernetes configurations and post-setup steps.

---

## Prerequisites

- Google Workspace admin access
- SSD deployed on Kubernetes
- Namespace where SSD is installed (referred to as `<NAMESPACE>`)
- Access to Kubernetes cluster with `kubectl`
- Google SAML **IdP Metadata XML** file downloaded from Google Admin Console

---

## Step 1: Download Google SAML Metadata

1. In **Google Admin Console**, create and configure a new **SAML App** for SSD.
2. Download the **IdP Metadata** file.
   - Default filename: `GoogleIDPMetadata.xml`

---

## Step 2: Create Kubernetes Secret for SAML Metadata

SSD expects the metadata file to be named **`metadata.xml`**.

1. Rename the downloaded file:

```bash
mv GoogleIDPMetadata.xml metadata.xml
```

2. Create a Kubernetes secret named **`saml-files`**:

```bash
kubectl create secret generic saml-files \
  --from-file=metadata.xml \
  -n <NAMESPACE>
```

---

## Step 3: Update `ssd-gate-config` ConfigMap

Edit the `ssd-gate-config` ConfigMap and add the following configuration:

```yaml
metadata_File: /app/saml/metadata.xml
```

Command:

```bash
kubectl edit cm ssd-gate-config -n <NAMESPACE>
```

---

## Step 4: Update `ssd-gate` Deployment to Mount SAML Secret

Edit the **`ssd-gate` deployment** and add the following entries.

### Volume Mounts

```yaml
volumeMounts:
  - mountPath: /app/config
    name: config-home
  - mountPath: /app/services
    name: services-config
  - mountPath: /app/saml
    name: saml-files
```

### Volumes

```yaml
volumes:
  - name: saml-files
    secret:
      secretName: saml-files
      defaultMode: 420
      optional: true
      items:
        - key: metadata.xml
          path: metadata.xml
```

Apply the changes:

```bash
kubectl edit deploy ssd-gate -n <NAMESPACE>
```

---

## Step 5: Restart SSD Gate Pod

Restart the gate deployment to apply the configuration changes:

```bash
kubectl rollout restart deploy ssd-gate -n <NAMESPACE>
```

Wait until the pod is fully running.

---

## Step 6: Configure Organisation and Role Mapping

When using **Basic Authentication**, SSD groups (e.g., `admin`) and **SAML groups** (e.g., `ssd-sandbox-admin`) are different.

To avoid **"team/org details not found"** issues, register the SAML admin group using the **supplychain-api**.

### API Details

- **URL:** `http://localhost:8099/ssdservice/v1/setup`
- **Method:** `POST`
- **Tool:** Postman / curl

### Payload

```json
{
  "organisationName": "opsmx",
  "organisationRoles": [
    {
      "group": "ssd-sandbox-admin",
      "permission": "admin"
    }
  ]
}
```

> Ensure the group name exactly matches the **Google SAML group**.

---

## Step 7: Restart Gate Service Again

After the organisation setup, restart the gate service once more:

```bash
kubectl rollout restart deploy ssd-gate -n <NAMESPACE>
```

---

## Step 8: Validate SAML Login

1. Access the SSD URL in a browser.
2. Login using your **OpsMx Google email ID**.
3. Successful login should display all SSD data without errors.

---

## Step 9: User & Team Access Management (Optional)

If required:

1. Navigate to:
   **Setup → Access Management**
2. Assign appropriate **roles and permissions** to:
   - Teams
   - Users

---

## Troubleshooting

### 403: `app_not_enabled_for_user`
- Ensure the Google SAML app is **enabled for the user or user group** in Google Admin Console.

### Organisation or Team Not Found
- Verify the **SAML group name** matches the payload sent to `/setup` API.
- Ensure the setup API was executed successfully.

---

## Summary

- Google SAML metadata is mounted via Kubernetes secret
- SSD Gate reads metadata from `/app/saml/metadata.xml`
- Organisation-role mapping is mandatory for SAML groups
- Proper restart sequence is required for changes to take effect

---

**Document Version:** 1.0

