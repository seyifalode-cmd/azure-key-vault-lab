#  Azure Key Vault Lab — Secrets & Keys Management

Deploying and managing Azure Key Vault for secure secrets, keys, and certificate management — including RBAC vs. Vault Access Policy, purge protection, and least privilege access control.

**Simulated Organization:** Scarstack Solutions Inc.  
**Role:** Cloud Security Engineer  
**Environment:** Azure Portal (East US)  
**Tags:** `Environment=Training` · `Owner=CloudSecurityEngineer`

---

##  Business Scenario

Scarstack Solutions Inc. is migrating sensitive configuration settings, encryption keys, and certificates to Azure for centralized management. This lab deploys two Azure Key Vaults with **intentionally contrasting security configurations** to demonstrate real-world access control trade-offs, soft-delete behavior, and the principle of least privilege.

---

##  Architecture Overview

```
rg-keyvault-sec (Resource Group — East US)
├── kv-scarstack-sec1b       # Vault Access Policy + Purge Protection ON
│   ├── Keys:         key-scarstack-01, key-scarstack-02 (RSA-2048)
│   ├── Secrets:      db-password, api-key
│   └── Certificates: cert-scarstack-01, cert-scarstack-02 (self-signed, exp. 2027-04-29)
│
└── kv-scarstack-secd        # Azure RBAC + Purge Protection OFF
    ├── Keys:         test-key (RSA-2048)
    ├── Secrets:      kv2-secret
    └── Certificates: kv2-cert (self-signed, exp. 2027-04-29)
```

---

##  Lab Steps

### Step 1 — Create Resource Group
Created `rg-keyvault-sec` in East US as the management boundary for both vaults. Grouping resources this way reflects enterprise practice: shared lifecycle, access control, and billing context under one logical container.

---

### Step 2 — Create Key Vault 1 (`kv-scarstack-sec1b`)

| Setting | Value |
|---|---|
| Access Model | Vault Access Policy |
| Purge Protection | Enabled |
| Networking | Public (all networks) |
| Soft-Delete Retention | 90 days |

> **Note:** The original name `kv-scarstack-sec1` was unavailable due to Azure's soft-delete name reservation (90-day hold). `kv-scarstack-sec1b` was used instead — a real-world operational constraint worth knowing.

**Why Vault Access Policy?**  
The traditional model: permissions granted directly on the vault per user/group/app. Simpler to configure, but less scalable than RBAC for larger teams.

**Why Purge Protection?**  
Once enabled, neither users nor Microsoft can permanently delete the vault or its contents during the retention window. Directly supports compliance requirements under SOC 2 and ISO 27001.

---

### Step 3 — Populate Key Vault 1

**Keys** (RSA-2048, Enabled)
- `key-scarstack-01`
- `key-scarstack-02`

**Secrets** (Enabled)
- `db-password`
- `api-key`

**Certificates** (Self-signed, Enabled, Expires 2027-04-29)
- `cert-scarstack-01`
- `cert-scarstack-02`

> Keys handle cryptographic operations (encryption at rest). Secrets store runtime credentials that applications pull instead of hardcoding. Certificates enable centralized SSL/TLS lifecycle management with auto-renewal, eliminating certificate sprawl.

---

### Step 4 — Create Key Vault 2 (`kv-scarstack-secd`)

| Setting | Value |
|---|---|
| Access Model | Azure RBAC |
| Purge Protection |  Disabled |
| Networking | Public (all networks) |

**Why Azure RBAC?**  
The modern, recommended model. Permissions are managed through Azure role assignments (e.g., `Key Vault Administrator`, `Key Vault Secrets User`) at the management plane. Provides finer granularity, better audit trails, and consistency with how all other Azure resources handle access.

**Why Purge Protection Disabled?**  
Intentionally contrasts with KV1. Also allows post-lab cleanup without waiting through the 90-day retention window.

---

### Step 5 — Trigger the RBAC Access Error

Attempted to create a key in `kv-scarstack-secd` immediately after vault creation.

**Error received:**
```
The operation is not allowed by RBAC. If role assignments were recently changed,
Please wait several minutes for role assignments to become effective.
```

**Why this happened:**  
In RBAC mode, *creating* the vault and *accessing its data plane* are separate concerns. Even as the subscription Owner, no Key Vault data plane role had been assigned yet. This is least privilege in action — infrastructure ownership does not automatically grant access to the secrets stored inside.

---

### Step 6 — Fix the RBAC Access Issue

Navigated to `rg-keyvault-sec > Access Control (IAM)` and created a role assignment:

| Field | Value |
|---|---|
| Principal | `youremail@gmail.com` |
| Role | Key Vault Administrator |
| Scope | `rg-keyvault-sec` (inherited by vault) |

Role propagation took approximately **6 minutes** to take effect across Azure's authorization systems.

---

### Step 7 — Populate Key Vault 2 (Post-Fix)

All three objects created successfully after propagation confirmed:

- **Key:** `test-key` (RSA-2048, Enabled)
- **Secret:** `kv2-secret` (Enabled)
- **Certificate:** `kv2-cert` (Self-signed, Enabled, Expires 2027-04-29)

---

## 💡 Key Takeaways

| Concept | Lesson Learned |
|---|---|
| Soft-delete name reservation | Vault names are held for 90 days post-deletion — plan naming conventions accordingly |
| RBAC vs. Vault Access Policy | RBAC is the recommended model; Vault Access Policy is simpler but less scalable |
| Least privilege at the data plane | Subscription Owner ≠ Key Vault data access — explicit role assignment required |
| Role propagation delay | RBAC changes in Azure can take several minutes to fully propagate |
| Purge Protection | Once enabled, it cannot be disabled — a deliberate, compliance-friendly constraint |

---

## 🛠️ Tools & Services

- Azure Key Vault
- Azure IAM / RBAC
- Microsoft Entra ID
- Azure Resource Manager
- Azure Portal


