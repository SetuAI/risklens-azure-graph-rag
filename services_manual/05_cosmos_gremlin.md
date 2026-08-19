
# 05 — Azure Cosmos DB for Apache Gremlin

The graph database. This is the wall chart the whole project is built around.

> Do this one **after** Blob Storage, Document Intelligence, Azure OpenAI and AI Search. It is the fiddliest of the eight, and it is easier once you are used to the rhythm of create → copy keys → test.

---

## 1. What it does for RiskLens

Everything else in this project stores **text**. This stores **connections**.

When we say "Infosys discloses cybersecurity risk", we are storing three things: a box for Infosys, a box for cybersecurity risk, and a line joining them. Do that for five companies across two years and you get a chart you can walk around.

That walking around is the point. A normal database can answer "what risks does Infosys have?" A graph database can answer "which risks do three or more of these companies share?" — because it can follow lines from box to box without you telling it the route in advance.

In graph vocabulary: boxes are **vertices** (also called nodes), lines are **edges**. You will see both words used.

---

## 2. Create it in the portal

### Create the account

1. Portal → search **Azure Cosmos DB** → **+ Create**
2. You are shown a list of APIs. Choose **Azure Cosmos DB for Apache Gremlin**.
   **This choice is permanent.** You cannot switch an account to a different API later — you would have to delete it and start again. Read the tile twice before clicking.
3. Fill in:
   - **Resource group** — `rg-risklens`
   - **Account name** — `cosmos-risklens-<yourinitials>`
   - **Location** — the same region you have used for everything else
   - **Capacity mode** — **Serverless**
4. **Review + create** → **Create**

Account creation takes roughly 5–10 minutes. This is normal. Let it run.

> **Why serverless:** the other option, *Provisioned throughput*, reserves capacity and charges you by the hour whether or not you use it. Serverless charges per request. For a teaching project that sits idle most of the day, serverless is dramatically cheaper.

### Create the database and graph

Once the account is ready:

1. Open the account → **Data Explorer** in the left menu
2. **New Graph**
3. **Database id** — create new, called `risklens`
4. **Graph id** — `riskgraph`
5. **Partition key** — type `/pk`
6. **OK**

### About that partition key

Azure asks for a partition key because Cosmos is built to spread very large graphs across many machines. The partition key is the property it uses to decide what goes where.

Our graph is small — a few hundred vertices. So this is a formality, but **a required one, and it cannot be changed later.**

We use `/pk`, and every vertex we create will carry a `pk` property set to its type: `company`, `risk`, `segment`, and so on. That gives a reasonable spread and keeps the code simple.

**Common beginner error:** creating a vertex without a `pk` property. It fails with an unhelpful message. Every vertex you write must have one.

---

## 3. Connect it

### Copy the keys

1. In your Cosmos account, open **Keys** in the left menu
2. Look for the **Gremlin Endpoint**. It looks like:
   `wss://cosmos-risklens-xyz.gremlin.cosmos.azure.com:443/`
3. Copy the **Primary Key**

> **Watch out:** the Keys page also shows a `https://...documents.azure.com` URI. That is the document endpoint, not the graph one. Using it is a very common mistake and produces a confusing connection error. You want the one containing **gremlin** and starting with **wss**.

### Add to `.env`

```
COSMOS_GREMLIN_ENDPOINT=wss://cosmos-risklens-xyz.gremlin.cosmos.azure.com:443/
COSMOS_GREMLIN_KEY=your-primary-key-here
COSMOS_DATABASE=risklens
COSMOS_GRAPH=riskgraph
```

### Install the library

```bash
pip install gremlinpython==3.4.13
```

**Pin that version.** Cosmos DB implements an older version of the Gremlin standard, and the newest `gremlinpython` releases do not connect to it cleanly. This is the single most common setup failure on this service, and the error message does not tell you the version is the problem. If your `requirements.txt` pins it, you will never meet this issue.

### The username format

Connecting needs a "username", but it is not a name — it is the path to your graph:

```
/dbs/risklens/colls/riskgraph
```

Get this wrong and you get an authentication error that looks like a wrong key. Check the path before you check the key.

---

## 4. Prove it works

Create `tests/test_cosmos.py` and run it.

```python
import os
from dotenv import load_dotenv
from gremlin_python.driver import client, serializer

load_dotenv()

g = client.Client(
    os.environ["COSMOS_GREMLIN_ENDPOINT"],
    "g",
    username=f"/dbs/{os.environ['COSMOS_DATABASE']}/colls/{os.environ['COSMOS_GRAPH']}",
    password=os.environ["COSMOS_GREMLIN_KEY"],
    message_serializer=serializer.GraphSONSerializersV2d0(),
)

# add two boxes and a line between them
g.submit("g.addV('company').property('id','test_infosys').property('pk','company').property('name','Infosys Limited')").all().result()
g.submit("g.addV('risk').property('id','test_cyber').property('pk','risk').property('name','Cybersecurity & Data Breach')").all().result()
g.submit("g.V('test_infosys').addE('DISCLOSES_RISK').to(g.V('test_cyber'))").all().result()

# walk from the company along the line, and see where you land
result = g.submit("g.V('test_infosys').out('DISCLOSES_RISK').values('name')").all().result()
print("Infosys is connected to:", result)

# clean up
g.submit("g.V('test_infosys').drop()").all().result()
g.submit("g.V('test_cyber').drop()").all().result()

g.close()
print("Cosmos DB Gremlin: connected")
```

**A good result looks like:**

```
Infosys is connected to: ['Cybersecurity & Data Breach']
Cosmos DB Gremlin: connected
```

That one line is the whole idea of a graph database, in miniature. You did not tell it *where* cybersecurity was. You said "start at Infosys and follow the DISCLOSES_RISK line", and it found the answer by walking. Multiply that by five companies, ten reports and twenty-nine risks and you have RiskLens.

---

## Seeing the graph

### In the portal

Data Explorer → your graph → run a query → **Graph** view. Vertices appear as coloured circles joined by lines. Click one and its properties show in a side panel.

Useful for a quick check. Be aware it is fairly basic — the layout is not adjustable and it becomes cluttered past a few hundred vertices.

### In your own code — the better option

Later in the project we build `src/utils/visualize_graph.py`, which pulls a portion of the graph and draws it with **pyvis** into an HTML file you open in a browser.

That version is worth the small effort: you choose the colours (companies one colour, risks another), it looks the same every time you run it, and you can share the HTML with anyone without them needing an Azure login.

---

## If something goes wrong

| Symptom                                          | Almost always the cause                                                                                 |
| ------------------------------------------------ | ------------------------------------------------------------------------------------------------------- |
| Connection hangs, then times out                 | Wrong endpoint — using the`documents.azure.com` one instead of the `gremlin...` one                |
| Authentication failure                           | Username path is wrong. Must be`/dbs/<database>/colls/<graph>` exactly                                |
| Odd errors on import, or a handshake failure     | `gremlinpython` version. Pin it to `3.4.13`                                                         |
| "Partition key not found" when adding a vertex   | You forgot`.property('pk', ...)` on that vertex                                                       |
| Query returns nothing but you know data is there | Vertex ids are case sensitive.`Infosys` and `infosys` are different vertices                        |
| Everything is very slow                          | Serverless has a cold start after idle time. The first query of a session is slower — this is expected |

---

## Cost note

Serverless bills per request, and this graph is small, so cost here is genuinely minor.

But the account still exists whether or not you use it, and it is easy to forget. It goes away with the resource group at the end of the project — which is exactly why everything lives in one resource group.

---

## Next

Move on to `06_content_safety.md`.

Do not start writing the real graph-loading code until this test file prints its success line. Every later problem in this project is ten times harder to diagnose if you are unsure whether the connection itself is sound.
