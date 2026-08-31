# Storage Credential vs External Location in Unity Catalog

## Question
Explain the architectural relationship and difference between a **Storage Credential** and an **External Location** in Databricks Unity Catalog.

---

## 1. Architectural Relationship

```
[ Azure Managed Identity / AWS IAM Role ]
                   │
                   ▼ (Encapsulated into)
[ 1. Storage Credential (Authentication Object) ]
  • Represents *HOW* Databricks authenticates with cloud storage.
                   │
                   ▼ (Referenced by)
[ 2. External Location (Authorization Object) ]
  • Combines a Cloud URI (`abfss://...`) + a Storage Credential.
  • Represents *WHERE* data resides and *WHO* has permission to access it.
                   │
                   ▼ (Grants access to)
[ Catalogs / Schemas / External Tables ]
```

---

## 2. DDL Configuration Syntax

```sql
-- Step 1: Define Storage Credential (Authentication)
CREATE STORAGE CREDENTIAL adls_managed_identity_cred
WITH (
  AZURE_MANAGED_IDENTITY = (
    RESOURCE_ID = '/subscriptions/.../accessConnectors/db-connector-prod'
  )
);

-- Step 2: Define External Location (Authorization)
CREATE EXTERNAL LOCATION curated_finance_lake
URL 'abfss://finance@citiadls.dfs.core.windows.net/'
WITH (STORAGE CREDENTIAL adls_managed_identity_cred);

-- Step 3: Grant Granular Access
GRANT READ, WRITE ON EXTERNAL LOCATION curated_finance_lake TO `finance-data-engineers`;
```
