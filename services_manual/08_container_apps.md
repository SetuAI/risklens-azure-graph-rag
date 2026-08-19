# 08 — Azure Container Apps

The shop window.

> The last service, and the most satisfying. At the end of this page your project has a web address you can send to anyone.

---

## 1. What it does for RiskLens

Everything so far runs on your laptop. `streamlit run app/main.py` works beautifully — for you, while your laptop is open, on your network.

To share it, the app has to run somewhere that is always on and reachable from anywhere.

**Container Apps runs your app for you.** You hand Azure a packaged version of your application; Azure runs it, gives it a public web address, handles the certificate, and restarts it if it crashes.

### Why containers

A **container** is your app plus everything it needs to run — Python, the libraries, your code — packaged into one thing.

The problem it solves is the oldest one in software: *"it works on my machine."* Your laptop has Python 3.11 and a particular set of installed libraries. Azure's machine has something else. A container removes the question entirely, because the environment travels with the app.

### Why this one rather than the alternatives

Azure has several ways to host things. Container Apps suits us because:

- It **scales to zero** — no traffic, no compute charge
- No servers to manage
- A public address and HTTPS certificate come free
- It can hold a **managed identity**, which is what makes Key Vault work properly

That scale-to-zero behaviour matters for a student project. An app nobody is using costs almost nothing.

---

## 2. Prepare the app

### The Dockerfile

This is the recipe for building your container. Put it in `app/Dockerfile`:

```dockerfile
FROM python:3.11-slim

WORKDIR /app

# install dependencies first, so this layer is cached
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# then the code
COPY src/ ./src/
COPY app/ ./app/
COPY data/reference/ ./data/reference/

EXPOSE 8501

CMD ["streamlit", "run", "app/main.py", \
     "--server.port=8501", \
     "--server.address=0.0.0.0"]
```

Two lines deserve explanation.

**Requirements are copied before the code, on purpose.** Docker caches each step. Your code changes constantly; your dependencies rarely do. This order means a code change rebuilds in seconds instead of reinstalling every library.

**`--server.address=0.0.0.0`** tells Streamlit to accept connections from outside the container. Without it the app runs, appears healthy, and is unreachable — a genuinely baffling failure the first time you meet it.

### What must not go in

Check before building:

- **No `.env`** — a `.dockerignore` file should exclude it
- **No PDFs or extracted JSON** — those live in Blob Storage
- **No `.venv`**

Create `.dockerignore` next to the Dockerfile:

```
.env
.venv/
__pycache__/
data/raw_pdfs/
data/extracted/
data/review/
notebooks/
tests/
.git/
```

### Test locally first

Never deploy something you have not run locally. It turns a two-minute fix into a twenty-minute one.

```bash
docker build -t risklens -f app/Dockerfile .
docker run -p 8501:8501 --env-file .env risklens
```

Open `http://localhost:8501`. If it works here, it will very likely work in Azure.

---

## 3. Deploy

The Azure CLI does this in one command. Install it, then:

```bash
az login
```

### One command

```bash
az containerapp up \
  --name risklens-app \
  --resource-group rg-risklens \
  --location eastus \
  --source . \
  --target-port 8501 \
  --ingress external
```

This builds the container in the cloud, creates the environment, deploys, and prints your public URL.

First run takes several minutes. Later runs are much faster.

| Flag                   | What it means                                               |
| ---------------------- | ----------------------------------------------------------- |
| `--source .`         | Build from this folder, using the Dockerfile                |
| `--target-port 8501` | The port Streamlit listens on inside the container          |
| `--ingress external` | Reachable from the internet.`internal` would restrict it. |

---

## 4. Give the app its keys — properly

The app is running but has no secrets. Two ways to fix that.

### The quick way — secrets on the app

```bash
az containerapp secret set \
  --name risklens-app \
  --resource-group rg-risklens \
  --secrets \
    openai-key=<value> \
    search-key=<value>
```

Then bind them to environment variables:

```bash
az containerapp update \
  --name risklens-app \
  --resource-group rg-risklens \
  --set-env-vars \
    AZURE_OPENAI_KEY=secretref:openai-key \
    AZURE_SEARCH_KEY=secretref:search-key
```

Works fine. But the keys now exist in two places.

### The proper way — managed identity

This is where `07_key_vault.md` pays off.

**Step 1 — give the app an identity:**

```bash
az containerapp identity assign \
  --name risklens-app \
  --resource-group rg-risklens \
  --system-assigned
```

This prints a `principalId`. Copy it.

**Step 2 — let that identity read from Key Vault:**

Portal → your Key Vault → **Access control (IAM)** → **+ Add role assignment**
→ Role: **Key Vault Secrets User**
→ Assign access to: **Managed identity** → select `risklens-app`

**Step 3 — tell the app which vault:**

```bash
az containerapp update \
  --name risklens-app \
  --resource-group rg-risklens \
  --set-env-vars KEY_VAULT_NAME=kv-risklens-ab
```

Now the app reads its own secrets from Key Vault, and **there is not a single key anywhere in your deployment configuration**.

> This is the moment the last two pages come together. The app has an identity. The vault trusts that identity. No password was involved at any point. That is how production systems should work, and you have just built one.

---

## 5. Check it is working

**Get the URL:**

```bash
az containerapp show \
  --name risklens-app \
  --resource-group rg-risklens \
  --query properties.configuration.ingress.fqdn \
  --output tsv
```

**Watch the logs live:**

```bash
az containerapp logs show \
  --name risklens-app \
  --resource-group rg-risklens \
  --follow
```

This is your main debugging tool. When the app fails to start, the reason is here.

**Redeploy after a change:**

```bash
az containerapp up \
  --name risklens-app \
  --resource-group rg-risklens \
  --source .
```

---

## 6. Scaling

By default the app scales to zero when idle — no traffic, no compute cost. The trade-off is a **cold start**: the first visitor after a quiet period waits a few seconds while it wakes up.

For a demo you can leave one instance always running:

```bash
az containerapp update \
  --name risklens-app \
  --resource-group rg-risklens \
  --min-replicas 1 \
  --max-replicas 3
```

**Set it back to zero afterwards.** A minimum of one means paying continuously.

> Rule of thumb: **zero while learning, one during a live demo, zero again afterwards.**

---

## If something goes wrong

| Symptom                                | Almost always the cause                                                          |
| -------------------------------------- | -------------------------------------------------------------------------------- |
| App starts, URL times out              | Missing`--server.address=0.0.0.0` in the Dockerfile                            |
| `--target-port` mismatch             | The port in the Dockerfile and the flag must be the same                         |
| Container fails to start               | Read the logs. It is usually a missing library or an unset environment variable. |
| `KeyError` on an env var in logs     | A secret was not bound. Check the`secretref:` names match.                     |
| Key Vault forbidden, but fine locally  | The managed identity has no role on the vault. Redo step 2.                      |
| Build succeeds locally, fails in Azure | Something excluded by`.dockerignore` that the app actually needs               |
| Very slow first load                   | Cold start. Expected at zero replicas.                                           |

---

## Cost

The consumption plan includes a free monthly allowance that comfortably covers a teaching demo, provided the app scales to zero.

The way to make this expensive is to set `--min-replicas 1` and forget about it.

---

## Cleaning up

When the project is finished:

```bash
az group delete --name rg-risklens --yes
```

Everything goes — app, search index, graph, storage, all eight services.

**Export anything you want to keep first.** The graph, the extracted JSON, your evaluation results. Deletion cannot be undone.

---

## What you learned here

- A container packages the app *and* its environment, which ends "works on my machine"
- Copy dependencies before code in a Dockerfile, so rebuilds stay fast
- `0.0.0.0` is what makes a containerised web app reachable
- Managed identity means the app proves who it is instead of holding a password
- Scale to zero costs almost nothing; a minimum of one costs continuously
- Always run the container locally before deploying

---

## You have finished the service manual

Eight services, eight passing tests, and a deployed application.

The next time you build something on Azure, this whole sequence will take you an afternoon — and in a later project you will learn to describe all eight in a single file and create them automatically. That will make sense to you now, because you know what each one actually is.

Which was the point of doing it by hand.
