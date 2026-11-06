---
name: agent-infrastructure-principles
version: 1.0.0
author: Lesley
license: MIT
description: Use when building agent pipelines or LLM workflows.
metadata:
  tags:
    - agents
    - llm
    - infrastructure
    - optimization
  related_skills:
    - autonomous-ai-agents
---

# Agent-Infrastructure Principles

Build agent workflows for how they'll actually be used: by autonomous agents that are always online, working in parallel, at machine speed. These principles come from Anthropic's migration of a 25 GB monorepo from git to a purpose-built system (Scion) under the load of 30,000+ agent-merged changes per day. They generalize to any LLM-driven pipeline: file processing, research, code generation, data mining.

## The six principles

### 1. Sparse context loading
**Principle:** The working copy is optional. Materialize only what you actually read.

Anthropic's CI jobs check out sparse trees and materialize files on demand. A 25 GB repo never touches most of its data in any single run.

**Do:**
- Search before reading. `grep`/`search_files` to locate the relevant lines, then read a narrow window with context.
- Never `cat` an entire file into the context when you need 20 lines of it.
- Ask the LLM for summaries of large files instead of loading them. A 10k-token file costs 10k tokens to read, 100 tokens to summarize.
- For multi-file tasks, extract only the function signatures and docstrings into context; load bodies on demand.

**Anti-pattern:** Reading every file in a directory to understand the codebase. Use a directory listing + targeted searches instead.

### 2. Client-server asymmetry: offload to the LLM
**Principle:** The server does heavy computation; the client is a thin wrapper.

In Scion, the server answers history/blame/merge queries; the client just displays and edits. The client doesn't re-implement git's algorithms.

**Do:**
- Let the LLM do reasoning, pattern matching, and summarization. It has GPU acceleration your Python script doesn't.
- Use Python only for file I/O, math, regex substitutions, and data shuffling.
- If a script exceeds ~30 lines of logic, ask: could the LLM do this in one prompt? Usually yes, and faster.

**Example:** Instead of a 50-line Python parser to extract error codes from logs, ask the LLM: "Extract all error codes from these logs and group them by severity."

**Anti-pattern:** Writing complex data-transformation code when a single LLM prompt would do it more robustly.

### 3. Derived data caching
**Principle:** Compute derived data recursively from its parents; cache aggressively; compute lazily when expensive.

Scion's "derived data" is blame, history, and indices computed from changesets. Each type can be eager or deferred.

**Do:**
- Cache intermediate analysis results to disk (JSON, SQLite) rather than recomputing them.
- Use a content-addressed cache: store by hash of input, so identical inputs never recompute.
- In conversation context, summarize old sections rather than keeping verbatim. The LLM doesn't need every word from turn 3 to answer turn 20.
- For LLM pipelines, cache subagent outputs keyed by input hash. Re-running a failed job should reuse all successful subjobs.

**Anti-pattern:** Re-reading and re-summarizing the same file every time a subagent runs. Cache the summary.

### 4. Always online: reference by path, not copy
**Principle:** Don't localize what can be remote. Fetch when needed.

Scion's clients don't hold a full repo copy. They pull blobs from the cache server when needed, and the daemon maintains the node-local cache.

**Do:**
- Keep large files on disk, reference them by path. Don't copy them into the agent's working directory.
- Don't download documentation into context when you can query the LLM's knowledge about it.
- Let the LLM server be the source of truth. It's local and fast; no need to replicate its data.
- Subagents should reference shared artifacts by path, not copy them.

**Anti-pattern:** `cp large_dataset.csv ./subagent_workspace/` — burns disk and RAM. Instead, pass the path to the subagent.

### 5. Content-addressed deduplication
**Principle:** All data is content-addressed; any cache can serve it without coordination.

In Scion, a blob stored under hash X is immutable and identical everywhere. No two servers need to sync.

**Do:**
- When multiple subagents produce similar artifacts, store by content hash, not by producer.
- Use a shared content store: `.store/$(sha256sum file | cut -d' ' -f1)/` rather than `./agent1/output/file` and `./agent2/output/file`.
- This is free deduplication and makes caching safe across agents.

**Implementation sketch:**
```bash
store_by_hash() {
    local file="$1"
    local hash=$(sha256sum "$file" | cut -d' ' -f1)
    local store=".content-store/$hash"
    mkdir -p "$store"
    cp -n "$file" "$store/"  # no-op if exists
    ln -sf "$store/$(basename $file)" "$file.link"
    echo "$file.link"
}
```

### 6. Cheap, incremental, atomic operations
**Principle:** Commits scale with the size of the change, not the repo. Everything is lock-free and atomic.

Scion creates changesets on a write-ahead log. No packfile generation, no locking, no global state.

**Do:**
- Write to a temp file, then `mv` to the final location (atomic rename). Never write in place.
- Batch small edits into one write rather than N individual writes.
- Don't rewrite entire files when only a few lines changed. Use `patch` or targeted edits.
- For LLM subagents, make each job's output atomic: write to `tmp/`, then rename to `output/` when done.

**Anti-pattern:** `open(file, "w")` to update one line in a large file. This truncates the file first — if the process crashes, the data is gone. Use temp+rename.

## Applying the principles

**For a data-mining pipeline:**
1. Search files to find relevant chunks, don't read whole files (sparse).
2. Let the LLM do the pattern matching; Python just shuffles data (offload).
3. Cache LLM responses keyed by input hash; reuse across subagents (derived data).
4. Pass file paths to subagents, not file contents (always online).
5. Store artifacts in a content-addressed store; dedupe automatically.
6. Write results atomically: `tmpfile` → `mv` (atomic).

**For a code-generation pipeline:**
1. Sparse checkouts: only load the files the agent is modifying.
2. The LLM generates code; your script runs it and captures output (asymmetry).
3. Cache test results keyed by file hash; skip tests if unchanged.
4. Reference the codebase by path, don't clone it per agent.
5. Dedupe generated artifacts by content hash.
6. Commit atomically: stage all changes, then push in one operation.

## Anti-patterns to avoid

| Anti-pattern | Violates | Fix |
|---|---|---|
| Reading whole files into context | Sparse loading | Search, then read narrow windows |
| Python logic when LLM prompt would work | Asymmetry | Let the LLM do the reasoning |
| Recomputing summaries every run | Derived data | Cache by input hash |
| Copying files per subagent | Always online | Pass paths, not copies |
| Duplicating artifacts across agents | Deduplication | Content-addressed store |
| In-place file writes | Atomicity | Temp + rename |

## Scaling notes

These principles become critical at scale:
- **10+ concurrent subagents:** Deduplication and atomic writes prevent race conditions.
- **100k+ token contexts:** Sparse loading is mandatory; you can't afford to load the whole file.
- **Multi-hour pipelines:** Caching and content addressing prevent re-running expensive steps after failures.

The underlying insight: **build for agents, not humans.** Agents are always online, work in parallel, and fail often. Optimize for cheap retries, shared state, and minimal context usage — not for offline work or single-threaded review.

## Source

Derived from Anthropic's research blog post: [Redesigning source control at Anthropic](https://www.anthropic.com/research/redesigning-source-control), updated October 15, 2026.

This skill translates the architectural principles from Scion (Anthropic's in-house version control system) into practical guidance for building LLM agent pipelines.
