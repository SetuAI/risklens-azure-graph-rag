# 04 — Azure AI Search

The index at the back of the book.

> This is the service that bills **by the hour**, whether you use it or not. It is the main reason we delete the resource group when the project is finished.

---

## 1. What it does for RiskLens

The graph stores connections. It does **not** store the paragraphs.

So when someone asks *"what did L&T say about supply chain risk?"*, the graph can tell you that L&T disclosed it — but the actual sentences, and the page number to cite, come from here.

Azure AI Search holds every chunk of text from all ten reports, and finds the relevant ones when a question arrives.

### Two ways of searching, and we use both

**Keyword search** looks for the words you typed. Ask for "attrition" and it finds paragraphs containing "attrition". Precise, but blind to synonyms.

**Vector search** uses the embeddings from Azure OpenAI. Ask about "employees leaving" and it finds a paragraph about "attrition rates" even though no word matches — because their numbers sit close together in meaning.

Each fails where the other succeeds. Keyword search misses paraphrasing. Vector search sometimes drifts to text that is *thematically* close but not what you asked for — and it is poor with exact names and numbers.

**Hybrid search runs both and combines the results.** That is what we use. It is not a compromise; it is genuinely better than either alone.

> **Plain version:** keyword search is looking someone up by exact name. Vector search is describing them and hoping to be understood. Hybrid does both and merges the answers.

---

## 2. Create it in the portal

1. Portal → search **AI Search** → **+ Create**
2. Fill in:

| Field          | Value                                                         |
| -------------- | ------------------------------------------------------------- |
| Resource group | `rg-risklens`                                               |
| Service name   | `srch-risklens-<yourinitials>`                              |
| Location       | Same region as everything else                                |
| Pricing tier   | **Basic** — click *Change Pricing Tier* to select it |

3. **Review + create** → **Create**

Takes a few minutes.

### On the tier

**Free** exists and sounds appealing, but it has tight limits on storage and index count, and one free service per subscription. Ten annual reports will not fit comfortably.

**Basic** is the smallest tier that works properly for this project.

> **This is the service to watch on cost.** Basic bills per hour from the moment it is created until it is deleted — using it or not, weekend or not. It is not expensive per hour, but a service left running for two months adds up. Delete the resource group when you are done.

---

## 3. Connect it

### Copy the keys

1. Open your search service
2. The **Url** is shown on the Overview page: `https://srch-risklens-ab.search.windows.net`
3. Left menu → **Keys** → copy the **Primary admin key**

> There are two kinds of key here. **Admin keys** can create and change indexes. **Query keys** can only search. We need admin for building, so use the admin key — but note the distinction, because in a real deployment your app should hold only a query key.

### Add to `.env`

```
AZURE_SEARCH_ENDPOINT=https://srch-risklens-ab.search.windows.net
AZURE_SEARCH_KEY=your-admin-key-here
AZURE_SEARCH_INDEX=risklens-chunks
```

The index does not exist yet. Our code creates it.

### Library

```
azure-search-documents
```

---

## 4. What an index is

Before the test, one idea worth having straight.

An **index** is a table with a fixed shape that you define in advance. You tell Azure what fields each record has, and which of them are searchable.

Ours looks like this:

| Field              | Holds                                      | Why                             |
| ------------------ | ------------------------------------------ | ------------------------------- |
| `id`             | A unique id for the chunk                  | Required                        |
| `content`        | The paragraph text itself                  | What we search and quote        |
| `content_vector` | The 1,536 numbers from the embedding model | Makes vector search possible    |
| `company_id`     | `C001`, `C002`, …                     | So we can filter to one company |
| `financial_year` | `FY2024-25`                              | So we can filter to one year    |
| `page_number`    | Where it came from                         | So we can cite it               |

Those last three exist so we can **filter**. Asking "what did Infosys say in FY2026 about attrition" should not search Reliance's reports at all. Filtering first makes answers both faster and more accurate.

> The company id and year come straight from the filename. This is why the naming convention mattered back in `01_blob_storage.md`.

---

## 5. Prove it works

Put this in `tests/test_search.py`.

```python
import os
from dotenv import load_dotenv
from azure.core.credentials import AzureKeyCredential
from azure.search.documents.indexes import SearchIndexClient
from azure.search.documents import SearchClient
from azure.search.documents.indexes.models import (
    SearchIndex, SimpleField, SearchableField, SearchFieldDataType,
)

load_dotenv()

endpoint = os.environ["AZURE_SEARCH_ENDPOINT"]
credential = AzureKeyCredential(os.environ["AZURE_SEARCH_KEY"])
test_index = "risklens-connection-test"

index_client = SearchIndexClient(endpoint=endpoint, credential=credential)

# 1. create a small test index
index = SearchIndex(
    name=test_index,
    fields=[
        SimpleField(name="id", type=SearchFieldDataType.String, key=True),
        SearchableField(name="content", type=SearchFieldDataType.String),
        SimpleField(name="company_id", type=SearchFieldDataType.String, filterable=True),
    ],
)
index_client.create_or_update_index(index)
print("created test index")

# 2. add two documents
search_client = SearchClient(endpoint=endpoint, index_name=test_index, credential=credential)
search_client.upload_documents([
    {"id": "1", "content": "We face risks from cyber attacks on our systems.", "company_id": "C001"},
    {"id": "2", "content": "Attrition among senior engineers remains elevated.", "company_id": "C001"},
])
print("uploaded 2 documents")

import time
time.sleep(2)   # indexing is not instant

# 3. search
results = search_client.search(search_text="cyber")
for r in results:
    print("found:", r["content"])

# 4. clean up
index_client.delete_index(test_index)
print("deleted test index")

print("Azure AI Search: connected")
```

Run it:

```bash
python tests/test_search.py
```

**A good result looks like:**

```
created test index
uploaded 2 documents
found: We face risks from cyber attacks on our systems.
deleted test index
Azure AI Search: connected
```

> Note the `time.sleep(2)`. Uploading a document does not make it searchable instantly — the service needs a moment to index it. If your search returns nothing immediately after uploading, wait and try again before assuming something is broken. This trips up almost everyone once.

---

## 6. The real index

The test index above is keyword-only, deliberately — it keeps the first contact simple.

The real index adds the **vector field**, which is what makes meaning-based search work. That involves a little more configuration: the field holds 1,536 numbers, and you tell Azure how to compare them.

We build that properly in `src/step4_build/load_search_index.py` during the lesson. The shape:

```python
# define fields, including the vector field
# create the index
# for each chunk of extracted text:
#     get its embedding from Azure OpenAI
#     upload chunk + embedding + company_id + year + page
```

**Upload in batches, not one at a time.** A few hundred documents at once is far faster than a few hundred separate calls.

---

## 7. Why this stays, alongside the graph

Students reasonably ask why we need both.

Because a good answer needs two different things:

**Which companies, and how they connect** → the graph
**What was actually written, and on which page** → this index

Take away the graph and the system cannot join facts across reports. Take away the index and it can tell you Infosys discloses cybersecurity risk, but cannot show you the sentence or the page — and an answer you cannot verify is not much of an answer.

There is also a teaching reason. In `notebooks/03_vector_vs_graph.ipynb` we run the Group B questions against this service **alone**, with the graph switched off, and record the wrong answers. That comparison is the proof that the graph was worth building.

---

## If something goes wrong

| Symptom                                  | Almost always the cause                                                                                 |
| ---------------------------------------- | ------------------------------------------------------------------------------------------------------- |
| `403 Forbidden`                        | You used a query key where an admin key is needed                                                       |
| Search returns nothing just after upload | Indexing lag. Wait a couple of seconds and search again.                                                |
| `Index not found`                      | Index name in`.env` does not match what was created                                                   |
| Vector field rejected                    | The number of dimensions must match your embedding model exactly — 1,536 for`text-embedding-3-small` |
| Free tier storage errors                 | Free tier is too small. Move to Basic.                                                                  |
| Upload rejected                          | Every document needs a unique`id`, and the field marked `key=True` must be a string                 |

---

## Cost

Basic tier bills **per hour**, continuously, from creation to deletion.

Nothing else in this project works this way — Blob, Cosmos serverless, Document Intelligence and OpenAI all bill by what you actually use.

Two habits that follow:

- Do not create this service until you are ready to use it
- Delete the resource group when the project is finished

---

## What you learned here

- Keyword search and vector search fail in different ways; hybrid uses both
- An index has a shape you define in advance, and that shape decides what you can filter on
- Extra fields like company and year are what make filtering possible — and they came from your filenames
- Indexing is not instant
- Some cloud services bill for existing, not for being used

---

## Next

`05_cosmos_gremlin.md` — the graph itself, and the most interesting service in the project.
