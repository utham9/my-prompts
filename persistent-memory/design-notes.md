# Agent Memory Hub design notes

These notes accompany [the master implementation prompt](agent-memory-hub.md) from the [Set Up Persistent Memory conversation](https://chatgpt.com/c/6ad87d87-7420-83ee-9ac6-fc1005f715ed).

## Choices from the original answer

- **One database:** Use SQLite with `sqlite-vec` for authoritative records, metadata, full-text search, and persistent embeddings. This simplifies backups and avoids synchronizing two databases. Keep the vector layer replaceable if later requirements call for another index.
- **Hybrid retrieval:** Combine vector search for conceptual questions with FTS5 for exact names, errors, and paths. Fuse ranked results instead of combining raw similarity and BM25 scores.
- **Controlled learning:** Preserve confirmed decisions, useful failures, and task outcomes. Avoid saving every intermediate message, and keep memory reviewable so stale or incorrect claims can be corrected.
- **Explicit lifecycle:** Retrieve relevant memory when a task starts and write a structured handoff when it ends. Persistent embeddings alone do not ensure another agent will use the knowledge.

The prompt specifies the detailed implementation and verification requirements; these notes are a short guide to its core trade-offs.
