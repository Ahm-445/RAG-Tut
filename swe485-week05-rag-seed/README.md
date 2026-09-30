# swe485-week05-rag-seed

Seed repository for **SWE 485 Week 5 — RAG: Grounding AI in Your Codebase**.

A small, deliberately flawed corpus plus two Colab notebooks. In Track A you
build the retrieve → augment → generate loop by hand and print every similarity
score. In Track B you hand the same corpus to Gemini's managed File Search tool
and write no retrieval code at all. The contrast between the two is the lesson.

Lab brief: [`../SWE485_Week05_Lab.md`](../SWE485_Week05_Lab.md)

---

## Quick start

1. Fork this into your team repository.
2. Get a free Gemini API key at [aistudio.google.com/apikey](https://aistudio.google.com/apikey).
3. Open `01_rag_by_hand.ipynb` in Colab and add the key as the Colab Secret
   **`GEMINI_API_KEY`**.
4. Run the notebook top to bottom.
5. Then run `02_managed_rag.ipynb`, and **run its last cell** to delete the store.

**Never paste an API key into a cell, and never commit one.** The notebooks read
the key through `userdata.get()` for exactly this reason. A key in your git
history is a Week 3 data-handling failure and is marked as one.

---

## Layout

```
swe485-week05-rag-seed/
├── config.py                    every perishable setting, in one place
├── 01_rag_by_hand.ipynb         Track A — the graded core
├── 02_managed_rag.ipynb         Track B — managed RAG, for contrast
├── queries.md                   the six queries and what each exposes
├── embeddings.npz               offline fallback (generated; not committed)
├── minibank/                    the corpus
│   ├── .ragignore
│   ├── auth/       login, password, tokens, mfa
│   ├── billing/    charges, refunds, invoices
│   ├── config/     settings.py — every runtime tunable
│   └── docs/       README.md and ADR-002
└── tools/
    ├── build_embeddings.py      regenerate embeddings.npz
    └── verify_corpus.py         check the planted failures still fire
```

## The corpus is broken on purpose

`minibank/` is a ~500-line account service — authentication on one side,
subscription billing on the other. The auth/billing split is not decorative: it
reproduces the worked trace from §3 of the study material, where a login query
scored ≈0.99 against auth and ≈0.44 against billing on toy 2-D vectors. You will
watch 768-dimensional vectors reproduce that ordering.

Every planted defect maps to one of the five failure modes in §6:

| Mode | Where | What was planted |
|---|---|---|
| 1 bad chunking | `auth/tokens.py` | `validate_token()` spans L43–L81; at 500/50 the cut lands at **L68**, between the expiry check and the signature check |
| 2 missed context | `auth/mfa.py` | excluded by `.ragignore`, never indexed |
| 3 stale source | `docs/README.md` | claims 24-hour tokens; `settings.py` says 3600s; `ADR-002` records the change and notes the README was not updated |
| 4 wrong retrieval | `billing/refund.py` | dense in *failed / declined / disputed*, outranking the real retry logic in `charges.py` |
| 5 needle | three files | the lapsed-subscription answer needs `charges.py` + `login.py` + `settings.py` |

**Do not fix the README.** It is wrong in the way real READMEs are wrong: it was
true once.

---

## Offline mode

Set `config.OFFLINE = True` and the notebook loads `embeddings.npz` instead of
calling the embedding API. Chunking, scoring, ranking and every Part B failure
still run; only the generation cells need the network.

Use it if the room's connection is unreliable during an assessed demo. A 20-mark
live assessment should not depend on the building's Wi-Fi.

`embeddings.npz` is **generated, not committed** — it must be rebuilt whenever
the corpus, the chunk settings or `EMBED_MODEL` change, or the offline path
silently diverges from the online one. The notebook asserts that the cached ids
match the current chunking and fails loudly if they do not.

---

## Maintaining this repository

```bash
python3 tools/verify_corpus.py                 # after ANY corpus edit — no key needed
GEMINI_API_KEY=... python3 tools/build_embeddings.py   # after any corpus or model change
```

`verify_corpus.py` checks that the planted failures still fire. This matters
more than it sounds: if a defect stops firing, the lab quietly becomes a
demonstration that RAG works fine, which is the opposite of the lesson. Four of
the five modes are structural and are checked offline. **Mode 4 is a ranking
outcome and can only be confirmed with a live key** — see `queries.md`.

Neither tool reimplements the chunker. Both extract the cell tagged `chunker`
from `01_rag_by_hand.ipynb`, so there is one definition of chunking in this
repository and it is the one students read.

### Perishable

`config.py` holds every model id, dimension and task-type name, with a comment
explaining why `gemini-embedding-001` was chosen over `gemini-embedding-2`
(explicit `task_type` field, and un-normalized vectors below 3072 dimensions
that force the normalization step §3 asks about). If `-001` is retired, edit
`prepare_query()` and `prepare_document()` to use the prompt-instruction format
and regenerate `embeddings.npz`. See `PERISHABLE_FIGURES.md`.

---

## Attribution

API patterns are adapted from Google's **Gemini API Cookbook**
([`quickstarts/Embeddings.ipynb`](https://github.com/google-gemini/cookbook/blob/main/quickstarts/Embeddings.ipynb),
[`quickstarts/File_Search.ipynb`](https://github.com/google-gemini/cookbook/blob/main/quickstarts/File_Search.ipynb)),
licensed Apache 2.0. The corpus, the planted failures and the notebooks are
original to SWE 485.
# RAG-Tut
