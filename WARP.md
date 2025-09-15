# WARP.md

This file provides guidance to WARP (warp.dev) when working with code in this repository.

Project: Basic QA Bot (Python, Gradio, LangChain, ChromaDB)

Quick start
- Python: Use Python 3.10+.
- Create a virtual environment and install dependencies listed in README.md.

Setup
- Create and activate a virtual environment:
  - macOS/Linux: python3 -m venv .venv && source .venv/bin/activate
  - Windows (PowerShell): py -m venv .venv; .venv\Scripts\Activate.ps1
- Install runtime dependencies (from README.md):
  - pip install gradio langchain langchain-ibm langchain-community chromadb pypdf

Run the app
- Start the Gradio UI:
  - python qabot.py
- Default UI will be served on http://127.0.0.1:7860

Linting and formatting
- This repo does not include configured linters/formatters. If you add them, common commands are:
  - Ruff (lint): ruff check .
  - Black (format): black .

Testing
- There are no tests in the repository yet. If you add pytest-based tests:
  - Run all tests: pytest -q
  - Run a single test file: pytest -q path/to/test_file.py
  - Run a single test: pytest -q path/to/test_file.py::test_name

High-level architecture and flow
- Purpose: Provide a simple RAG-style QA bot that lets a user upload a PDF, builds a vector store over its contents, and answers questions via an LLM-backed RetrievalQA chain. The Gradio app is the UI.
- Primary file: qabot.py
  - Document loading: document_loader(file)
    - Uses langchain_community.document_loaders.PyPDFLoader to load PDF pages into LangChain Document objects.
  - Chunking: text_splitter(data)
    - Uses RecursiveCharacterTextSplitter with chunk_size=1000 and chunk_overlap=200 to split Documents into chunks.
  - Embeddings and vector store: watsonx_embedding() and vector_database(chunks)
    - Intended to provide an embeddings model and construct a Chroma vector store from the chunks.
  - Retrieval: retriever(file)
    - Builds a retriever from the Chroma vector store via vectordb.as_retriever().
  - LLM + QA chain: get_llm() and retriever_qa(file, query)
    - Intended to instantiate an LLM and wire it with the retriever into a RetrievalQA chain (chain_type="stuff").
  - UI: Gradio Interface
    - Inputs: a single PDF file and a query textbox.
    - Output: a textbox with the model’s answer.

Important implementation notes and gaps to resolve
- LLM instantiation is incomplete:
  - get_llm() returns llm but llm is never defined; you must instantiate a concrete LLM (e.g., from langchain, langchain-openai, langchain-ibm, etc.) and return it.
- Embeddings class references are incomplete/mismatched:
  - watsonx_embedding() references LLMEmbedding but that class is not imported or defined in this repo; choose a concrete embeddings class (e.g., from langchain_openai or langchain_community) and import it.
  - vector_database() calls llm_embedding(), but the function is named watsonx_embedding(); update to call the correct function or rename consistently.
- Gradio file input type vs. loader expectation:
  - The Gradio File input is configured with type="filepath" which passes a string path. document_loader currently expects file.name; update document_loader to accept a path string (or change the Gradio input to return a tempfile object and adjust accordingly).
- Chroma persistence (optional):
  - Chroma can run in-memory or with a persist_directory; the current code creates a transient store each invocation. Decide if you want persistence and set persist_directory accordingly.
- RetrievalQA invocation shape:
  - RetrievalQA.from_chain_type(...).invoke(query) can accept a string, but depending on chain setup you may prefer qa.invoke({"query": query}) for clarity.

Operational tips
- Environment variables/keys:
  - Depending on your chosen LLM/embeddings provider, you will likely need to set provider-specific API keys in your shell (e.g., export PROVIDER_API_KEY=...). Wire these into get_llm() and your embeddings class as needed.
- Local development with auto-reload (optional):
  - For quick iteration you can wrap the Gradio launch in if __name__ == "__main__": and use gradio’s reload=False/True per preference.

Files of interest
- qabot.py: End-to-end app including data prep, retrieval, and UI.
- README.md: Short description and dependency list.

How future agents should proceed when modifying the app
- When implementing the missing LLM and embeddings, update imports and function bodies in qabot.py without changing the public function signatures unless necessary.
- If adding persistence or configuration, prefer a small settings block or environment variables over hard-coding values in multiple places.
- If adding tests, create a tests/ directory and use pytest; consider dependency injection for LLM/retriever to make components testable without external calls.
