# Agentic RAG

An **Agentic Retrieval-Augmented Generation (RAG)** pipeline built with [LangGraph](https://github.com/langchain-ai/langgraph) and [LangChain](https://github.com/langchain-ai/langchain). Instead of a single-shot retrieve-then-generate call, the pipeline is modeled as a self-correcting state machine that grades its own retrievals and generations, falling back to a live web search when the local knowledge base isn't good enough.

## How it works

The graph in `graph/graph.py` wires together the following steps:

1. **Retrieve** — fetch relevant chunks from a local Chroma vector store.
2. **Grade documents** — an LLM grader scores each retrieved chunk for relevance to the question, filtering out irrelevant ones.
3. **Decide** — if any documents were filtered out as irrelevant, route to web search; otherwise go straight to generation.
4. **Web search** *(fallback)* — query [Tavily](https://tavily.com/) and add the results to the document set.
5. **Generate** — an LLM produces an answer grounded in the available documents.
6. **Self-check** — the generation is graded for:
   - **Hallucination**: is it actually grounded in the retrieved documents? If not, retry generation.
   - **Answer relevance**: does it address the original question? If not, fall back to web search.

This corrective loop continues until the generation is grounded and useful, at which point the graph ends.

## Project structure

```
main.py                          # Entry point — runs the compiled graph on a sample question
ingestion.py                     # Loads source docs, splits them, and builds/exposes the Chroma retriever
graph/
  graph.py                       # LangGraph StateGraph wiring nodes and conditional edges
  state.py                       # Shared GraphState schema (question, documents, generation, web_search)
  consts.py                      # Node name constants
  nodes/
    retrieve.py                  # Retrieve node
    grade_documents.py           # Document relevance grading node
    generate.py                  # Answer generation node
    web_search.py                # Tavily web search fallback node
  chains/
    retrieval_grader.py          # LLM chain: grades document relevance
    generation.py                # LLM chain: generates the answer
    hallucination_grader.py      # LLM chain: checks generation is grounded in documents
    answer_grader.py             # LLM chain: checks generation answers the question
    router.py                    # LLM chain: routes a question to vectorstore or web search
    tests/                       # Chain tests
```

## Knowledge base

`ingestion.py` ingests three Lilian Weng blog posts into a Chroma vector store (`./.chroma`) using OpenAI embeddings:

- [LLM Powered Autonomous Agents](https://lilianweng.github.io/posts/2023-06-23-agent/)
- [Prompt Engineering](https://lilianweng.github.io/posts/2023-03-15-prompt-engineering/)
- [Adversarial Attacks on LLMs](https://lilianweng.github.io/posts/2023-10-25-adv-attack-llm/)

## Setup

This project uses [Poetry](https://python-poetry.org/) for dependency management and requires Python 3.10–3.12.

```bash
poetry install
```

Create a `.env` file in the project root with the required API keys:

```
OPENAI_API_KEY=your-openai-api-key
TAVILY_API_KEY=your-tavily-api-key
```

## Usage

Run the graph on the sample question ("what is agent memory?"):

```bash
poetry run python main.py
```
