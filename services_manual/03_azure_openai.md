
# 03 — Azure OpenAI Service

The brain that understands meaning.

> This is the service most likely to delay you — not because it is hard, but because **access sometimes needs approval on your subscription**. Check that today, even if you are not ready to build yet.

---

## 1. What it does for RiskLens

Two completely different jobs. Same service, two models.

### Job 1 — Understanding text (the chat model)

We use **GPT-4o**. Give it a page from an annual report and ask it to find every risk mentioned, and it comes back with a structured list. It also writes the final answers a user reads.

Where it earns its keep is **understanding**. When L&T writes *"failure of information technology systems could disrupt project delivery"*, no keyword search would match that to a category called "IT System Failure". The model understands it means the same thing.

### Job 2 — Turning text into numbers (the embedding model)

We use **text-embedding-3-small**.

An embedding turns a piece of text into a long list of numbers that captures its meaning. Two passages about hacking end up with similar numbers, even if they share no words at all.

That is what makes search work by meaning rather than by exact wording. Ask about "cyber attacks" and it finds a paragraph about "unauthorised system access", because their numbers sit close together.

> **Plain version:** the chat model reads and writes. The embedding model measures how similar two pieces of text are in meaning. You need both.

---

## 2. Check access first

On some subscriptions, Azure OpenAI is available immediately. On others, it needs to be requested and approved.

**Check now**, before you plan a recording day around it:

1. Portal → search **Azure OpenAI**
2. Try **+ Create**

If the form opens normally, you have access. If it tells you to apply, submit the form and expect to wait — it can take a day or two.

---

## 3. Create it in the portal

1. Portal → **Azure OpenAI** → **+ Create**
2. Fill in:

| Field          | Value                            |
| -------------- | -------------------------------- |
| Resource group | `rg-risklens`                  |
| Region         | See the warning below            |
| Name           | `aoai-risklens-<yourinitials>` |
| Pricing tier   | Standard S0                      |

3. Leave the network and tags tabs at defaults
4. **Review + create** → **Create**

### Region matters more here than anywhere else

**Not every model is available in every region.** A region may have GPT-4o but not the embedding model, or have both but no spare capacity.

Before creating the resource, check Microsoft's model availability page and pick a region that has **both** GPT-4o and a text embedding model. `East US` and `Sweden Central` are usually safe choices.

If you pick badly, you cannot move the resource. You delete it and create a new one in a different region.

---

## 4. Deploy the models

Creating the resource does not give you a model. It gives you an empty place to put models. This surprises everyone the first time.

1. Open your Azure OpenAI resource
2. Click **Go to Azure AI Foundry portal** (this may be named Azure OpenAI Studio depending on when you read this)
3. Left menu → **Deployments** → **Deploy model**

Deploy **two**:

| Model                      | Deployment name to use     | What it is for                   |
| -------------------------- | -------------------------- | -------------------------------- |
| `gpt-4o`                 | `gpt-4o`                 | Reading text and writing answers |
| `text-embedding-3-small` | `text-embedding-3-small` | Turning text into numbers        |

### The deployment name is what your code uses

This is the single most common confusion with this service.

The **model** is `gpt-4o`. The **deployment** is a name *you* choose. You could call your deployment `banana` and your code would then have to say `banana`.

**Give the deployment the same name as the model.** It removes an entire class of confusion, and there is no benefit to doing otherwise.

### About the rate limit

When deploying, you set a tokens-per-minute limit. The default is usually fine to start.

If you later see errors mentioning `429` or "rate limit", it means you are sending requests faster than your allowance. Either raise the limit here, or add a short wait between requests in your code.

---

## 5. Connect it

### Copy the keys

Back in the Azure portal, on your Azure OpenAI resource → **Keys and Endpoint**.

Copy **KEY 1** and the **Endpoint**, which looks like:

```
https://aoai-risklens-ab.openai.azure.com/
```

### Add to `.env`

```
AZURE_OPENAI_ENDPOINT=https://aoai-risklens-ab.openai.azure.com/
AZURE_OPENAI_KEY=your-key-here
AZURE_OPENAI_API_VERSION=2024-10-21
AZURE_OPENAI_CHAT_DEPLOYMENT=gpt-4o
AZURE_OPENAI_EMBEDDING_DEPLOYMENT=text-embedding-3-small
```

> **On the API version:** this tells Azure which version of the interface you expect. The value above is an example — check Microsoft's documentation for a current supported version. If you get a strange error mentioning an unsupported parameter, an outdated API version is a likely cause.

### Library

```
openai
```

The same `openai` package talks to Azure — you use the `AzureOpenAI` client instead of the standard one.

---

## 6. Prove it works

Put this in `tests/test_openai.py`.

```python
import os
from dotenv import load_dotenv
from openai import AzureOpenAI

load_dotenv()

client = AzureOpenAI(
    azure_endpoint=os.environ["AZURE_OPENAI_ENDPOINT"],
    api_key=os.environ["AZURE_OPENAI_KEY"],
    api_version=os.environ["AZURE_OPENAI_API_VERSION"],
)

# --- Job 1: the chat model ---
response = client.chat.completions.create(
    model=os.environ["AZURE_OPENAI_CHAT_DEPLOYMENT"],
    messages=[
        {"role": "system", "content": "You map risk descriptions to standard categories. Reply with only the category name."},
        {"role": "user", "content": "Map this to a standard risk category: 'failure of information technology systems could disrupt project delivery'"},
    ],
    max_tokens=20,
)
print("chat model says:", response.choices[0].message.content.strip())

# --- Job 2: the embedding model ---
embedding = client.embeddings.create(
    model=os.environ["AZURE_OPENAI_EMBEDDING_DEPLOYMENT"],
    input="cyber attacks on our systems",
)
vector = embedding.data[0].embedding
print(f"embedding length: {len(vector)} numbers")
print(f"first three: {[round(v, 4) for v in vector[:3]]}")

print("Azure OpenAI: connected")
```

Run it:

```bash
python tests/test_openai.py
```

**A good result looks like:**

```
chat model says: IT System Failure
embedding length: 1536 numbers
first three: [-0.0142, 0.0231, -0.0087]
Azure OpenAI: connected
```

Two things just happened that are worth pausing on.

The chat model read a sentence that never contained the words "IT System Failure" and produced exactly that category. **That is Step 2 of our pipeline, working, in one call.**

The embedding model turned a short sentence into 1,536 numbers. Those numbers are how the search index will later find this text when someone asks about hacking in completely different words.

---

## 7. Cost

You pay per **token** — roughly, per chunk of a word. Both what you send and what comes back.

The bulk of the cost in this project is Step 2: sending thousands of pages of report text to GPT-4o for entity extraction.

**Ways to keep it sensible:**

- Send only the sections you need — risk, MD&A, segments, board — not all 300 pages of every report
- Do not re-run extraction on pages you have already processed
- Test prompts on **one page** until the output looks right, then run the batch
- The embedding model is far cheaper than the chat model, so embedding everything is fine; sending everything to GPT-4o is not

> The pattern is the same as Document Intelligence: **get it right on a small sample, then run once at scale, then save the output.**

---

## If something goes wrong

| Symptom                                                      | Almost always the cause                                                                                                               |
| ------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------- |
| `DeploymentNotFound`                                       | The name in`.env` does not match the deployment name in the Foundry portal. Check for typos and capitals.                           |
| `401 Unauthorized`                                         | Wrong key, or you used the endpoint from a different resource                                                                         |
| `429` errors                                               | Rate limit. Slow down, or raise the tokens-per-minute on the deployment.                                                              |
| Error about an unsupported parameter                         | Your`AZURE_OPENAI_API_VERSION` is old. Update it.                                                                                   |
| Cannot find the model when deploying                         | It is not offered in your region. Delete and recreate in a supported region.                                                          |
| The model returns something that is not on our category list | Your prompt is not constraining it enough. This is a prompt problem, not a model problem — we fix it in the taxonomy mapping lesson. |

---

## What you learned here

- One service, two models, doing two unrelated jobs
- Creating the resource is not the same as deploying a model
- **Your code uses the deployment name, not the model name**
- Embeddings turn meaning into numbers, which is what makes search work by meaning
- Region availability is a real constraint, and it cannot be changed after creation
- Test prompts small, run large once, save the output

---

## Next

`04_ai_search.md` — where those embeddings get stored so they can actually be searched.
