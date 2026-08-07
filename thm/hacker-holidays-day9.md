# THM - Hacker Holidays Day 9: CryptoCabana
**Category:** Cloud
**Difficulty:** Medium

## Tools Used
- Browser
- Azure CLI (Cloud Shell)
- curl

## Steps

1. Logged into Azure Portal with provided credentials (Username + Temporary Access Pass)
2. Opened Azure Cloud Shell (Bash), selected Az-Subs-CTF subscription, verified access: `az account show`
3. Opened `https://cryptocabanaf5scjagc.z13.web.core.windows.net/` — found seed phrase backup kiosk
4. Fetched `app.js`: `curl https://cryptocabanaf5scjagc.z13.web.core.windows.net/app.js` — found hardcoded storage account name, container name, and SAS token
5. Listed all containers using the leaked SAS token:
   `az storage container list --account-name cryptocabanaf5scjagc --sas-token "<SAS>" --query "[].name" -o tsv`
   — found `$web`, `backups`, `vault`
6. Listed blobs in `vault` container — found `backup-service-account.json` and `seed_phrase.txt`
7. Downloaded `backup-service-account.json` — found exposed service principal credentials: `client_id`, `client_secret`, `tenant_id`, and `key_vault_name: ccabana-kv-f5scjagc`
8. Authenticated as the service principal:
   `az login --service-principal --username <client_id> --password <client_secret> --tenant <tenant_id>`
9. Listed Key Vault secrets: `az keyvault secret list --vault-name ccabana-kv-f5scjagc --query "[].name" -o tsv`
   — found `key-shard-1`, `key-shard-2`, `key-shard-3`, `master-key`
10. Retrieved `key-shard-1` and `key-shard-3` — contained flag fragments
11. Retrieved `key-shard-2` — current value was a rotation notice, not a flag fragment
12. `master-key` returned 403 — service principal lacked permission
13. Listed versions of `key-shard-2` — found two versions, older one pre-dating the rotation:
    `az keyvault secret list-versions --vault-name ccabana-kv-f5scjagc --name key-shard-2 -o table`
14. Retrieved old version of `key-shard-2` by ID — contained the missing flag fragment
15. Assembled flag from all three shards

## What I Learned
- Client-side JavaScript should never contain SAS tokens or cloud credentials — anything in the browser is public
- SAS tokens scoped to `srt=sco` allow listing all containers, not just the intended one — always restrict scope to the minimum required
- Storing service principal credentials in a blob readable by the SAS token creates a direct privilege escalation chain: public token → private credentials → Key Vault access
- Azure Key Vault retains all versions of a secret after rotation — rotating a secret does not delete the old value, it just adds a new one on top
- Always explicitly purge old secret versions after rotation, otherwise they remain recoverable by anyone with list-versions and get permissions
- Cloud misconfigurations rarely exist in isolation — this room chained four separate issues into a full compromise
