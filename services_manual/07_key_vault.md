# 07 — Azure Key Vault

The locked drawer for passwords.

> **You do not need this while building.** It belongs to the deployment stage. Everything up to now has used `.env` on your own machine, which is correct for local work. This page is about what changes when the app leaves your laptop.

---

## 1. What it does for RiskLens

Right now every key in this project sits in one file called `.env` on your computer.

That is fine, as long as two things are true: the file never leaves your machine, and only you can read it.

Deployment breaks both. The app now runs on Azure's machines. Those machines need the keys — so how do they get them?

**The wrong answers, in order of how bad they are:**

- Put the keys in the code. They end up in git, and public-repo scanners find exposed Azure keys within minutes. This genuinely happens, constantly.
- Bake `.env` into the container image. Anyone who can pull the image has your keys.
- Paste them into the deployment settings by hand. Works, but now they exist in several places and nobody knows how many.

**Key Vault is the right answer.** Secrets live in one place. The app asks for them at startup. Nobody ever sees the values — not in the code, not in git, not in the deployment configuration.

### The extra thing you get

Key Vault records **who read which secret and when**. So if a key ever needs to be replaced, you can see where it was used. `.env` gives you nothing like that.

---

## 2. Create it in the portal

1. Portal → search **Key Vault** → **+ Create**
2. Fill in:

| Field          | Value                                            |
| -------------- | ------------------------------------------------ |
| Resource group | `rg-risklens`                                  |
| Key vault name | `kv-risklens-<yourinitials>` (globally unique) |
| Region         | Same as everything else                          |
| Pricing tier   | Standard                                         |

3. On the **Access configuration** tab, choose **Azure role-based access control (RBAC)**
4. **Review + create** → **Create**

### Give yourself permission

This step surprises everyone: **creating a Key Vault does not let you read or write secrets in it.** Ownership and access are separate things.

1. Open your Key Vault → **Access control (IAM)**
2. **+ Add** → **Add role assignment**
3. Role: **Key Vault Secrets Officer**
4. Members: select your own account
5. **Review + assign**

Wait a minute or two for it to take effect. If you get a "forbidden" error immediately afterwards, this is usually why — give it a moment and try again.

> **Why separate at all?** Because in a real organisation, the person who creates infrastructure is often not the person allowed to read production passwords. Azure keeps the two apart by default. It is briefly annoying and fundamentally correct.

---

## 3. Add your secrets

### In the portal

1. Key Vault → **Secrets** → **+ Generate/Import**
2. Name: `azure-openai-key`
3. Value: paste the key
4. **Create**

Repeat for each one.

### Naming

Secret names allow **letters, numbers and dashes only** — no underscores. So your `.env` names need translating:

| `.env` name                       | Key Vault secret name               |
| ----------------------------------- | ----------------------------------- |
| `AZURE_OPENAI_KEY`                | `azure-openai-key`                |
| `AZURE_SEARCH_KEY`                | `azure-search-key`                |
| `COSMOS_GREMLIN_KEY`              | `cosmos-gremlin-key`              |
| `DOC_INTELLIGENCE_KEY`            | `doc-intelligence-key`            |
| `CONTENT_SAFETY_KEY`              | `content-safety-key`              |
| `AZURE_STORAGE_CONNECTION_STRING` | `azure-storage-connection-string` |

**Store only the secrets.** Endpoints and deployment names are not sensitive — they can stay as ordinary configuration.

---

## 4. Connect it

Key Vault needs two libraries that are not yet in `requirements.txt`. Add them now:

```
azure-keyvault-secrets
azure-identity
```

```bash
pip install azure-keyvault-secrets azure-identity
```

### Sign in locally

Reading from Key Vault does not use a key — that would defeat the point. It uses **your identity**.

Install the Azure CLI, then:

```bash
az login
```

Your code then signs in as you, automatically.

> **This is the idea worth taking away.** `DefaultAzureCredential` looks for a signed-in identity. On your laptop that is your `az login` session. On Azure it is the app's own managed identity. **Same code, both places, and no key anywhere.** Getting rid of keys entirely is better than protecting them.

---

## 5. Prove it works

Put this in `tests/test_keyvault.py`.

```python
import os
from dotenv import load_dotenv
from azure.identity import DefaultAzureCredential
from azure.keyvault.secrets import SecretClient

load_dotenv()

vault_name = os.environ["KEY_VAULT_NAME"]          # e.g. kv-risklens-ab
vault_url = f"https://{vault_name}.vault.azure.net"

client = SecretClient(vault_url=vault_url, credential=DefaultAzureCredential())

# 1. write a test secret
client.set_secret("risklens-test", "it works")
print("wrote test secret")

# 2. read it back
value = client.get_secret("risklens-test").value
print("read back:", value)

# 3. list what is stored (names only - values are never listed)
print("secrets in vault:")
for prop in client.list_properties_of_secrets():
    print("  ", prop.name)

# 4. clean up
client.begin_delete_secret("risklens-test").wait()
print("deleted test secret")

print("Key Vault: connected")
```

Add to `.env`:

```
KEY_VAULT_NAME=kv-risklens-ab
```

**A good result looks like:**

```
wrote test secret
read back: it works
secrets in vault:
   azure-openai-key
   azure-search-key
   ...
   risklens-test
deleted test secret
Key Vault: connected
```

> Notice that listing gives you **names, not values**. You cannot accidentally print every secret to a log. That is deliberate.

---

## 6. Using it in the app

The pattern: try Key Vault, fall back to `.env`. So the same code runs locally and deployed.

```python
import os
from azure.identity import DefaultAzureCredential
from azure.keyvault.secrets import SecretClient

def get_secret(env_name: str, vault_secret_name: str) -> str:
    """Read from Key Vault if configured, otherwise from .env."""
    vault_name = os.getenv("KEY_VAULT_NAME")
    if vault_name:
        client = SecretClient(
            vault_url=f"https://{vault_name}.vault.azure.net",
            credential=DefaultAzureCredential(),
        )
        return client.get_secret(vault_secret_name).value
    return os.environ[env_name]
```

**Read secrets once, at startup.** Do not call Key Vault on every request — it is a network call, it is slower, and there are rate limits.

---

## 7. Rotating a key

The point of all this becomes obvious the day a key is exposed.

**Without Key Vault:** find every place the key was pasted, update each one, redeploy, hope you found them all.

**With Key Vault:** update one secret, restart the app. Done.

Every Azure service gives you **two keys** for exactly this reason. Switch the app to key 2, regenerate key 1, switch back later. No downtime.

---

## If something goes wrong

| Symptom                                 | Almost always the cause                                                          |
| --------------------------------------- | -------------------------------------------------------------------------------- |
| `Forbidden` when reading a secret     | The RBAC role has not been assigned, or has not taken effect yet. Wait a minute. |
| `DefaultAzureCredential failed`       | You are not signed in. Run`az login`.                                          |
| Secret name rejected                    | Underscores are not allowed. Use dashes.                                         |
| Vault name unavailable                  | Globally unique. Add initials and numbers.                                       |
| Works locally, fails when deployed      | The deployed app has no identity yet. See`08_container_apps.md`.               |
| Deleted a secret and cannot recreate it | Soft-delete keeps it for a retention period. Purge it, or use a different name.  |

---

## Cost

Effectively free at this usage. Key Vault charges per operation, and reading a handful of secrets at startup is nothing.

---

## What you learned here

- `.env` is right for local work and wrong for deployment
- Creating a vault and being allowed to use it are separate permissions
- Secret names allow dashes, not underscores
- **The real prize is not protecting keys — it is not having keys at all**, via managed identity
- Read secrets once at startup, not per request
- Two keys per service exist so you can rotate without downtime

---

## Next

`08_container_apps.md` — putting the app on the internet, and giving it the identity that makes this page pay off.
