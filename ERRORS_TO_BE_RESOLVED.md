# Errors To Be Resolved

A to-do list of every bug found while checking if this repo actually runs.
Verified by running `poetry install`, `poetry run python main.py`, and
`poetry run pytest` on a clean checkout. Work through these one at a time,
top to bottom — later bugs can't be verified until the earlier ones are fixed
because everything fails at import time first.

Status legend: `[ ]` not started, `[x]` resolved.

---

## 1. `[x]` `langchain_openai` is used but never installed — RESOLVED

**Where:** `graph/chains/retrieval_grader.py`, `graph/chains/generation.py`,
`graph/chains/hallucination_grader.py`, `graph/chains/answer_grader.py`,
`graph/chains/router.py` — all do `from langchain_openai import ChatOpenAI`.

**Problem:** `langchain-openai` is not listed in `pyproject.toml` and is not
present in `poetry.lock`. After a clean `poetry install`, every one of these
files raises `ModuleNotFoundError: No module named 'langchain_openai'`.

**How to resolve:** Add `langchain-openai` to the `dependencies` list in
`pyproject.toml` (a version compatible with `langchain 1.2.0`), then run
`poetry lock` and `poetry install` to regenerate the lock file.

**What was done:** Added `"langchain-openai (>=1.0.0,<2.0.0)"` to
`pyproject.toml`, ran `poetry lock` (resolved `langchain-openai 1.1.6`,
pulling in `openai`, `jiter`, `sniffio` as new transitive deps) then
`poetry install`. Verified with:
```bash
poetry run python -c "from langchain_openai import ChatOpenAI"
```
which now succeeds. Confirmed the fix is complete (not masking a different
problem) by checking that `poetry run pytest`'s failure mode changed from
`ModuleNotFoundError: No module named 'langchain_openai'` to
`openai.OpenAIError: Missing credentials` — i.e. the *next* bug in this list
(#9, no API key set), not this one.

---

## 2. `[x]` `langchain_chroma` is used but never installed — RESOLVED

**Where:** `ingestion.py` — `from langchain_chroma import Chroma`.

**Problem:** Same issue as #1. `langchain-chroma` is not a declared
dependency, so importing `ingestion.py` fails with
`ModuleNotFoundError: No module named 'langchain_chroma'`. Since
`graph/nodes/retrieve.py` imports `ingestion`, this also breaks the whole
graph.

**How to resolve:** Add `langchain-chroma` to `pyproject.toml`, then
`poetry lock && poetry install`.

**What was done:** Added `"langchain-chroma (>=1.0.0,<2.0.0)"` to
`pyproject.toml`, ran `poetry lock` (resolved `langchain-chroma 1.1.0` —
`chromadb` was already a dependency, so this only added the one thin
integration package) then `poetry install`. Verified with
`python -c "from langchain_chroma import Chroma"`, which now succeeds.
`ingestion.py` still fails, but now on the next documented bug (#4, the
broken `langchain.text_splitter` import path) instead of this one.

---

## 3. `[x]` `langchain_tavily` is used but never installed — RESOLVED

**Where:** `graph/nodes/web_search.py` — `from langchain_tavily import
TavilySearch`.

**Problem:** The project depends on the raw `tavily-python` package, but the
code actually imports the separate `langchain-tavily` integration package,
which isn't declared anywhere. Fails with `ModuleNotFoundError: No module
named 'langchain_tavily'`.

**How to resolve:** Add `langchain-tavily` to `pyproject.toml`, then
`poetry lock && poetry install`. (Keep or drop the plain `tavily-python`
dependency depending on whether anything else still needs it directly —
right now nothing does.)

**What was done:** Added `"langchain-tavily (>=0.2.0,<0.3.0)"` to
`pyproject.toml` (kept `tavily-python` too — `langchain-tavily` pulls it in
as its own transitive dependency anyway, and it's harmless either way), ran
`poetry lock` (resolved `langchain-tavily 0.2.17`) then `poetry install`.
Verified with `python -c "from langchain_tavily import TavilySearch"`,
which now succeeds. Loading `web_search.py`'s module-level code now fails
only on the expected next issue — `TAVILY_API_KEY` not set (bug #9) — not
on a missing module.

---

## 4. `[x]` `ingestion.py` imports a module path that no longer exists — RESOLVED

**Where:** `ingestion.py`, line 2 —
`from langchain.text_splitter import RecursiveCharacterTextSplitter`.

**Problem:** In `langchain` 1.x (the version pinned in `pyproject.toml`,
`1.2.0`), text splitters were moved out of the main `langchain` package into
their own `langchain-text-splitters` package. `langchain.text_splitter` no
longer exists, so this raises `ModuleNotFoundError: No module named
'langchain.text_splitter'`.

**How to resolve:** Change the import to:
```python
from langchain_text_splitters import RecursiveCharacterTextSplitter
```
(`langchain-text-splitters` is already pulled in transitively, but consider
adding it to `pyproject.toml` explicitly since the code imports it directly.)

**What was done:** Changed the import as above, and added
`"langchain-text-splitters (>=1.0.0,<2.0.0)"` to `pyproject.toml` as a
direct dependency (it was already present transitively via `langchain`, so
`poetry install` needed nothing new — `poetry lock` just formalized it).
Verified with `python -c "from langchain_text_splitters import
RecursiveCharacterTextSplitter"`. `ingestion.py` now imports cleanly and
gets all the way to its runtime logic — it only fails afterward on a
sandbox network restriction (`WebBaseLoader` can't reach
`lilianweng.github.io` from this environment), which is unrelated to this
bug.

---

## 5. `[x]` `graph/chains/generation.py` imports `hub` from the wrong place — RESOLVED

**Where:** `graph/chains/generation.py`, line 1 — `from langchain import hub`.

**Problem:** In `langchain` 1.x, `hub` is no longer exposed from the
top-level `langchain` package. This raises `ImportError: cannot import name
'hub' from 'langchain'`. This is the first thing that breaks when running
`main.py`, since `graph/graph.py` → `graph/nodes/generate.py` →
`graph/chains/generation.py` pulls it in.

**How to resolve:** Use the standalone `langchainhub` package directly
(already a declared dependency):
```python
import langchainhub
prompt = langchainhub.Client().pull("rlm/rag-prompt")
```
Or, simpler and more robust long-term, drop the `hub.pull()` call entirely
and hardcode the "rlm/rag-prompt" prompt text as a local `ChatPromptTemplate`
so the app doesn't depend on a network call to LangChain's hub just to boot.

**What was done:** Kept the `hub.pull()` approach (network dependency
accepted, per decision to keep prompt fidelity with the live hub rather than
hardcode a possibly-stale copy). Changed the import to
`from langchain_classic import hub` — `langchain-classic` (already pulled in
transitively via `langchain-community`) ships a proper drop-in `hub.pull()`
that returns a real prompt object, unlike calling the low-level
`langchainhub.Client().pull()` directly (which returns a raw manifest
needing manual deserialization). Added `"langchain-classic
(>=1.0.0,<2.0.0)"` to `pyproject.toml` as a direct dependency since the code
now imports it by name. Verified with `python -c "from langchain_classic
import hub"`, and confirmed `graph/chains/generation.py` now imports past
this line entirely — with a dummy `OPENAI_API_KEY` set, `hub.pull()`
correctly attempts a real network call to `api.smith.langchain.com`, blocked
only by this sandbox's network policy, not by any remaining code issue.

---

## 6. `[x]` The vector store is never actually populated — RESOLVED

**Where:** `ingestion.py`, lines 23-28.

**Problem:** The code that loads the three blog posts, splits them, and
builds the Chroma vector store is commented out:
```python
# vectorstore = Chroma.from_documents(
#     documents=doc_splits,
#     collection_name="rag-chroma",
#     embedding=OpenAIEmbeddings(),
#     persist_directory="./.chroma",
# )
```
Only an empty `Chroma(...)` handle pointing at `./.chroma` is created below
it. Even once bugs #1-#5 are fixed and API keys are supplied, `retrieve()`
will always return zero documents, because nothing was ever written to the
store. The whole RAG pipeline effectively has no knowledge base.

**How to resolve:** Either:
- Uncomment the `Chroma.from_documents(...)` call so the store is built on
  every run (simplest, but re-embeds and re-splits on every import — costs
  API calls each time), or
- Add a one-time ingestion step (e.g. a `if not
  os.path.exists("./.chroma"):` guard, or a separate `poetry run python
  ingestion.py` script the README tells users to run once) that builds the
  store only if `./.chroma` doesn't already exist, then have `retriever`
  just open the existing store.

**What was done:** Took the guarded, build-once approach. The
fetch/split/embed block now only runs `if not
os.path.exists(CHROMA_PERSIST_DIR)`; the final `retriever = Chroma(...)`
line (unchanged) always just opens whatever's at `./.chroma`, whether this
run just built it or a previous run already did. Also added `.chroma/` to
`.gitignore` — it's a generated data artifact (a SQLite DB of embeddings),
not source, and shouldn't ever be committed. Verified both branches
directly: with `.chroma/` absent, the build branch correctly triggers (and
attempts the real network fetch, blocked only by this sandbox's network
policy — same as bug #4); with `.chroma/` present, the import skips the
build entirely and no network call is attempted at all.

---

## 7. `[ ]` Importing the graph has a network call + file-write side effect

**Where:** `graph/graph.py`, line 86 —
`app.get_graph().draw_mermaid_png(output_file_path="graph.png")`.

**Problem:** This line runs at **module import time**, not inside a
function. Every time anything does `from graph.graph import app` (including
`main.py` and the test suite), it makes a network call to an external
mermaid-rendering service and writes `graph.png` to disk as a side effect.
This will hang or fail entirely in any offline/sandboxed environment, and is
generally surprising behavior for a module import.

**How to resolve:** Move this line out of module scope — e.g. into an
`if __name__ == "__main__":` block, or a separate small script
(`scripts/render_graph.py`) that's only run manually when someone wants to
regenerate the diagram.

---

## 8. `[ ]` The "adaptive" question router is built but never wired in

**Where:** `graph/chains/router.py` defines `question_router`, but
`graph/graph.py` never imports or calls it.

**Problem:** The README and commit history ("Adaptive RAG / Question
Router") describe the graph as routing each question to either the
vectorstore or a live web search. In the actual graph, the entry point is
hardcoded:
```python
workflow.set_entry_point(RETRIEVE)
```
So every question always goes to `RETRIEVE` first — `question_router` is
dead code and the advertised adaptive-routing behavior doesn't exist.

**How to resolve:** Add a conditional entry point that calls
`question_router` first and routes to `RETRIEVE` or `WEBSEARCH` accordingly,
e.g.:
```python
workflow.set_conditional_entry_point(
    route_question,
    {
        WEBSEARCH: WEBSEARCH,
        RETRIEVE: RETRIEVE,
    },
)
```
where `route_question(state)` calls `question_router.invoke(...)` and
returns `"websearch"` or `"vectorstore"`-mapped node names.

---

## 9. `[ ]` No `.env` / API keys documented as required, but not validated

**Where:** `main.py`, `ingestion.py`, `graph/nodes/web_search.py`, all
`graph/chains/*.py` — all rely on `OPENAI_API_KEY` and `TAVILY_API_KEY`
being present via `load_dotenv()`.

**Problem:** Not a code bug, but worth tracking: there's no `.env` file
(expected — it's gitignored) and no check that the keys actually loaded.
If a user runs `main.py` without creating `.env`, they'll get a raw,
unhelpful API-client error rather than a clear "missing API key" message.

**How to resolve:** Add a small startup check in `main.py` (or a shared
`config.py`) that fails fast with a clear message if `OPENAI_API_KEY` or
`TAVILY_API_KEY` is missing from the environment.

---

## How this list was verified

```bash
poetry install                 # succeeds
poetry run python -c "import ingestion"   # fails on bug #4
poetry run python main.py                 # fails on bug #5
poetry run pytest                         # 0 tests collected, fails on bug #1
```
Fix bugs in order (1 → 9); re-run `poetry run python main.py` after each one
to confirm the next bug in the chain surfaces as expected.
