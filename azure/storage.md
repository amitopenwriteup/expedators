# Lab: Create a Storage Account and One Blob Container in the Azure Portal

## Objective
Create an Azure Storage Account, create a single blob container, upload a file, and verify access using the Azure Portal (UI).

**Estimated time:** 15 minutes
**Skill level:** Beginner
**Cost:** Minimal (less than $0.10)
**Prerequisites:** An active Azure subscription and a web browser.

---

## Background
An Azure **Storage Account** is the top-level container for Azure storage services. **Blob Storage** stores unstructured data such as images, documents, and backups. Blobs live inside **containers**, and each container has an access level that controls whether anonymous users can read its contents.

---

## Lab Architecture

```
Storage Account (rg-storage-lab)
└── Blob Storage
    └── images  (container, Blob-level anonymous read access)
        └── index.html
```

---

## Part 1: Create a Storage Account

### Step 1: Start the wizard
1. Sign in to the Azure Portal (**https://portal.azure.com**).
2. Click **Create a resource** (+ icon).
3. Search for **Storage account**.
4. Click **Create**.

### Step 2: Basics tab

| Field | Value |
|-------|-------|
| **Subscription** | Select your subscription |
| **Resource group** | Click **Create new** > enter `rg-storage-lab` > **OK** |
| **Storage account name** | `stglab` + your initials + random numbers (e.g., `stglabjs12345`) |
| **Region** | East US (or closest to you) |
| **Performance** | Standard |
| **Redundancy** | Locally-redundant storage (LRS) |

> **Naming rules:** 3-24 characters, lowercase letters and numbers only, and globally unique.

### Step 3: Advanced tab
1. Click **Next: Advanced**.
2. Under **Security**:
   - **Allow enabling anonymous access on individual containers**: Enabled (checked)
   - **Enable storage account key access**: Enabled
3. **Hierarchical namespace**: Disabled.
4. Click **Next: Networking**.

### Step 4: Networking and remaining tabs
1. **Network access**: Enable public access from all networks.
2. Click **Next: Data protection** and keep the defaults.
3. Click **Next: Encryption** and keep the defaults.
4. Click **Next: Tags**, skip tags, and click **Review + create**.

### Step 5: Create
1. Review all settings.
2. Click **Create**.
3. Wait 30-60 seconds for the deployment to finish.
4. Click **Go to resource**.

---

## Part 2: Create a Blob Container

1. In the left menu, click **Containers** (under **Data storage**).
2. Click **+ Container**.
3. Enter:

| Field | Value |
|-------|-------|
| **Name** | `images` |
| **Anonymous access level** | **Blob** (anonymous read access for blobs only) |

4. Click **Create**.
5. Confirm `images` appears in the container list.

---

## Part 3: Upload a Blob

### Step 1: Create a sample file
1. Open Notepad or any text editor.
2. Paste this content:

```html
<!DOCTYPE html>
<html>
<head><title>Test Page</title></head>
<body>
<h1>Hello from Azure Blob Storage!</h1>
<p>This file is publicly accessible.</p>
</body>
</html>
```

3. Save it as `index.html` on your desktop.

### Step 2: Upload to Azure
1. In the Azure Portal, click the **images** container.
2. Click **Upload**.
3. Click **Browse for files** and select `index.html`.
4. Expand **Advanced** and set:
   - **Blob type**: Block blob
   - **Block size**: Keep default
   - **Access tier**: Hot
5. Click **Upload**.
6. Wait for the upload to complete and confirm `index.html` is listed.

---

## Part 4: Verify Blob Access

1. Click the uploaded **index.html** blob.
2. In the properties, copy the **URL**.
   - Example: `https://stglabjs12345.blob.core.windows.net/images/index.html`
3. Open a new browser tab.
4. Paste the URL and press **Enter**.
5. You should see your HTML page displayed.

The page loads without signing in because the container allows anonymous read access at the **Blob** level.

---

## Verification Checklist

- [ ] Storage account created successfully with a unique name
- [ ] LRS redundancy configured
- [ ] Container `images` created
- [ ] `index.html` uploaded to the container
- [ ] Blob URL opens in the browser

---

## Troubleshooting

| Issue | Solution |
|-------|----------|
| **Storage account name rejected** | Use 3-24 lowercase letters and numbers only, and make the name globally unique. |
| **Anonymous access level is greyed out** | Enable anonymous access for the storage account: go to **Configuration** and turn on **Allow Blob anonymous access**. |
| **Blob URL shows an error** | Verify the container access level is **Blob**, and check the URL is exactly correct (case-sensitive). |
| **File upload fails** | Check your network connection and that the storage account deployment finished. |

---

## Key Takeaways

- A storage account is the top-level container for Azure storage services.
- Blobs are stored inside containers, and each container has its own access level.
- **Blob** access level allows anonymous read of individual blobs if the URL is known.
- **Private** access level requires authentication and is the safer default for sensitive data.

---

## Clean Up Resources

To avoid charges:
1. Go to **Resource groups**.
2. Click `rg-storage-lab`.
3. Click **Delete resource group**.
4. Type the resource group name `rg-storage-lab` to confirm.
5. Click **Delete**.

All resources in the group will be permanently deleted.
