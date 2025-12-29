# Security Model

**Threat modeling, encryption, and security hardening for AnyFS deployments**

This guide covers security considerations for deploying AnyFS-based filesystems, from single-user local use to multi-tenant cloud services.

---

## Threat Model

### Actors

| Actor | Description | Trust Level |
|-------|-------------|-------------|
| **User** | Legitimate filesystem user | Trusted for their data |
| **Other User** | Another tenant (multi-tenant) | Untrusted (isolation required) |
| **Operator** | System administrator | Trusted for ops, not data |
| **Attacker** | External malicious actor | Untrusted |
| **Compromised Host** | Server with attacker access | Assume worst case |

### Assets to Protect

| Asset | Confidentiality | Integrity | Availability |
|-------|----------------|-----------|--------------|
| File contents | High | High | High |
| File metadata (names, sizes) | Medium | High | High |
| Directory structure | Medium | High | Medium |
| Encryption keys | Critical | Critical | High |
| Audit logs | Medium | Critical | Medium |
| User credentials | Critical | Critical | High |

### Attack Vectors

| Vector | Mitigation |
|--------|------------|
| Network interception | TLS for all traffic |
| Unauthorized access | Authentication + authorization |
| Data theft (at rest) | Encryption (SQLCipher) |
| Data theft (in memory) | Memory protection, key isolation |
| Tenant data leakage | Strict isolation, no cross-tenant dedup |
| Path traversal | PathFilter middleware, input validation |
| Denial of service | Rate limiting, quotas |
| Privilege escalation | Principle of least privilege |
| Audit tampering | Append-only logs, signatures |

---

## Encryption at Rest

### SQLCipher Integration

For encrypted SQLite backends, use SQLCipher:

```rust
use rusqlite::Connection;

pub struct EncryptedSqliteBackend {
    conn: Connection,
}

impl EncryptedSqliteBackend {
    /// Open an encrypted database.
    ///
    /// # Security Notes
    /// - Key should be 256 bits of cryptographically random data
    /// - Or use a strong passphrase with proper key derivation
    pub fn open(path: &Path, key: &EncryptionKey) -> Result<Self, FsError> {
        let conn = Connection::open(path)
            .map_err(|e| FsError::Backend(e.to_string()))?;

        // Apply encryption key (MUST be first operation)
        match key {
            EncryptionKey::Raw(bytes) => {
                // Raw 256-bit key (hex encoded for SQLCipher)
                let hex_key = hex::encode(bytes);
                conn.execute_batch(&format!("PRAGMA key = \"x'{}'\";", hex_key))?;
            }
            EncryptionKey::Passphrase(pass) => {
                // Passphrase (SQLCipher uses PBKDF2 internally)
                conn.execute_batch(&format!("PRAGMA key = '{}';", escape_sql(pass)))?;
            }
        }

        // Verify encryption is working
        conn.execute_batch("SELECT count(*) FROM sqlite_master;")
            .map_err(|_| FsError::Backend("Invalid encryption key".into()))?;

        // Configure after key is set
        conn.execute_batch("
            PRAGMA journal_mode = WAL;
            PRAGMA synchronous = NORMAL;
        ")?;

        Ok(Self { conn })
    }
}

pub enum EncryptionKey {
    /// Raw 256-bit key (32 bytes)
    Raw([u8; 32]),
    /// Passphrase (key derived via PBKDF2)
    Passphrase(String),
}
```

### Key Derivation

For passphrase-based keys, SQLCipher uses PBKDF2 internally. For custom key derivation:

```rust
use argon2::{Argon2, password_hash::SaltString};
use rand::rngs::OsRng;

/// Derive a 256-bit key from a passphrase.
pub fn derive_key(passphrase: &str, salt: &[u8]) -> [u8; 32] {
    let argon2 = Argon2::default();
    let mut key = [0u8; 32];

    argon2.hash_password_into(
        passphrase.as_bytes(),
        salt,
        &mut key,
    ).expect("key derivation failed");

    key
}

/// Generate a random salt for key derivation.
pub fn generate_salt() -> [u8; 16] {
    let mut salt = [0u8; 16];
    OsRng.fill_bytes(&mut salt);
    salt
}
```

**Salt storage:** Store salt separately from encrypted data (e.g., in a key management service or config file).

---

## Key Management

### Key Lifecycle

```
┌─────────┐     ┌─────────┐     ┌─────────┐     ┌─────────┐
│ Generate│ ──→ │  Store  │ ──→ │   Use   │ ──→ │ Rotate  │
└─────────┘     └─────────┘     └─────────┘     └─────────┘
                                                     │
                                                     ▼
                                               ┌─────────┐
                                               │ Destroy │
                                               └─────────┘
```

### Key Generation

```rust
use rand::rngs::OsRng;
use rand::RngCore;

/// Generate a cryptographically secure 256-bit key.
pub fn generate_key() -> [u8; 32] {
    let mut key = [0u8; 32];
    OsRng.fill_bytes(&mut key);
    key
}

/// Generate a key ID for tracking.
pub fn generate_key_id() -> String {
    let mut id = [0u8; 16];
    OsRng.fill_bytes(&mut id);
    format!("key_{}", hex::encode(id))
}
```

### Key Storage

**Never store keys:**
- In source code
- In plain text config files
- In the same location as encrypted data
- In environment variables (visible in process lists)

**Recommended storage:**

| Environment | Solution |
|-------------|----------|
| Development | File with restricted permissions (0600) |
| Production (cloud) | KMS (AWS KMS, GCP KMS, Azure Key Vault) |
| Production (on-prem) | HSM or dedicated secrets manager |
| User devices | OS keychain (macOS Keychain, Windows Credential Manager) |

```rust
/// Key storage abstraction.
pub trait KeyStore: Send + Sync {
    /// Retrieve a key by ID.
    fn get_key(&self, key_id: &str) -> Result<[u8; 32], KeyError>;

    /// Store a new key, returns key ID.
    fn store_key(&self, key: &[u8; 32]) -> Result<String, KeyError>;

    /// Delete a key (after rotation).
    fn delete_key(&self, key_id: &str) -> Result<(), KeyError>;

    /// List all key IDs.
    fn list_keys(&self) -> Result<Vec<String>, KeyError>;
}

/// AWS KMS implementation.
pub struct AwsKmsKeyStore {
    client: aws_sdk_kms::Client,
    master_key_id: String,
}

impl KeyStore for AwsKmsKeyStore {
    fn get_key(&self, key_id: &str) -> Result<[u8; 32], KeyError> {
        // Keys are stored encrypted in DynamoDB/S3
        // Decrypt using KMS
        let encrypted = self.fetch_encrypted_key(key_id)?;

        let decrypted = self.client.decrypt()
            .key_id(&self.master_key_id)
            .ciphertext_blob(Blob::new(encrypted))
            .send()
            .await?;

        let plaintext = decrypted.plaintext().unwrap();
        let mut key = [0u8; 32];
        key.copy_from_slice(&plaintext.as_ref()[..32]);
        Ok(key)
    }

    // ... other methods
}
```

### Key Rotation

Regular key rotation limits damage from key compromise:

```rust
impl EncryptedSqliteBackend {
    /// Rotate encryption key.
    ///
    /// This re-encrypts the entire database with a new key.
    /// Can take a long time for large databases.
    pub fn rotate_key(&self, new_key: &EncryptionKey) -> Result<(), FsError> {
        let new_key_sql = match new_key {
            EncryptionKey::Raw(bytes) => format!("\"x'{}'\"", hex::encode(bytes)),
            EncryptionKey::Passphrase(pass) => format!("'{}'", escape_sql(pass)),
        };

        // SQLCipher's PRAGMA rekey re-encrypts the database
        self.conn.execute_batch(&format!("PRAGMA rekey = {};", new_key_sql))
            .map_err(|e| FsError::Backend(format!("key rotation failed: {}", e)))?;

        Ok(())
    }
}

/// Key rotation schedule.
pub struct KeyRotationPolicy {
    /// Maximum age of a key before rotation.
    pub max_key_age: Duration,
    /// Maximum amount of data encrypted with one key.
    pub max_data_encrypted: u64,
    /// Whether to auto-rotate.
    pub auto_rotate: bool,
}

impl Default for KeyRotationPolicy {
    fn default() -> Self {
        Self {
            max_key_age: Duration::from_secs(90 * 24 * 60 * 60),  // 90 days
            max_data_encrypted: 1024 * 1024 * 1024 * 100,         // 100 GB
            auto_rotate: true,
        }
    }
}
```

**Rotation workflow:**

1. Generate new key
2. Store new key in key store
3. Re-encrypt database with `PRAGMA rekey`
4. Update key ID reference
5. Audit log the rotation
6. After retention period, delete old key

---

## Multi-Tenant Isolation

### Isolation Strategies

| Strategy | Isolation Level | Complexity | Use Case |
|----------|-----------------|------------|----------|
| **Separate databases** | Strongest | Low | Few large tenants |
| **Separate tables** | Strong | Medium | Many small tenants |
| **Row-level** | Moderate | High | Shared infrastructure |

**Recommendation:** Separate databases (one SQLite file per tenant).

### Per-Tenant Keys

Each tenant should have their own encryption key:

```rust
pub struct MultiTenantBackend {
    key_store: Arc<dyn KeyStore>,
    tenant_backends: RwLock<HashMap<TenantId, Arc<EncryptedSqliteBackend>>>,
}

impl MultiTenantBackend {
    /// Get or create backend for a tenant.
    pub fn get_tenant(&self, tenant_id: &TenantId) -> Result<Arc<EncryptedSqliteBackend>, FsError> {
        // Check cache
        {
            let backends = self.tenant_backends.read().unwrap();
            if let Some(backend) = backends.get(tenant_id) {
                return Ok(backend.clone());
            }
        }

        // Create new backend
        let key = self.key_store.get_key(&tenant_id.key_id())?;
        let path = self.tenant_db_path(tenant_id);
        let backend = Arc::new(EncryptedSqliteBackend::open(&path, &EncryptionKey::Raw(key))?);

        // Cache it
        let mut backends = self.tenant_backends.write().unwrap();
        backends.insert(tenant_id.clone(), backend.clone());

        Ok(backend)
    }
}
```

### Cross-Tenant Dedup Considerations

**Warning:** Cross-tenant deduplication can leak information.

```
Tenant A uploads secret.pdf (hash: abc123)
Tenant B uploads same file → instantly deduped → B knows A has that file
```

**Options:**

| Approach | Dedup Savings | Privacy |
|----------|---------------|---------|
| No cross-tenant dedup | None | Full privacy |
| Convergent encryption | Partial | Leaks file existence |
| Per-tenant keys before hash | None | Full privacy |

**Recommendation:** Only deduplicate within a tenant, not across tenants.

```rust
impl HybridBackend {
    fn blob_id_for_tenant(&self, tenant_id: &TenantId, data: &[u8]) -> String {
        // Include tenant ID in hash to prevent cross-tenant dedup
        let mut hasher = Sha256::new();
        hasher.update(tenant_id.as_bytes());
        hasher.update(data);
        hex::encode(hasher.finalize())
    }
}
```

---

## Audit Logging

### What to Log

| Event | Severity | Data to Capture |
|-------|----------|-----------------|
| File read | Info | path, user, timestamp, size |
| File write | Info | path, user, timestamp, size, hash |
| File delete | Warning | path, user, timestamp |
| Permission change | Warning | path, user, old/new perms |
| Login success | Info | user, IP, timestamp |
| Login failure | Warning | user, IP, timestamp, reason |
| Key rotation | Critical | key_id, user, timestamp |
| Admin action | Critical | action, user, timestamp |

### Audit Log Schema

```sql
CREATE TABLE audit_log (
    seq         INTEGER PRIMARY KEY AUTOINCREMENT,
    timestamp   INTEGER NOT NULL,
    event_type  TEXT NOT NULL,
    severity    TEXT NOT NULL,  -- 'info', 'warning', 'critical'
    actor       TEXT,           -- user ID or 'system'
    actor_ip    TEXT,
    resource    TEXT,           -- path or resource ID
    action      TEXT NOT NULL,
    details     TEXT,           -- JSON
    signature   BLOB            -- HMAC for tamper detection
);

CREATE INDEX idx_audit_timestamp ON audit_log(timestamp);
CREATE INDEX idx_audit_actor ON audit_log(actor);
CREATE INDEX idx_audit_resource ON audit_log(resource);
```

### Tamper-Evident Logging

Sign audit entries to detect tampering:

```rust
use hmac::{Hmac, Mac};
use sha2::Sha256;

type HmacSha256 = Hmac<Sha256>;

pub struct AuditLogger {
    conn: Connection,
    signing_key: [u8; 32],
    prev_signature: RwLock<Vec<u8>>,  // Chain signatures
}

impl AuditLogger {
    pub fn log(&self, event: AuditEvent) -> Result<(), FsError> {
        let timestamp = SystemTime::now()
            .duration_since(UNIX_EPOCH)
            .unwrap()
            .as_secs() as i64;

        let details = serde_json::to_string(&event.details)?;

        // Create signature (includes previous signature for chaining)
        let prev_sig = self.prev_signature.read().unwrap().clone();
        let signature = self.sign_entry(timestamp, &event, &details, &prev_sig);

        self.conn.execute(
            "INSERT INTO audit_log (timestamp, event_type, severity, actor, actor_ip, resource, action, details, signature)
             VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?)",
            params![
                timestamp,
                event.event_type,
                event.severity,
                event.actor,
                event.actor_ip,
                event.resource,
                event.action,
                details,
                &signature[..],
            ],
        )?;

        // Update chain
        *self.prev_signature.write().unwrap() = signature;

        Ok(())
    }

    fn sign_entry(
        &self,
        timestamp: i64,
        event: &AuditEvent,
        details: &str,
        prev_sig: &[u8],
    ) -> Vec<u8> {
        let mut mac = HmacSha256::new_from_slice(&self.signing_key).unwrap();

        mac.update(&timestamp.to_le_bytes());
        mac.update(event.event_type.as_bytes());
        mac.update(event.action.as_bytes());
        mac.update(details.as_bytes());
        mac.update(prev_sig);  // Chain to previous entry

        mac.finalize().into_bytes().to_vec()
    }

    /// Verify audit log integrity.
    pub fn verify_integrity(&self) -> Result<bool, FsError> {
        let mut prev_sig = Vec::new();

        let mut stmt = self.conn.prepare(
            "SELECT timestamp, event_type, severity, actor, actor_ip, resource, action, details, signature
             FROM audit_log ORDER BY seq"
        )?;

        let rows = stmt.query_map([], |row| {
            Ok(AuditRow {
                timestamp: row.get(0)?,
                event_type: row.get(1)?,
                severity: row.get(2)?,
                actor: row.get(3)?,
                actor_ip: row.get(4)?,
                resource: row.get(5)?,
                action: row.get(6)?,
                details: row.get(7)?,
                signature: row.get(8)?,
            })
        })?;

        for row in rows {
            let row = row?;

            let expected_sig = self.sign_entry(
                row.timestamp,
                &row.to_event(),
                &row.details,
                &prev_sig,
            );

            if expected_sig != row.signature {
                return Ok(false);  // Tampered!
            }

            prev_sig = row.signature;
        }

        Ok(true)
    }
}
```

### Audit Log Retention

```rust
impl AuditLogger {
    /// Rotate old audit logs to cold storage.
    pub fn rotate(&self, max_age: Duration) -> Result<RotationStats, FsError> {
        let cutoff = SystemTime::now()
            .duration_since(UNIX_EPOCH)
            .unwrap()
            .as_secs() as i64 - max_age.as_secs() as i64;

        // Export old entries to archive
        let old_entries: Vec<AuditRow> = self.conn.prepare(
            "SELECT * FROM audit_log WHERE timestamp < ?"
        )?.query_map([cutoff], |row| /* ... */)?.collect();

        // Write to archive file (compressed, signed)
        self.write_archive(&old_entries)?;

        // Delete from active log
        let deleted = self.conn.execute(
            "DELETE FROM audit_log WHERE timestamp < ?",
            [cutoff],
        )?;

        Ok(RotationStats { archived: old_entries.len(), deleted })
    }
}
```

---

## Access Control

### Path-Based Access Control

Use `PathFilterLayer` middleware for path-based restrictions:

```rust
let backend = SqliteBackend::open("data.db")?
    .layer(PathFilterLayer::builder()
        // Allow specific directories
        .allow("/home/{user}/**")
        .allow("/shared/**")
        // Block sensitive paths
        .deny("**/.env")
        .deny("**/.git/**")
        .deny("**/node_modules/**")
        // Block by extension
        .deny("**/*.key")
        .deny("**/*.pem")
        .build());
```

### Role-Based Access Control

Implement RBAC at the application layer:

```rust
#[derive(Debug, Clone)]
pub enum Role {
    Admin,
    ReadWrite,
    ReadOnly,
    Custom(Vec<Permission>),
}

#[derive(Debug, Clone)]
pub enum Permission {
    Read(PathPattern),
    Write(PathPattern),
    Delete(PathPattern),
    Admin,
}

pub struct RbacMiddleware<B> {
    inner: B,
    user_roles: Arc<dyn RoleProvider>,
}

impl<B: FsRead> FsRead for RbacMiddleware<B> {
    fn read(&self, path: impl AsRef<Path>) -> Result<Vec<u8>, FsError> {
        let path = path.as_ref();
        let user = current_user()?;
        let role = self.user_roles.get_role(&user)?;

        if !role.can_read(path) {
            return Err(FsError::AccessDenied {
                path: path.to_path_buf(),
                reason: "insufficient permissions".into(),
            });
        }

        self.inner.read(path)
    }
}

impl<B: FsWrite> FsWrite for RbacMiddleware<B> {
    fn write(&self, path: impl AsRef<Path>, data: &[u8]) -> Result<(), FsError> {
        let path = path.as_ref();
        let user = current_user()?;
        let role = self.user_roles.get_role(&user)?;

        if !role.can_write(path) {
            return Err(FsError::AccessDenied {
                path: path.to_path_buf(),
                reason: "write permission denied".into(),
            });
        }

        self.inner.write(path, data)
    }
}
```

---

## Network Security

### TLS Configuration

Always use TLS for network communication:

```rust
use tonic::transport::{Server, ServerTlsConfig, Identity, Certificate};

pub async fn serve_with_tls(
    backend: impl Fs,
    addr: &str,
    cert_path: &Path,
    key_path: &Path,
) -> Result<(), Box<dyn Error>> {
    let cert = std::fs::read_to_string(cert_path)?;
    let key = std::fs::read_to_string(key_path)?;

    let identity = Identity::from_pem(cert, key);

    let tls_config = ServerTlsConfig::new()
        .identity(identity);

    Server::builder()
        .tls_config(tls_config)?
        .add_service(FsServiceServer::new(FsServer::new(backend)))
        .serve(addr.parse()?)
        .await?;

    Ok(())
}
```

### Client Certificate Authentication (mTLS)

For high-security deployments, require client certificates:

```rust
use tonic::transport::ClientTlsConfig;

pub async fn connect_with_mtls(
    addr: &str,
    ca_cert: &Path,
    client_cert: &Path,
    client_key: &Path,
) -> Result<FsServiceClient<Channel>, Box<dyn Error>> {
    let ca = std::fs::read_to_string(ca_cert)?;
    let cert = std::fs::read_to_string(client_cert)?;
    let key = std::fs::read_to_string(client_key)?;

    let tls_config = ClientTlsConfig::new()
        .ca_certificate(Certificate::from_pem(ca))
        .identity(Identity::from_pem(cert, key));

    let channel = Channel::from_shared(addr.to_string())?
        .tls_config(tls_config)?
        .connect()
        .await?;

    Ok(FsServiceClient::new(channel))
}
```

---

## Security Checklist

### Development

- [ ] No secrets in source code
- [ ] No secrets in logs
- [ ] Input validation on all paths
- [ ] Error messages don't leak sensitive info

### Deployment

- [ ] TLS enabled for all network traffic
- [ ] Encryption at rest (SQLCipher)
- [ ] Keys stored in secure key management system
- [ ] Key rotation policy defined and automated
- [ ] Audit logging enabled
- [ ] Rate limiting configured
- [ ] Quotas configured

### Operations

- [ ] Regular security audits
- [ ] Vulnerability scanning
- [ ] Audit log review
- [ ] Key rotation executed
- [ ] Backup encryption verified
- [ ] Access reviews (who has what permissions)

### Multi-Tenant

- [ ] Tenant isolation verified
- [ ] Per-tenant encryption keys
- [ ] No cross-tenant dedup (or risk accepted)
- [ ] Tenant data segregation in backups

---

## Summary

| Layer | Protection |
|-------|------------|
| **Transport** | TLS, mTLS |
| **Authentication** | Tokens, certificates |
| **Authorization** | RBAC, PathFilter |
| **Data at rest** | SQLCipher encryption |
| **Key management** | KMS, rotation |
| **Audit** | Tamper-evident logging |
| **Isolation** | Per-tenant DBs and keys |

Security is not optional. Build it in from the start.
