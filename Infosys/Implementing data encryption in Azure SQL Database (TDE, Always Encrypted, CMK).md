Data encryption in Azure SQL Database operates on two complementary levels: **Transparent Data Encryption (TDE)** protects data at rest, while **Always Encrypted** protects data throughout its lifecycle, including during query processing. Customer-managed keys (CMK) can be layered onto TDE to give you control over the key lifecycle. The three mechanisms solve different problems and are not mutually exclusive—most production deployments combine TDE as a baseline with Always Encrypted for the most sensitive columns.

---

## Understand the encryption layers

Azure SQL offers three encryption layers, each targeting a different threat model:

| Layer | What it protects | Application changes | Protects against DBAs? |
|---|---|---|---|
| **TDE** | Database files, transaction logs, backups | None | No |
| **Always Encrypted** | Data in transit, at rest, and during computation | Client driver required | Yes |
| **Column-level encryption (T-SQL)** | Specific columns you explicitly encrypt/decrypt | Application code manages keys | No |

TDE encrypts the physical files on disk and is transparent to applications. Always Encrypted keeps encryption keys entirely outside the database engine, so the database never sees plaintext. Column-level encryption via `ENCRYPTBYKEY` gives granular control but requires explicit key management in your code.

---

## Transparent Data Encryption (TDE)

TDE is **enabled by default** on all Azure SQL databases. It encrypts database files, transaction logs, and backups in real time using AES-256 in cipher block chaining (CBC) mode.

### How TDE works

TDE uses a two-key hierarchy:

1. **Database Encryption Key (DEK)** — a symmetric key that encrypts the database.
2. **TDE protector** — an asymmetric key (or certificate) that encrypts the DEK. By default, this is a service-managed certificate stored in the master database.

The server encrypts the DEK with the TDE protector and stores the encrypted DEK in the user database. When decryption is needed, the server sends the protected DEK to the protector for unwrapping.

### Performance impact

TDE produces a CPU overhead despite hardware acceleration via Intel AES-NI. Some DTUs or vCores are consumed for encryption and decryption. For large databases, scaling up the compute tier can neutralize the impact and speed up the initial encryption process. In steady state, the overhead is typically modest for most workloads.

### When to use TDE

TDE is the right choice when you need **compliance-driven at-rest encryption** without modifying application code. It does not protect data from users who can connect to the database with valid permissions—that requires Always Encrypted.

---

## Customer-managed keys (CMK) for TDE

By default, Azure manages the TDE protector. With CMK, you supply your own asymmetric key stored in **Azure Key Vault** or **Azure Managed HSM**, enabling Bring Your Own Key (BYOK) scenarios and separation of duties between key administrators and data administrators.

### Key requirements

The TDE protector must be an **asymmetric RSA key** with a length of **2,048 or 3,072 bits**, in an **enabled** state, with a past activation date and a future expiration date if those fields are set. The key vault must have **soft delete and purge protection enabled**, or TDE protector configuration will fail.

### Access model: managed identity

The Azure SQL logical server accesses the key vault using a **managed identity**—either system-assigned or user-assigned. Two access models are available:

- **Azure RBAC (recommended)**: Grant the server identity the **Key Vault Crypto Service Encryption User** role. This provides flexibility and granularity.
- **Vault access policies**: Grant the server identity `get`, `wrapKey`, and `unwrapKey` permissions. Simpler but less flexible.

The managed identity must be granted access **before** enabling TDE with CMK on the server.

### Server-level vs. database-level CMK

For Azure SQL Database, the TDE protector is set at the **server level** and inherited by all encrypted databases on that server. Database-level CMK is also available for Azure SQL Database (all editions) using a user-assigned managed identity, including cross-tenant scenarios, but it is not available for Azure SQL Managed Instance.

### Switching from service-managed to CMK

Switching from a service-managed key to a customer-managed key **does not re-encrypt the database files**. Only the DEK is re-encrypted, which is a fast online operation with no downtime.

### Key rotation

Key rotation can be **manual or automated**. When using automated rotation in Azure Key Vault or Managed HSM, the system uses the latest key version. Ensure that the server's managed identity retains the required permissions on the vault throughout rotation. For geo-replication scenarios, the secondary server's managed identity must also have permissions on the primary server's key vault.

### Best practices for CMK

- Use a **dedicated key vault** for Azure SQL. Do not share it with other services, as Key Vault enforces throttling limits that can affect database performance during failover.
- Limit associations: no more than **500 General Purpose databases** or **200 Business Critical databases** per key vault to avoid throttling.
- Enable **Key Vault auditing** to log key operations for compliance.

---

## Always Encrypted

Always Encrypted ensures that the **database engine never processes plaintext values**. Client applications hold the encryption keys and perform all encryption and decryption. Even a database administrator with high-level access cannot view protected data.

### Architecture: two-key hierarchy

Always Encrypted uses:

1. **Column Master Key (CMK)** — an asymmetric key stored in a trusted key store (Azure Key Vault, Windows Certificate Store, or HSM). The CMK protects one or more column encryption keys.
2. **Column Encryption Key (CEK)** — a symmetric key (AES-256) that encrypts the actual column data. The CEK is stored in the database **in encrypted form**, protected by the CMK.

The client driver retrieves the encrypted CEK from the database, uses the CMK (which never leaves the key store) to decrypt it locally, and then uses the plaintext CEK to encrypt or decrypt column values before sending data to or after receiving data from SQL Database.

### Setting up Always Encrypted

**Step 1: Create the column master key metadata.** The actual key material remains in the key store.

```sql
CREATE COLUMN MASTER KEY MyCMK
WITH (
    KEY_STORE_PROVIDER_NAME = 'AZURE_KEY_VAULT',
    KEY_PATH = 'https://mykeyvault.vault.azure.net/keys/MyCMK/abc123'
);
```

**Step 2: Create the column encryption key**, encrypted by the CMK.

```sql
CREATE COLUMN ENCRYPTION KEY MyCEK
WITH VALUES (
    COLUMN_MASTER_KEY = MyCMK,
    ALGORITHM = 'RSA_OAEP',
    ENCRYPTED_VALUE = 0x01700000016C006F00...
);
```

**Step 3: Define encrypted columns** with the appropriate encryption type.

```sql
CREATE TABLE Employees (
    EmployeeID int PRIMARY KEY,
    SSN char(11) COLLATE Latin1_General_BIN2
        ENCRYPTED WITH (
            ENCRYPTION_TYPE = DETERMINISTIC,
            ALGORITHM = 'AEAD_AES_256_CBC_HMAC_SHA_256',
            COLUMN_ENCRYPTION_KEY = MyCEK
        ),
    Salary money
        ENCRYPTED WITH (
            ENCRYPTION_TYPE = RANDOMIZED,
            ALGORITHM = 'AEAD_AES_256_CBC_HMAC_SHA_256',
            COLUMN_ENCRYPTION_KEY = MyCEK
        )
);
```

### Choosing deterministic vs. randomized encryption

- **Deterministic encryption**: The same plaintext always produces the same ciphertext. Use it when you need **equality comparisons, joins, or `WHERE` clause filtering** on the encrypted column. It leaks frequency information, so it is less secure against inference attacks.
- **Randomized encryption**: The same plaintext produces different ciphertexts each time. Use it for **stronger security** when you do not need equality operations on that column.

### Secure enclaves

Standard Always Encrypted restricts operations on encrypted columns to equality comparisons. **Always Encrypted with secure enclaves** extends functionality to support **range comparisons, pattern matching (`LIKE`), sorting, and other rich computations** while keeping data confidential.

Azure SQL Database supports two enclave technologies:

- **Intel SGX enclaves**: Require the **vCore purchasing model** and **DC-series hardware**. Intel SGX reaches end of support on **October 31, 2027**; after that date, Azure will automatically migrate databases to supported hardware and enable VBS enclaves.
- **VBS enclaves (Virtualization-Based Security)**: Software-based, with no hardware dependency. VBS allows Always Encrypted with secure enclaves to be used across all Azure SQL Database offerings (provisioned or serverless, vCore or DTU). VBS enclaves **do not support attestation**.

### Attestation

**Attestation** is a protocol that proves to the client that the enclave is genuine and running trusted code. It applies **only to Intel SGX enclaves**. VBS enclaves do not support attestation. For SGX, configure **Microsoft Azure Attestation** by creating an attestation provider and setting the recommended attestation policy.

### Client driver requirements

Always Encrypted requires a client driver that supports the feature (e.g., Microsoft .NET Data Provider for SQL Server, JDBC Driver, ODBC Driver, or Go driver). The application must be configured to enable Always Encrypted and, for enclave computations, to enable enclave calculations and attestation.

### Limitations

- Not all SQL operations are supported on encrypted columns, even with enclaves (e.g., certain aggregate functions, `LIKE` patterns with leading wildcards under standard Always Encrypted).
- Performance overhead exists due to client-side encryption/decryption and the inability to use certain indexes on randomized columns.
- Key management complexity increases significantly compared to TDE.

---

## Comparison: TDE vs. Always Encrypted

| Dimension | TDE | Always Encrypted |
|---|---|---|
| **Protects against** | Physical theft of files, backups, disks | DBAs, cloud operators, high-privilege users |
| **Encryption scope** | Entire database files, logs, backups | Specific columns |
| **Application changes** | None | Client driver configuration required |
| **Query capabilities on encrypted data** | Fully transparent; all T-SQL works | Restricted; equality (deterministic) or rich ops with enclaves |
| **Key location** | DEK in database; protector in Azure (service-managed or Key Vault) | CEK in database (encrypted); CMK in external key store |
| **Performance impact** | Modest CPU overhead | Client-side crypto overhead; index limitations |
| **Best for** | Compliance-driven at-rest protection | Protecting data from privileged insiders; end-to-end confidentiality |

The two approaches are **complementary**. TDE provides baseline at-rest encryption for the entire database. Always Encrypted adds column-level protection for the most sensitive fields, ensuring even DBAs cannot see plaintext. You can enable both simultaneously.

---

## Decision guide

**Start with TDE.** It is enabled by default and costs nothing in application changes. It satisfies most regulatory requirements for encryption at rest.

**Add CMK for TDE when** you need separation of duties between key administrators and data administrators, or when regulations require you to own and rotate the keys that protect your data. Use a dedicated key vault, a managed identity with the Key Vault Crypto Service Encryption User role, and automated key rotation.

**Add Always Encrypted when** the threat model includes privileged insiders—DBAs, cloud operators, or anyone who can query the database directly. Use **deterministic encryption** for columns that must support joins or equality filters, and **randomized encryption** for columns where maximum security matters more than query flexibility.

**Add secure enclaves when** you need range queries, sorting, or pattern matching on encrypted columns. Choose **VBS enclaves** for broad hardware compatibility and serverless support; choose **Intel SGX** only if you require hardware-based isolation and are prepared to migrate before October 2027.

---

## Operational checklist

- [ ] Verify TDE is enabled on every database (it is on by default, but confirm it has not been disabled).
- [ ] If using CMK: enable soft delete and purge protection on the key vault before configuring TDE.
- [ ] Grant the server's managed identity the Key Vault Crypto Service Encryption User role (RBAC) or `get`/`wrapKey`/`unwrapKey` (access policy).
- [ ] Use a dedicated key vault for Azure SQL; do not exceed 500 GP or 200 BC databases per vault.
- [ ] Configure key rotation (automated via Key Vault rotation policy, or manual with a documented schedule).
- [ ] For Always Encrypted: store the CMK in Azure Key Vault, not in the database or application config.
- [ ] Choose deterministic encryption only for columns that require equality operations; use randomized elsewhere.
- [ ] If using enclaves: select VBS for new deployments (no attestation needed, broader hardware support); plan SGX migration before October 31, 2027.
- [ ] Ensure client drivers are configured for Always Encrypted and, where applicable, enclave computations and attestation.
- [ ] Enable Key Vault auditing and monitor key operations in Azure Monitor.
