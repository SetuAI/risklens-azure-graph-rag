
# 01 — Azure Blob Storage

The cupboard the books sit in.

> This is the first service you create, and the easiest of the eight. Its real job here is to teach you the rhythm: **create → copy keys → connect → test**. Every service after this follows the same four steps.

---

## 1. What it does for RiskLens

Our ten annual reports have to live somewhere the rest of the project can reach.

Right now they are on your laptop. That works while you are the only person running the code. It stops working the moment your app is deployed to the cloud — the app runs on Azure's machines, and Azure's machines cannot see your Downloads folder.

Blob Storage is a place in the cloud to keep files. "Blob" just means *a file, of any type* — Azure does not care whether it is a PDF, an image or a spreadsheet.

So the flow becomes:

**Your laptop → Blob Storage → Document Intelligence reads from there**

We also use it for a second thing later: storing the **extracted JSON** that Document Intelligence produces. That output costs real money to generate, so once we have it, we keep it safe rather than re-creating it.

### One thing to be clear about

Blob Storage is **not** a database. You cannot ask it questions. You cannot search inside the files. It stores a file and gives it back when you ask by name. Nothing more.

That simplicity is the point. Files go in a cupboard; questions get answered elsewhere.

---

## 2. Create it in the portal

### Create the storage account

1. Go to **portal.azure.com**
2. Search for **Storage accounts** in the top bar → **+ Create**
3. Fill in the Basics tab:

| Field                | What to put                                            | Why                                                                                                  |
| -------------------- | ------------------------------------------------------ | ---------------------------------------------------------------------------------------------------- |
| Subscription         | Yours                                                  | —                                                                                                   |
| Resource group       | `rg-risklens`                                        | Same group as everything else, so one delete removes it all                                          |
| Storage account name | `strisklens<yourinitials>01`                         | See naming rules below                                                                               |
| Region               | The same region you chose in`00_before_you_start.md` | Keep every service together                                                                          |
| Primary service      | Azure Blob Storage                                     | What we are using it for                                                                             |
| Performance          | **Standard**                                     | Premium is for high-speed workloads. We are storing PDFs.                                            |
| Redundancy           | **LRS (Locally-redundant storage)**              | The cheapest option. It keeps copies within one data centre, which is plenty for a teaching project. |

4. Leave every other tab at its defaults
5. **Review + create** → **Create**

Takes about a minute.

### The naming rules will catch you out

Storage account names are the fussiest in all of Azure:

- **Lowercase letters and numbers only** — no capitals, no dashes, no underscores
- **3 to 24 characters**
- **Globally unique** — unique across every Azure customer in the world, not just your account

That last one surprises people. `storage1` was taken years ago. If Azure says the name is unavailable, it is not a bug — someone else has it. Add your initials and a number.

### Create the container

A **container** is a folder inside the storage account. You need at least one.

1. Open your new storage account
2. Left menu → **Containers** (under *Data storage*)
3. **+ Container**
4. Name: `annual-reports`
5. Public access level: **Private (no anonymous access)**
6. **Create**

> **Leave it private.** The other options make your files readable by anyone on the internet who knows the address. Annual reports are public documents, so nothing terrible would happen here — but forming the habit matters. In a real project this setting is how company data ends up leaked, and it is one of the most common cloud mistakes in the world.

---

## 3. Connect it

### Copy the connection string

1. In your storage account, left menu → **Access keys** (under *Security + networking*)
2. You will see **key1** and **key2**. Either works — two exist so you can replace one without downtime
3. Under key1, find **Connection string** → click **Show** → copy it

It is a long line that looks roughly like:

```
DefaultEndpointsProtocol=https;AccountName=strisklensab01;AccountKey=xxxxx...;EndpointSuffix=core.windows.net
```

> **This one string contains full access to your storage account.** Treat it exactly like a password. It goes in `.env`, and `.env` is never committed to git.

### Add to `.env`

```
AZURE_STORAGE_CONNECTION_STRING=DefaultEndpointsProtocol=https;AccountName=...;EndpointSuffix=core.windows.net
AZURE_STORAGE_CONTAINER=annual-reports
```

Two things to watch:

- **No quotes** around the value, and **no spaces** around the `=`
- The whole connection string goes on **one line**, however long it looks

### Library

Already in `requirements.txt`, so nothing to install if you have run it:

```
azure-storage-blob
```

---

## 4. Prove it works

Open `tests/test_blob.py` and put this in it.

```python
import os
from dotenv import load_dotenv
from azure.storage.blob import BlobServiceClient

load_dotenv()

connection_string = os.environ["AZURE_STORAGE_CONNECTION_STRING"]
container_name = os.environ["AZURE_STORAGE_CONTAINER"]

service = BlobServiceClient.from_connection_string(connection_string)
container = service.get_container_client(container_name)

# 1. write a small test file
container.upload_blob(name="hello.txt", data=b"RiskLens is connected", overwrite=True)
print("uploaded hello.txt")

# 2. list what is in the container
print("files in container:", [b.name for b in container.list_blobs()])

# 3. read it back
downloaded = container.get_blob_client("hello.txt").download_blob().readall()
print("file says:", downloaded.decode())

# 4. clean up
container.delete_blob("hello.txt")
print("deleted hello.txt")

print("Blob Storage: connected")
```

Run it:

```bash
python tests/test_blob.py
```

**A good result looks like:**

```
uploaded hello.txt
files in container: ['hello.txt']
file says: RiskLens is connected
deleted hello.txt
Blob Storage: connected
```

Four operations — write, list, read, delete. That is everything Blob Storage does. If those four work, this service is fully set up and you never need to think about it again.

---

## Upload the annual reports

Now put the real files in. Two ways.

### Through the portal — fine for a one-off

1. Storage account → **Containers** → `annual-reports`
2. **Upload** → select your ten PDFs → **Upload**

Simple, but you would not want to do it repeatedly.

### Through code — the better habit

Create `src/utils/upload_reports.py`:

```python
import os
from pathlib import Path
from dotenv import load_dotenv
from azure.storage.blob import BlobServiceClient

load_dotenv()

service = BlobServiceClient.from_connection_string(
    os.environ["AZURE_STORAGE_CONNECTION_STRING"]
)
container = service.get_container_client(os.environ["AZURE_STORAGE_CONTAINER"])

pdf_folder = Path("data/raw_pdfs")

for pdf in sorted(pdf_folder.glob("*.pdf")):
    with open(pdf, "rb") as f:
        container.upload_blob(name=pdf.name, data=f, overwrite=True)
    print(f"uploaded {pdf.name}")

print("\nin container now:")
for blob in container.list_blobs():
    print(f"  {blob.name}  ({blob.size / 1_000_000:.1f} MB)")
```

```bash
python src/utils/upload_reports.py
```

You should see ten files listed. Annual reports are typically 5–30 MB each, so expect this to take a couple of minutes.

> `overwrite=True` means running it again replaces the files rather than failing. That makes the script safe to re-run, which matters more than you would think — you will re-run it.

### Check your filenames first

Before uploading, confirm the names follow the convention:

```bash
ls data/raw_pdfs/
```

Expected:

```
C001_infosys_FY2024-25.pdf
C001_infosys_FY2025-26.pdf
C002_eternal_FY2024-25.pdf
C002_eternal_FY2025-26.pdf
C003_lt_FY2024-25.pdf
...
```

**This matters more than it looks.** Later steps read the company id and the financial year straight out of the filename. Get the names right now and the pipeline knows which company and which year every page belongs to, without having to work it out from the document itself. Get them wrong and you will be fixing it in three different places.

---

## If something goes wrong

| Symptom                                         | Almost always the cause                                                                                                          |
| ----------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------- |
| `KeyError: 'AZURE_STORAGE_CONNECTION_STRING'` | `.env` is missing, or has quotes/spaces around the value, or you are running from the wrong folder. Run from the project root. |
| `The specified container does not exist`      | Container name in`.env` does not match the portal. Check spelling — it is case sensitive.                                     |
| `AuthenticationFailed`                        | Connection string copied incompletely. Copy the whole line again.                                                                |
| Name unavailable when creating the account      | Someone else in the world has it. Add initials and numbers.                                                                      |
| `The specified blob already exists`           | You forgot`overwrite=True`.                                                                                                    |
| Upload is very slow                             | Normal for 30 MB PDFs. Let it run.                                                                                               |

---

## Cost

Effectively nothing. Ten PDFs is a few hundred megabytes, and Blob Storage at Standard LRS is priced per gigabyte per month at a very low rate.

This is the cheapest service in the project. It still disappears with the resource group at the end.

---

## What you learned here

- Files that your cloud app needs must live in the cloud, not on your laptop
- A connection string is a password — it belongs in `.env`, never in code or git
- Private by default, always
- Consistent filenames are not tidiness; they carry information the pipeline depends on
- **Create → copy keys → connect → test.** Same four steps for the next seven services.

---

## Next

`02_document_intelligence.md` — where the PDFs stop being pictures of pages and start being text you can work with.

Do not move on until `test_blob.py` prints its success line.
