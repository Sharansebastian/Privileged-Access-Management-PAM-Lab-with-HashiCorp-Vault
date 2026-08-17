# HashiCorp Vault PAM Project — Complete Workflow

A full record of everything done in this project, from initial setup through the RBAC/identity extension.

---

## Part 1: Base Project Setup (Course Steps 0–7)

### Step 0 — Install Vault
- Downloaded Vault from `developer.hashicorp.com/vault/install`
- Extracted to `C:\vault` (Windows) or installed via `brew install hashicorp/tap/vault` (Mac)
- Verified with `vault --version`

### Step 1 — (Setup/orientation, part of Step 0 flow)
- Opened a terminal window inside `C:\vault`

### Step 2 — Start Vault in dev/learning mode
```
vault server -dev
```
- Left this terminal window running (the live Vault server)
- Copied the **Root Token** printed in the output (ignored the Unseal Key — dev-mode only)

### Step 3 — Connect to Vault from a second terminal
Opened a **new** terminal window and ran:
```
set VAULT_ADDR=http://127.0.0.1:8200
set VAULT_TOKEN=<root_token>
cd C:\vault
```

### Step 4 — Store a secret
```
vault kv put secret/database-admin username="dbadmin" password="SuperSecretPassword123"
```

### Step 5 — Retrieve the secret
```
vault kv get secret/database-admin
```

### Step 6 — Issue a short-lived JIT token
```
vault token create -ttl=2m
```
- Temporarily switched `VAULT_TOKEN` to this new token, confirmed access worked
- Waited out the 2 minutes, confirmed `vault kv get` then failed (expiration proven)
- Switched `VAULT_TOKEN` back to the Root Token

### Step 7 — Enable audit logging
```
vault audit enable file file_path=C:\vault\vault-audit.log
```
- Re-ran `vault kv get secret/database-admin` to generate a log entry
- Reviewed the log via `notepad C:\vault\vault-audit.log`
- Stopped the server afterward (`Ctrl+C` in the first terminal)

**Also explored:** the Vault Web UI at `http://127.0.0.1:8200`, logging in with the Root Token and browsing to view the stored secret visually.

---

## Part 2: RBAC / Identity Lifecycle Extension (Custom Add-on)

### Goal
Extend the base project into a fuller PAM simulation covering: two roles with different permissions, full token lifecycle (issue → use → expire/revoke → fail), and a complete audit trail.

### Step A — Created two ACL policies
**`admin-policy.hcl`:**
```hcl
path "secret/data/database-admin" {
  capabilities = ["create", "read", "update", "delete"]
}
```

**`operator-policy.hcl`:**
```hcl
path "secret/data/database-admin" {
  capabilities = ["read"]
}
```

Loaded into Vault:
```
vault policy write admin-policy admin-policy.hcl
vault policy write operator-policy operator-policy.hcl
```
*(Also explored creating these via the UI: Access Control → ACL Policies → Create ACL policy.)*

### Step B — Created tokens tied to each policy
```
vault token create -policy="admin-policy" -display-name="admin-user" -ttl=5m
vault token create -policy="operator-policy" -display-name="operator-user" -ttl=2m
```
- Noted that the Vault UI's **Access → Tokens** flow wasn't available in this version — token creation stayed CLI-based
- Copied both raw token values for testing

### Step C — Demonstrated policy differences (least privilege)
- Logged in as **operator** → `vault kv get secret/database-admin` succeeded (read allowed)
- Logged in as **operator** → `vault kv put ...` **failed** with `permission denied` (write blocked, as designed)
- Logged in as **admin** → `vault kv put ...` succeeded (write allowed)

### Step D — Token lifecycle: expiration
- Waited out the TTLs (2m for operator, 5m for admin)
- Attempted to use each token again after expiry → both failed with `permission denied` / `invalid token`
- Attempted `vault token lookup <token>` on an expired token → got `bad token`, confirming Vault fully deletes expired tokens rather than just marking them inactive

### Step E — Token lifecycle: manual revocation
- Created a fresh operator token with a longer TTL (1h) specifically to test revocation independent of natural expiry:
```
vault token create -policy="operator-policy" -display-name="operator-user-2" -ttl=1h
```
- Confirmed it worked initially
- As **Root**, revoked it manually:
```
vault token revoke <token>
```
- Re-tested the same token → failed immediately, proving revocation cuts access instantly regardless of remaining TTL

### Step F — Tested self-revocation vs. cross-revocation (least-privilege check)
- Confirmed any token can revoke **itself** (`vault token revoke -self`) — a built-in Vault capability, not policy-dependent
- Confirmed the **operator token could NOT revoke the admin token** — failed with `permission denied`, proving operator has no authority over other identities' access

### Step G — Explored (and ruled out) the Leases UI section
- Checked **Access Control → Leases** in the UI, found it empty
- Confirmed this is expected: Leases track *dynamic secrets* and certain auth-method logins, not tokens created via `vault token create` (which live in Vault's separate token store)
- Attempted `vault read lease` directly — confirmed this isn't valid syntax; the correct commands are `vault token lookup <token>` (for tokens) or `vault lease lookup <lease_id>` (for actual leases, not used in this project)

### Step H — Reviewed the full audit trail
- Reviewed `vault-audit.log` to confirm it captured: authentication/token creation events, successful secret reads, denied write attempts, and revocation actions — all five categories from the original PAM lifecycle goal

---

## Part 3: Identity Modeling (Entities & Groups)

### Step I — Explored Vault Entities
- Navigated: Access Control → Entities → Create Entity
- Learned an **Entity** represents an actual identity (a person or service), distinct from any single token
- Initially attached a policy (`admin-policy`) directly to an entity, and added metadata (e.g., `role: administrator`, `department: IT`)

### Step J — Corrected the identity model: Entities vs. Groups
- Recognized that naming entities `admin` / `operator` was mixing concepts — realized:
  - **Entities** should be named after actual people/services (e.g., `ram`, `gokul`)
  - **Groups** should represent the roles (`admin`, `operator`) and hold the policies
- Revised plan going forward:
  1. Create entities named after real identities (`ram`, `gokul`)
  2. Create Groups named `admin` and `operator`
  3. Attach `admin-policy` / `operator-policy` to the **groups**, not individual entities
  4. Add entities as members of the appropriate group

### Step K — Considered (and set aside) Entity Aliases
- Explored linking an entity to a real login credential via an **Alias**, which requires the token's **accessor** (not the raw token value)
- Decided aliasing raw CLI-created tokens added complexity without much learning value — aliases are best suited to proper login methods (userpass, LDAP, OIDC)
- Kept entities and tokens conceptually linked (entity = identity/role record, token = credential) rather than formally aliased, for this project's scope

---

## Where the project currently stands
- ✅ Vault server running in dev mode, secret stored and retrievable
- ✅ Two ACL policies enforcing least privilege (admin vs. operator)
- ✅ Full token lifecycle demonstrated: issuance, use, expiration, manual revocation, and failed cross-revocation
- ✅ Audit logging enabled and reviewed, covering all required event types
- 🔄 In progress: proper Entity (person) + Group (role) structure, replacing the earlier direct entity-to-policy attachment

## Suggested next steps
1. Finish creating both entities (`ram`, `gokul`) with descriptive metadata
2. Create the `admin` and `operator` Groups, attach the matching policies, and add each entity as a member
3. Re-verify access behavior now flows through group membership rather than direct entity policies
4. Optionally revisit audit log entries to see how they reference these named identities
