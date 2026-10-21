# Build a Production-Quality, Local-First, Cross-Agent Persistent Memory System

## Role

You are a senior AI infrastructure engineer, systems architect, and Python developer specializing in agent memory, Model Context Protocol (MCP), vector retrieval, local databases, retrieval-augmented generation, and autonomous coding workflows.

Build a complete, production-quality, local-first persistent memory system that enables multiple AI coding agents—including OpenAI Codex, Claude Code, Cursor, and custom MCP-compatible agents—to share durable knowledge across sessions, processes, projects, and agent handoffs.

**This is an implementation task, not merely an architecture proposal.** Produce functioning source code, database migrations, tests, documentation, and verified client integration instructions.

Follow the repository's existing conventions if integrating into an existing codebase. Avoid unrelated changes.

---

# 1. Primary objective

Develop a reusable system tentatively named **Agent Memory Hub**.

The system must support:

1. Persistent memory across separate agent sessions.
2. Shared memory between different coding agents.
3. Semantic retrieval using vector embeddings.
4. Keyword retrieval using SQLite FTS5.
5. Hybrid retrieval combining vector and lexical relevance.
6. Project-level and global memory isolation.
7. Explicit handoffs between autonomous agents.
8. Automated memory capture and consolidation.
9. Memory deduplication and contradiction detection.
10. Incremental updates rather than repeated full indexing.
11. Retrieval within configurable token budgets.
12. Versioned memory with provenance and history.
13. Offline-first operation.
14. Safe concurrent access.
15. Interoperability through MCP.
16. A simple CLI for inspection and maintenance.
17. Backup, restore, export, and import.
18. Comprehensive automated tests.

The application must not depend on a paid cloud API.

Build it as a standalone tool that can be installed globally and reused across repositories.

# 2. Technology stack

Use the following preferred stack:

- Language: Python 3.11+
- Package management: uv
- MCP implementation: official Python MCP SDK, using the current stable documented API
- Primary database: SQLite
- Vector search: sqlite-vec
- Keyword search: SQLite FTS5
- Embeddings: local sentence-transformers model
- Default embedding model: `sentence-transformers/all-MiniLM-L6-v2`, 384 dimensions
- Configuration: TOML
- CLI: Typer
- Validation: Pydantic
- Testing: pytest
- Linting: Ruff
- Type checking: mypy or pyright
- Logging: structured Python logging
- Optional HTTP transport: MCP Streamable HTTP

Do not use LangChain or LlamaIndex unless a concrete requirement justifies the additional dependency.

Use direct, maintainable abstractions wherever possible.

Research the current documentation and pin compatible dependency versions before implementation.

Do not assume that deprecated MCP or vector extension APIs are valid.

# 3. Storage architecture

Use a single SQLite database as the authoritative record of memory.

Suggested default location:

`~/.agent-memory-hub/memory.db`

Use configurable XDG-compatible paths where practical.

Store:

- Memory content
- Memory metadata
- Agent registrations
- Project registrations
- Session records
- Handoff records
- Memory relationships
- Memory revisions
- Embedding metadata
- Audit events
- Indexing jobs

Create and maintain:

1. Ordinary SQLite tables for structured records.
2. FTS5 virtual tables for lexical retrieval.
3. sqlite-vec vector tables for semantic similarity.

The authoritative content and metadata must live in ordinary SQLite tables.

FTS and vector tables are derived indexes, not independent sources of truth.

A database backup must be sufficient to restore the application, including its embeddings and indexes.

If indexes become corrupted or incompatible, they must be rebuildable from authoritative memory records.

Support SQLite WAL mode, foreign keys, bounded busy timeouts, connection lifecycle management, and transactions.

Document the limitation that a SQLite file must not be concurrently written over an unreliable network filesystem.

# 4. Memory taxonomy

Implement the following memory types as explicit metadata, not loosely defined text labels.

## 4.1 Semantic memory

Durable technical facts and knowledge:

- Architecture
- Project structure
- APIs
- Design patterns
- Dependencies
- Coding conventions
- Reusable solutions
- Tool configuration
- User-approved preferences

Example:

"The backend uses FastAPI with a repository pattern and PostgreSQL."

## 4.2 Episodic memory

What happened during a particular task or session:

- Actions taken
- Tests executed
- Failures encountered
- Debugging discoveries
- Files modified
- Solutions attempted
- Outcomes

Example:

"Codex attempted a schema migration, discovered an incompatible column type, and reverted the migration."

## 4.3 Procedural memory

How recurring work should be performed:

- Build procedures
- Testing workflows
- Deployment procedures
- Repository-specific rules
- Troubleshooting sequences
- Approved commands

Example:

"Run database migrations against the temporary test database before running integration tests."

## 4.4 Decision memory

Architectural and product decisions:

- Problem
- Options considered
- Chosen approach
- Rationale
- Trade-offs
- Date
- Source agent
- Revision history

Example:

"SQLite was selected instead of PostgreSQL because the initial deployment is single-user and local-first."

## 4.5 Working memory

Short-lived task context:

- Current objective
- Current plan
- Active blockers
- Open questions
- Next steps
- Working assumptions

Working memory must be scoped to a task or session.

Expired working memory can be summarized and archived rather than permanently retained as active context.

## 4.6 Handoff memory

Structured agent-to-agent transfer records:

- Producing agent
- Intended receiving agent, if known
- Project and task
- Completed work
- Files touched
- Test outcomes
- Remaining tasks
- Known problems
- Required next action
- Relevant memory references

Handoffs must survive process restarts.

# 5. Memory schema

Design a normalized, migration-managed SQLite schema.

Suggested tables:

### projects

- id
- canonical_root
- name
- repository_remote
- repository_identity
- created_at
- updated_at

### agents

- id
- name
- provider
- version
- capabilities
- created_at

### sessions

- id
- project_id
- agent_id
- started_at
- ended_at
- status
- summary

### memories

- id
- project_id, nullable for permitted global memory
- scope
- memory_type
- title
- content
- content_hash
- status
- importance
- confidence
- source_type
- source_reference
- source_agent_id
- source_session_id
- created_at
- updated_at
- last_accessed_at
- expires_at
- revision
- embedding_status

Additional tables:

- memory_versions
- memory_relations
- memory_tags
- memory_tag_links
- memory_embeddings_metadata
- handoffs
- tasks
- audit_events
- indexing_jobs
- schema_migrations

Use stable IDs.

Represent timestamps consistently in UTC.

Track origin and provenance wherever possible.

Supported memory status values should include:

- active
- archived
- superseded
- disputed
- deleted

Use soft deletion by default.

Create explicit relationships such as:

- supports
- contradicts
- supersedes
- relates_to
- derived_from

Prevent orphaned references and enforce database constraints.

# 6. Embedding engine

Build an interchangeable embedding provider interface.

Required methods:

- embed_text
- embed_query
- embed_batch
- get_model_identifier
- get_dimensions

The default implementation must use a locally downloaded sentence-transformers model.

Requirements:

1. Work on macOS, Linux, and Windows.
2. Support CPU inference.
3. Support GPU acceleration where available without requiring it.
4. Reuse an initialized model within a process.
5. Cache embeddings based on normalized content hash and model identity.
6. Batch embedding operations.
7. Avoid regenerating embeddings when text has not changed.
8. Store embedding model name, version, dimensions, and normalization configuration.
9. Detect incompatible embedding dimensions.
10. Provide explicit index migration and re-embedding commands.

Never silently mix embeddings generated by incompatible models in the same vector index.

The first run may download the model, with explicit user notification and instructions for pre-downloading it.

After installation and model caching, normal operations must work without internet connectivity.

Handle embedding model initialization failures gracefully.

# 7. Vector search

Implement semantic similarity search using sqlite-vec.

Support:

- Top-K nearest-neighbor retrieval
- Scope-aware filtering
- Project filtering
- Memory type filtering
- Active-status filtering
- Minimum similarity threshold
- Embedding version validation

Confirm the distance semantics of the installed sqlite-vec version.

Convert distances into consistent ranking signals before hybrid fusion.

Avoid presenting an arbitrary distance conversion as a calibrated probability.

For the initial implementation, exact nearest-neighbor search is acceptable.

Design the index adapter so that a future approximate nearest-neighbor implementation can replace it without changing MCP interfaces.

Do not create a separate vector server.

# 8. Full-text retrieval

Implement lexical search using SQLite FTS5.

Support:

- Keyword search
- Exact phrase search
- Technical symbols and identifiers
- BM25 ranking
- Search over titles and content
- Filtering by project and memory type

Technical identifiers should receive careful treatment.

Examples:

- `auth.middleware`
- `JWT_SECRET`
- `useEffect`
- `src/database/repository.py`

Document how tokenization handles punctuation and symbols.

Use an additional exact-match or substring strategy where FTS tokenization is inadequate.

Maintain FTS indexes automatically when authoritative records change.

# 9. Hybrid retrieval engine

Implement hybrid retrieval as a core feature.

For each request:

1. Validate the caller's permitted memory scopes.
2. Normalize the query.
3. Run vector retrieval.
4. Run FTS retrieval.
5. Optionally retrieve exact matches.
6. Merge candidate sets.
7. Remove duplicate records.
8. Apply hard metadata and authorization filters.
9. Fuse ranked results.
10. Apply bounded secondary boosts.
11. Enforce the token budget.
12. Return ranked memories with provenance.

Use Reciprocal Rank Fusion (RRF) as the default fusion algorithm.

Suggested formula:

`RRF(d) = Σ [weight_i / (k + rank_i(d))]`

Use a configurable RRF constant and retrieval weights.

Avoid directly combining raw BM25 and cosine similarity values without normalization.

Secondary ranking signals may include:

- Memory importance
- Freshness
- Project relevance
- Memory type
- User verification
- Source reliability

Keep secondary boosts bounded so weak but recent memories do not overpower strongly relevant results.

Exclude deleted, expired, and superseded memory by default.

Disputed memories must either be excluded or clearly marked as disputed.

Return a useful explanation of why each record was retrieved.

Example result:

```json
{
  "memory_id": "mem_123",
  "title": "Authentication architecture",
  "content": "Authentication uses JWT...",
  "memory_type": "decision",
  "scope": "project",
  "relevance_score": 0.82,
  "rank": 1,
  "retrieval_sources": ["semantic", "fts"],
  "source_agent": "codex",
  "status": "active"
}
```

The relevance score must be documented as a ranking score, not an empirically calibrated confidence probability.

# 10. Context budget management

Create a retrieval context builder.

Required features:

- Configurable maximum token budget
- Approximate or model-specific token counting
- Ranking-based context selection
- Deduplication
- Compact summaries
- Project context prioritization
- Source references
- Safe truncation

Prioritize:

1. Current task and handoff
2. Relevant project decisions
3. Architecture and constraints
4. Known failures and solutions
5. Relevant global preferences
6. Older episodic context

Provide a default context budget of 4,000 tokens, configurable per request.

Never return the entire database as context.

Never silently truncate a retrieved memory in a way that reverses or materially changes its meaning.

# 11. MCP server

Expose the functionality using the current official MCP Python SDK.

Implement at least these tools:

### memory_search

Inputs:

- query
- project_id
- scopes
- memory_types
- limit
- max_tokens

Returns relevant memory records and provenance.

### memory_add

Creates a new memory record.

Validate all fields and enforce authorization.

### memory_update

Updates a record with optimistic concurrency checking.

Reject stale revisions instead of silently overwriting them.

### memory_get

Retrieves a single memory by ID.

### memory_delete

Soft-deletes a memory.

### memory_list

Lists memory records using bounded pagination and filters.

### memory_link

Creates typed relationships between memories.

### memory_consolidate

Finds potential duplicate or overlapping memories.

The first version must propose merges, not automatically destroy original records.

### memory_handoff_create

Creates a structured handoff.

### memory_handoff_get

Retrieves a handoff by ID.

### memory_handoff_latest

Retrieves the newest relevant handoff for a project or task.

### memory_context

Builds a compact, token-budgeted context packet for a task.

### memory_project_register

Registers a project using a canonical local path and repository identity.

### memory_session_start

Creates a new session record.

### memory_session_end

Closes a session and optionally records a summary.

### memory_stats

Returns database statistics, memory counts, embedding health, and indexing backlog.

### memory_health

Checks database integrity, extension availability, model compatibility, and index synchronization.

Provide structured tool errors.

Use stdio as the default transport for desktop coding agents.

Support Streamable HTTP as an optional transport with explicit binding and access controls.

Do not expose an unauthenticated memory-write service on a public network.

# 12. Cross-agent compatibility

Support the following clients:

- OpenAI Codex
- Claude Code
- Cursor
- Any conforming MCP client

Create documented configuration examples for each supported client.

Verify configuration formats against current official documentation.

Do not invent settings, command-line flags, or unsupported lifecycle hooks.

Agents must connect to the same authoritative database.

If multiple MCP server processes are running locally, they should open the same SQLite database and use transaction-safe access.

Provide a way to verify that two different clients are connected to the same memory store.

The test should:

1. Write a memory through one client.
2. Terminate that client session.
3. Read the memory through a different client.
4. Confirm the content and metadata match.

# 13. Automatic memory lifecycle

Implement a reliable capture workflow.

## Session start

When invoked:

- Identify the project.
- Register the agent and session.
- Retrieve current project context.
- Retrieve relevant architecture decisions.
- Retrieve the latest handoff.
- Return a compact startup context packet.

## During task execution

Agents should be able to record:

- Significant discoveries
- Confirmed constraints
- Architectural decisions
- Important failures
- Reusable solutions

Avoid saving every conversational message.

Prefer concise, durable knowledge over verbose transcripts.

## Session completion

When invoked:

- Record completed work.
- Record unresolved blockers.
- Record test results.
- Create or update the task handoff.
- Store durable learnings.
- Close the session.

An explicit lifecycle command or real supported client hook is required to initiate these operations.

Do not assume all coding agents provide automatic startup, completion, or shutdown callbacks.

Provide a fallback workflow using MCP tools and documented instructions.

A process crash may prevent a normal session-end hook. Support recovery of abandoned sessions.

# 14. Deduplication and consolidation

Avoid uncontrolled memory growth.

Implement three levels of duplicate detection.

### Exact duplicates

Use canonicalized content hashing.

### Near duplicates

Use vector similarity and lexical overlap.

### Contradictions

Identify potentially conflicting statements within the same project and subject.

Similarity does not prove equivalence.

A highly similar statement may contradict another statement.

Never automatically merge contradictory facts.

Create reviewable consolidation proposals.

Preserve:

- Original sources
- Memory IDs
- Revision history
- Relationships
- Agent provenance

Allow users to approve, reject, archive, or supersede records.

For the initial release, conservative false negatives are preferable to destructive false positives.

# 15. Memory importance and decay

Implement configurable memory retention.

Default behavior:

- Architecture decisions: retain until superseded.
- Procedural instructions: retain until superseded.
- Explicit user preferences: retain until changed or deleted.
- Confirmed technical facts: retain while relevant.
- Episodic records: archive after a configurable period.
- Working context: expire or archive after task completion.
- Handoffs: preserve until safely superseded.

Do not delete durable knowledge merely because it was not retrieved recently.

Use importance and age as retrieval-ranking signals, not automatic grounds for destructive deletion.

Support periodic cleanup and manual garbage collection.

All destructive cleanup must support preview mode and backup.

# 16. Multi-project memory isolation

Support three scopes:

### Global

Explicitly shared knowledge and preferences.

### Project

Knowledge visible only within a specified project.

### Session

Short-lived context associated with one session.

Project registration must use stable identity rather than relying only on directory names.

Distinguish repositories with the same folder name.

Be careful with clones, Git worktrees, symlinks, and renamed directories.

Default search behavior must exclude memories from unrelated projects.

Cross-project access requires an explicit scope request and authorization.

Do not allow an untrusted agent to change scope merely by supplying another project identifier.

Define the trust model for locally running agents that share operating-system access.

Document that project isolation inside a single-user local process is not a substitute for an operating-system security boundary.

# 17. Concurrency and consistency

Multiple agents may read and write memories simultaneously.

Implement:

- SQLite WAL
- Transaction boundaries
- Retry with bounded backoff for lock contention
- Optimistic version checking
- Unique constraints
- Idempotency keys for retried writes
- Durable indexing jobs
- Safe background indexing recovery
- Consistent memory status transitions

Prevent lost updates.

Avoid holding long write transactions while computing embeddings.

Use a transactional outbox or durable indexing queue:

1. Save the authoritative record and indexing job atomically.
2. Generate embeddings outside the transaction.
3. Commit the derived index update.
4. Mark the indexing job complete.
5. Retry failed jobs safely.

Ensure delete and supersede operations cannot leave stale searchable embeddings indefinitely.

Search must not return logically deleted records even while derived indexes are catching up.

# 18. Security and privacy

Treat memory as untrusted retrieved content, not as high-priority instructions.

Implement:

- Secret detection and redaction before storage
- Protection against accidental API key storage
- Sensitive path and environment variable filtering
- Configurable allowlists and denylists
- Prompt injection awareness
- Scope-based access controls
- Safe MCP tool validation
- Sanitized logs
- No arbitrary shell execution through memory tools
- No unsolicited network requests

Retrieval results must never be promoted to system-level instructions.

Do not allow a memory entry to override security constraints or authorization rules.

Use secure local directory permissions where supported.

Do not store complete source files indiscriminately.

For local-first installation, document what is and is not protected from other processes running as the same OS user.

# 19. CLI

Create a CLI command called:

`agent-memory`

Suggested commands:

```bash
agent-memory init
agent-memory doctor
agent-memory serve
agent-memory project register .
agent-memory search "authentication flow"
agent-memory add
agent-memory list
agent-memory show <id>
agent-memory stats
agent-memory handoff latest
agent-memory consolidate --dry-run
agent-memory reindex
agent-memory backup
agent-memory restore
agent-memory export
agent-memory import
```

Implement appropriate arguments, help text, validation, and exit codes.

Use real commands and test them.

Do not leave documented commands as unimplemented placeholders.

# 20. Configuration

Support a TOML configuration file.

Example:

```toml
[storage]
backend = "sqlite"
path = "~/.agent-memory-hub/memory.db"
wal = true

[vector]
backend = "sqlite-vec"
embedding_model = "sentence-transformers/all-MiniLM-L6-v2"
dimensions = 384

[retrieval]
mode = "hybrid"
semantic_weight = 0.6
keyword_weight = 0.4
rrf_k = 60
default_limit = 10
max_context_tokens = 4000

[memory]
deduplication_enabled = true
auto_capture_enabled = false
automatic_destructive_merge = false

[mcp]
transport = "stdio"

[security]
default_scope = "project"
allow_cross_project_search = false
redact_secrets = true
```

Ensure every configuration option has actual implementation support.

Validate invalid combinations during startup.

Provide documented defaults.

Do not require the user to configure unnecessary options for a basic installation.

# 21. Suggested repository layout

```text
agent-memory-hub/
├── pyproject.toml
├── uv.lock
├── README.md
├── LICENSE
├── configs/
│   └── example.toml
├── src/
│   └── agent_memory/
│       ├── __init__.py
│       ├── cli.py
│       ├── config.py
│       ├── models.py
│       ├── mcp_server.py
│       ├── storage/
│       │   ├── database.py
│       │   ├── migrations.py
│       │   └── repositories.py
│       ├── embeddings/
│       │   ├── base.py
│       │   └── local.py
│       ├── retrieval/
│       │   ├── semantic.py
│       │   ├── lexical.py
│       │   ├── hybrid.py
│       │   └── context.py
│       ├── memory/
│       │   ├── service.py
│       │   ├── consolidation.py
│       │   ├── lifecycle.py
│       │   └── handoffs.py
│       └── security/
│           ├── redaction.py
│           └── scopes.py
├── migrations/
├── integrations/
│   ├── codex/
│   ├── claude-code/
│   └── cursor/
├── tests/
│   ├── unit/
│   ├── integration/
│   └── e2e/
└── docs/
    ├── architecture.md
    ├── installation.md
    ├── mcp-integration.md
    ├── retrieval.md
    ├── security.md
    └── troubleshooting.md
```

Adjust the structure where necessary to avoid unnecessary abstractions.

# 22. Testing requirements

Use test-first or test-with-the-change development.

At minimum, test:

### Persistence

- Memory survives process restart.
- Data survives MCP server restart.
- Backup and restore retain memory records.
- Database migrations preserve existing data.

### Retrieval

- Exact text retrieval works.
- Semantic retrieval finds paraphrases.
- Hybrid retrieval merges candidates correctly.
- Project filtering works.
- Token budgets are enforced.
- Deleted memories are excluded.
- Superseded memories are excluded by default.
- Unavailable embedding models degrade gracefully to lexical search.

### Consistency

- FTS updates after edits.
- Vectors update after edits.
- Deletions do not leak through stale indexes.
- Reindexing is idempotent.
- Re-embedding handles model changes.

### Concurrency

- Multiple readers work.
- Concurrent writers do not corrupt records.
- Stale revisions are rejected.
- Retried requests do not create duplicate records.

### Security

- Secrets are redacted.
- Unauthorized project access is denied.
- Untrusted memory content is not executed.
- Unsafe path and configuration inputs are rejected.

### MCP interoperability

- MCP tools are discoverable.
- Tool schemas validate inputs.
- Tool responses use expected structures.
- Persistent memory is visible across independent client sessions.

### Handoffs

- Agent A saves a handoff.
- Agent A exits.
- Agent B retrieves the handoff.
- Agent B continues using the correct task context.

Build a small semantic-search evaluation dataset containing technical paraphrases, exact identifiers, and irrelevant distractors.

Report retrieval quality using metrics such as Recall@K and MRR.

Do not claim semantic search quality based solely on tests where the query exactly matches the stored text.

# 23. Performance requirements

Target a single-user local installation containing 10,000–100,000 memory records.

Establish benchmarks for:

- Memory insertion
- Keyword retrieval
- Semantic retrieval
- Hybrid retrieval
- Startup context generation
- Batch embedding generation
- Reindexing
- Concurrent access

Set realistic performance targets after measuring the actual environment.

Avoid hardcoding optimistic latency claims.

Keep model loading out of the critical path where possible.

Document memory consumption, disk footprint, and embedding generation throughput.

# 24. Installation and onboarding

Make installation straightforward.

The ideal first-run workflow:

```bash
uv tool install .
agent-memory init
agent-memory doctor
```

The initialization command should:

1. Create necessary directories.
2. Initialize SQLite.
3. Apply migrations.
4. Check vector extension availability.
5. Configure the default embedding model.
6. Create an example configuration.
7. Display integration instructions.
8. Run a lightweight health check.

Provide a straightforward uninstall procedure that does not silently delete memory data.

# 25. Agent instructions

Generate reusable instructions for agents.

Agents should follow these principles:

### Before starting

- Identify the project and current task.
- Retrieve relevant memories.
- Read the latest applicable handoff.
- Check existing decisions before changing architecture.

### During execution

- Prefer current repository evidence over potentially stale memories.
- Preserve provenance.
- Save confirmed discoveries.
- Avoid repeatedly storing the same fact.
- Do not store secrets.
- Resolve contradictions conservatively.

### Before finishing

- Record the task outcome.
- Store durable learnings.
- Create a structured handoff.
- Record outstanding issues.
- Include test results.
- Avoid saving unnecessary intermediate reasoning.

Memory is advisory context.

The checked-out repository, validated tests, explicit user instructions, and current tool outputs take precedence over stale remembered facts.

# 26. Incremental delivery plan

Implement in these stages.

## Phase 1 — Core persistence

Deliver:

- Project skeleton
- Configuration
- SQLite schema
- Migrations
- CRUD operations
- Memory versioning
- CLI
- Unit tests

Do not advance until persistence tests pass.

## Phase 2 — Retrieval

Deliver:

- Local embeddings
- sqlite-vec integration
- FTS5 integration
- Hybrid ranking
- Filtering
- Token-budgeted context builder
- Retrieval evaluation tests

## Phase 3 — MCP

Deliver:

- MCP server
- Read/write/search tools
- Structured responses
- Client configuration examples
- End-to-end persistence test

## Phase 4 — Agent lifecycle

Deliver:

- Session registration
- Task tracking
- Handoff creation
- Handoff retrieval
- Explicit capture workflows
- Verified supported hooks where available

## Phase 5 — Memory intelligence

Deliver:

- Deduplication
- Consolidation proposals
- Contradiction management
- Expiration and archival
- Audit trail
- Index recovery

## Phase 6 — Production hardening

Deliver:

- Concurrency tests
- Security tests
- Backup and restore
- Health diagnostics
- Performance benchmarks
- Documentation

Keep each phase independently testable.

# 27. Definition of done

The project is complete only when:

1. A fresh local installation succeeds.
2. SQLite schema and migrations work.
3. Embeddings are stored persistently.
4. Semantic search returns meaningful results.
5. Hybrid retrieval works correctly.
6. A running MCP server exposes the required tools.
7. Independent clients can share the same database.
8. Agents can exchange persistent handoffs.
9. Memory survives application restart.
10. Concurrent operations are safe.
11. Project boundaries are enforced.
12. Backups can be restored.
13. Relevant automated tests pass.
14. Linting and type checking pass.
15. Integration documentation is accurate.
16. The final code changes are reviewed for correctness and security.

Do not mark work complete based only on code generation.

Run the tests and provide the actual results.

If a test cannot be run, report that limitation clearly.

# 28. Final deliverables

Provide:

- Complete working source code
- Reproducible dependency lockfile
- Database migrations
- MCP server
- CLI
- Automated tests
- Example configuration
- Codex integration instructions
- Claude Code integration instructions
- Cursor integration instructions
- Example cross-agent workflow
- Backup and restore guide
- Troubleshooting guide
- Architecture documentation
- Security limitations
- Performance benchmark results
- Known limitations and next steps

Finish with a real demonstration:

1. Start the memory service.
2. Create a project.
3. Save a technical decision as Agent A.
4. Save a handoff.
5. Exit the originating process.
6. Start a separate Agent B session.
7. Retrieve the decision through hybrid search.
8. Retrieve the handoff.
9. Update the memory using version checking.
10. Restart the service and verify persistence.

If actual Codex, Claude Code, or Cursor instances are unavailable in the environment, test two independent MCP clients and clearly distinguish that interoperability test from an unverified vendor-specific integration.

## Implementation instructions

Start by inspecting the repository and development environment.

Read relevant official documentation before implementing unfamiliar APIs.

Write a short technical design and implementation plan, then immediately begin implementing Phase 1.

Follow established coding conventions, write tests alongside implementation, and run tests after each phase.

Choose sensible defaults rather than blocking progress on nonessential configuration questions.

Do not build a web dashboard until the core system is complete and tested.

Do not substitute mock implementations for required functionality.

Do not omit difficult requirements silently.

At the end, report what was implemented, verified, partially implemented, and not completed.

**The primary success criterion is simple: any supported coding agent should be able to learn something important, terminate, and have another agent retrieve and correctly use that knowledge in a later session without relying on conversational context.**
