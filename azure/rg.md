# Lab: Create a Resource Group Using the Azure Portal

## Objective
Create, view, tag, and delete an Azure Resource Group using the Azure Portal (UI).

**Estimated time:** 10 minutes
**Prerequisites:** An active Azure subscription (free trial works) and a web browser.

---

## Background
A **resource group** is a logical container that holds related Azure resources (VMs, storage accounts, databases, etc.). Resources in a group share the same lifecycle, so you can manage, monitor, and delete them together.

---

## Part 1: Sign In to the Azure Portal

1. Open your browser and go to **https://portal.azure.com**.
2. Sign in with your Azure account credentials.
3. Wait for the **Azure Portal Home** page to load.

---

## Part 2: Create a Resource Group

### Step 1: Open Resource Groups
1. In the **search bar** at the top, type `Resource groups`.
2. Select **Resource groups** from the results.

   *(Alternative: Click the portal menu (☰) → **Resource groups**.)*

### Step 2: Start the creation wizard
1. Click **+ Create** at the top of the Resource groups page.

### Step 3: Fill in the Basics tab

| Field | Value |
|-------|-------|
| **Subscription** | Select your subscription |
| **Resource group name** | `rg-lab-demo-001` |
| **Region** | Choose a nearby region (e.g., `Central India` or `South India`) |

> **Naming tip:** Names must be unique within your subscription, 1–90 characters, and can include letters, numbers, underscores, hyphens, periods, and parentheses.

### Step 4: Add Tags
1. Click **Next: Tags**.
2. Add the following tags:

| Name | Value |
|------|-------|
| `Environment` | `Lab` |
| `Owner` | `<your-name>` |
| `Purpose` | `Training` |

### Step 5: Review and create
1. Click **Next: Review + create**.
2. Wait for **Validation passed** to appear.
3. Click **Create**.

---

## Part 3: Verify the Resource Group

1. Click the **Notifications** bell icon (top right). You should see **Resource group created**.
2. Click **Go to resource group**, or go to **Resource groups** and select `rg-lab-demo-001`.
3. On the **Overview** page, confirm:
   - Subscription name and ID
   - Location (region)
   - Tags
   - Deployment status

---


---

## Verification Checklist

- [ ] Signed in to the Azure Portal
- [ ] Created `rg-lab-demo-001` in the chosen region
- [ ] Added tags


---

## Troubleshooting

| Issue | Solution |
|-------|----------|
| **Name already exists** | Choose a different name; it must be unique in your subscription. |
| **No subscription listed** | Make sure your account has an active subscription or permission to use one. |
| **Create button disabled** | You may lack the *Contributor* or *Owner* role on the subscription. |
| **Policy error on region** | An Azure Policy may restrict allowed regions; pick an allowed one. |

---

## Key Takeaways

- A resource group is a **logical container** for related resources.
- The resource group has a **region** that stores its metadata, but resources inside can be in different regions.
- **Tags** help with organization and cost tracking.
- Deleting a resource group deletes **everything** inside it.
