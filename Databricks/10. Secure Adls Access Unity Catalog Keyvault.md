# Scenario 10: Secure ADLS Gen2 Access Architecture: Unity Catalog, Secrets & RBAC

## Problem Statement
How do you architect secure, segregated access to Azure Data Lake Storage (ADLS Gen2) from Databricks across Dev, Test, and Prod environments without hardcoding credentials or exposing storage keys?

---

## 1. Modern Enterprise Security Architecture (Unity Catalog + Managed Identities)

```
[ Databricks Workspace (Dev / Test / Prod) ]
                      │
                      ▼
            [ Unity Catalog Metastore ]
                      │
                      ▼ (Azure Access Connector / Managed Identity)
     [ Storage Credentials (No Keys, Pure Entra ID Identity) ]
                      │
                      ▼ (Grants on External Locations)
  [ ADLS Gen2 Storage Account: `abfss://curated@adlsdev.dfs.core.windows.net/` ]
```

---

## 2. Pillar 1: Unity Catalog External Locations & Storage Credentials

The modern Databricks standard eliminates DBFS mounts and shared access keys completely:

1. **Create Azure Access Connector:** Provision an Azure Managed Identity for Databricks in the Azure Subscription.
2. **Assign Storage RBAC in Azure:** Grant `Storage Blob Data Contributor` to the Access Connector on the ADLS Gen2 account.
3. **Define Storage Credential in Unity Catalog:**
   ```sql
   CREATE STORAGE CREDENTIAL dev_adls_credential
   WITH (
     AZURE_MANAGED_IDENTITY = (
       RESOURCE_ID = '/subscriptions/.../providers/Microsoft.Databricks/accessConnectors/db-access-connector-dev'
     )
   )
   COMMENT 'Managed Identity credential for Dev ADLS';
   ```
4. **Create External Location:**
   ```sql
   CREATE EXTERNAL LOCATION curated_dev_lake
   URL 'abfss://curated@adlsdev.dfs.core.windows.net/'
   WITH (STORAGE CREDENTIAL dev_adls_credential);
   ```
5. **Grant Granular Permissions to Groups/Roles:**
   ```sql
   GRANT READ, WRITE ON EXTERNAL LOCATION curated_dev_lake TO `data-engineers-dev`;
   ```

---

## 3. Pillar 2: Azure Key Vault Secret Scopes (For Legacy/Non-UC Connectors)

For legacy clusters or external API connections, secure credentials via **Azure Key Vault**:

1. Store secrets in Azure Key Vault (e.g., `prod-service-principal-secret`).
2. Create an Azure Key Vault-backed Secret Scope in Databricks CLI / UI:
   ```bash
   databricks secrets create-scope \
     --scope amex-keyvault-scope \
     --scope-backend-type AZURE_KEYVAULT \
     --resource-id "/subscriptions/.../vaults/kv-amex-prod" \
     --dns-name "https://kv-amex-prod.vault.azure.net/"
   ```
3. Retrieve dynamically in code without leaking plain text:
   ```python
   client_secret = dbutils.secrets.get(scope="amex-keyvault-scope", key="sp-secret")
   ```

---

## 4. Pillar 3: Multi-Environment Isolation & Governance

```
                    ┌──> Dev Catalog  ──> points to dev_storage_account
Unity Catalog Metastore ──┼──> Test Catalog ──> points to test_storage_account
                    └──> Prod Catalog ──> points to prod_storage_account (Restricted RBAC)
```

- **Environment Isolation:** Dev, Test, and Prod workspaces bind to their respective storage accounts.
- **Table-Level & Column-Level Access Control:**
  ```sql
  -- Restrict column access (e.g. Cardholder PII) to Authorized Risk Team only
  ALTER TABLE prod.finance.cardholders 
  ALTER COLUMN pan_card_number SET MASK pan_mask_function;
  
  -- Row-level security filtering by region
  ALTER TABLE prod.finance.transactions 
  SET ROW FILTER region_filter_function ON (region);
  ```
