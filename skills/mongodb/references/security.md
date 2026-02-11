# MongoDB Security Reference

## Authentication Mechanism Selection

| Mechanism | Edition | Choose When |
|-----------|---------|-------------|
| **SCRAM-SHA-256** | Community+ | Default for dev/staging. Simple username/password. Always SHA-256 over SHA-1. |
| **x.509** | Community+ | Mutual TLS environments. Best for service-to-service auth and replica set internal auth. |
| **LDAP** | Enterprise/Atlas | Corporate AD/OpenLDAP integration. **Deprecated in 8.0** — migrate to OIDC. |
| **Kerberos** | Enterprise/Atlas | Existing Kerberos/SSO infrastructure. Short-lived tickets. |
| **AWS IAM** | Atlas | AWS-hosted apps. No MongoDB passwords — uses IAM users/roles directly. |
| **OIDC** | Enterprise 7.0+/Atlas | SSO via Okta, Azure AD, etc. Modern replacement for LDAP. |

**Decision shortcut**: Dev → SCRAM. Atlas on AWS → IAM. Enterprise SSO → OIDC. Service accounts → x.509.

## Built-in Roles

### Database-Level

| Role | Grants | When to Use |
|------|--------|-------------|
| `read` | Read all non-system collections | Reporting, analytics, read-only agents |
| `readWrite` | Read + insert/update/remove | Application CRUD service accounts |
| `dbAdmin` | Schema ops, indexing, stats. **No data read.** | DBA index management, validation |
| `userAdmin` | Create/modify users and roles | Per-DB access control management |

### Cluster-Level

| Role | Grants | When to Use |
|------|--------|-------------|
| `clusterAdmin` | Full cluster operations | Replica set/sharding administration |
| `readAnyDatabase` | `read` on all DBs (except local/config) | Cross-DB monitoring |
| `root` | **Everything** | Emergency only. **Never for apps or agents.** |

### Custom Roles — When Built-ins Are Wrong

```javascript
db.createRole({
  role: "agentOrderWriter",
  privileges: [
    { resource: { db: "shop", collection: "orders" }, actions: ["find", "insert", "update"] },
    { resource: { db: "shop", collection: "products" }, actions: ["find"] }
  ],
  roles: []
})
```

**Rule**: If an agent only touches 2 collections, it gets exactly 2 collection privileges. No inherited roles.

Discover available actions: `db.adminCommand({listActionTypes: 1})`. Roles are defined on a specific DB — use `admin` DB for cluster-wide custom roles.

## Connection String Security

### Patterns

```
# Atlas (SRV — always use this)
mongodb+srv://user:pass@cluster0.example.mongodb.net/mydb?tls=true&authSource=admin

# Self-managed with TLS
mongodb://user:pass@host:27017/mydb?tls=true&tlsCAFile=/path/ca.pem&authSource=admin
```

### Rules

- **Always SRV** on Atlas — enables auto TLS + DNS-based host discovery.
- **Always `tls=true`** — Atlas enforces; self-managed must set `--tlsMode requireTLS`.
- **`authSource`**: `admin` for SCRAM, `$external` for LDAP/x.509/Kerberos/OIDC.
- **Special chars in passwords**: `encodeURIComponent()` before embedding in URI.
- **Never `tlsAllowInvalidCertificates: true`** in production — enables MITM.

### Credential Handling

```
# CORRECT — environment variable
export MONGODB_URI="mongodb+srv://..."
client = MongoClient(os.environ["MONGODB_URI"])

# CORRECT — secrets manager
uri = secrets_manager.get_secret("mongodb-uri")

# WRONG — hardcoded
client = MongoClient("mongodb+srv://admin:P@ssw0rd@...")
```

Use AWS Secrets Manager, HashiCorp Vault, or cloud IAM. Rotate credentials on a schedule.

## Client-Side Encryption Comparison

### Automatic vs Explicit CSFLE

| Mode | How It Works | Version |
|------|-------------|---------|
| **Automatic** | Encrypted fields defined in JSON schema; driver handles transparently | 4.2+ |
| **Explicit** | Manual `encrypt()`/`decrypt()` calls per field; full control | 4.0+ |

### CSFLE vs Queryable Encryption

| Feature | CSFLE (4.2+) | Queryable Encryption (7.0+) |
|---------|-------------|----------------------------|
| Equality queries | Deterministic encryption only | Yes (fully random) |
| Range queries | No | Yes |
| Encryption type | Deterministic or Random | Always random |
| Pattern leakage | Deterministic leaks equality patterns | None |
| Server sees plaintext | Never | Never |
| Edition | Enterprise / Atlas | Enterprise / Atlas |

**Decision**: Use **Queryable Encryption** on MongoDB 7.0+. Fall back to **CSFLE** on older versions.

### What to Encrypt

PII (SSN, DOB, email, phone), financial data (card numbers, bank accounts), health records, stored tokens/API keys.

### Key Architecture

- **Customer Master Key (CMK)** → external KMS (AWS KMS, Azure Key Vault, GCP KMS)
- **Data Encryption Keys (DEKs)** → stored in MongoDB key vault collection, encrypted by CMK
- **Local master key** → dev/test only, **never production**

## Network Security Checklist

```
[ ] IP allowlist configured (never 0.0.0.0/0 in production)
[ ] Private endpoints (AWS PrivateLink / Azure PL / GCP PSC) for production
[ ] VPC peering as alternative to private endpoints
[ ] Self-managed: bind to specific interfaces, not 0.0.0.0
[ ] Firewall rules restrict port 27017 to app servers only
```

**Preference order**: Private endpoints > VPC peering > IP allowlisting.

## Encryption at Rest

- **Atlas**: enabled by default. Customer-managed keys (CMK) available for compliance.
- **Enterprise self-managed**: WiredTiger encrypted storage engine (AES256-CBC default, AES256-GCM Linux only) + KMIP integration.
- **Community**: use volume-level encryption (LUKS, EBS encryption).

## Audit Logging

Minimum audit scope: auth events + DDL (schema changes). Enable CRUD auditing selectively on sensitive collections only (high volume impact). Atlas: use audit log download via API and integrate with SIEM.

```yaml
auditLog:
  destination: file
  format: JSON
  path: /var/log/mongodb/audit.json
  filter: '{ atype: { $in: ["authenticate", "createUser", "dropDatabase"] } }'
```

## Common Mistakes

| Mistake | Impact | Fix |
|---------|--------|-----|
| Hardcoded credentials in source | Leak via git/logs | Env vars, secrets manager, IAM |
| `root` role for app accounts | Full cluster compromise | Scoped `readWrite` or custom role |
| TLS disabled | Plaintext credentials on wire | `tls=true`; `--tlsMode requireTLS` |
| `0.0.0.0/0` allowlist | DB exposed to internet | Specific CIDRs; private endpoints |
| No auth on self-managed | Anyone with network access = admin | `--auth` flag or `security.authorization: enabled` |
| `tlsAllowInvalidCertificates: true` | MITM attacks | Always validate certs in production |
| Stale credentials never rotated | Increased blast radius | Rotate; prefer short-lived tokens (IAM/OIDC) |
| Sensitive fields stored unencrypted | PII exposed in breach | CSFLE or Queryable Encryption |
| Overly broad network peering | Lateral movement risk | Least-privilege network access; segment VPCs |

## Quick Security Checklist

```
[ ] Auth enabled (SCRAM-SHA-256 minimum)
[ ] TLS enforced on all connections
[ ] Credentials in secrets manager (never in code)
[ ] Least-privilege roles (no root for apps)
[ ] IP allowlist or private endpoints configured
[ ] Encryption at rest enabled
[ ] CSFLE/Queryable Encryption for PII fields
[ ] Audit logging on for auth + DDL events
[ ] Credential rotation scheduled
[ ] Connection string uses SRV + authSource
```
