# Secure Secret Management in Azure: Azure Key Vault & Databricks Secret Scopes

## Question
How do you securely manage database connection strings, API keys, and service principal secrets in Azure Data Factory and Azure Databricks without hardcoding credentials?

---

## 1. Architectural Security Blueprint

```
[ Azure Key Vault (`kv-citi-prod`) ]
  ├── Stores: DB Passwords, API Tokens, Private Keys
  └── Secured via: Private Endpoints, Azure RBAC, Zero Public IP Access
         │
         ├──> [ Azure Data Factory ] ── (Referenced via Managed Identity Linked Service)
         │
         └──> [ Azure Databricks ]   ── (Referenced via AKV-Backed Secret Scope)
```

---

## 2. Step-by-Step Configuration

### Step 1: Azure Key Vault-Backed Secret Scope in Databricks
Create the secret scope via Databricks CLI or UI:
```bash
databricks secrets create-scope \
  --scope citi-akv-scope \
  --scope-backend-type AZURE_KEYVAULT \
  --resource-id "/subscriptions/.../vaults/kv-citi-prod" \
  --dns-name "https://kv-citi-prod.vault.azure.net/"
```

### Step 2: Retrieve Secrets Dynamically in PySpark Code
```python
# Credentials are never exposed in plain text or logged in Spark UI
db_password = dbutils.secrets.get(scope="citi-akv-scope", key="oracle-prod-password")
db_user = dbutils.secrets.get(scope="citi-akv-scope", key="oracle-prod-username")

jdbc_url = f"jdbc:oracle:thin:@//db-prod.internal.net:1521/PROD_DW"
df = spark.read.format("jdbc") \
    .option("url", jdbc_url) \
    .option("user", db_user) \
    .option("password", db_password) \
    .option("dbtable", "TRADING.SETTLEMENTS") \
    .load()
```

---

## 3. Best Practice: Eliminate Secrets with Azure Managed Identities & Unity Catalog
- Replace service principal secrets entirely with **Azure Managed Identities** (Access Connectors) assigned directly to Databricks Unity Catalog Storage Credentials.
