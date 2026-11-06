# Agent-Infrastructure Principles

Build agent workflows for how they'll actually be used: by autonomous agents that are always online, working in parallel, at machine speed.

These principles come from Anthropic's migration of a 25 GB monorepo from git to a purpose-built system (Scion) under the load of 30,000+ agent-merged changes per day.

## Principles

1. **Sparse context loading** - Search before reading; load only what you need
2. **Client-server asymmetry** - Offload reasoning to the LLM; keep Python thin
3. **Derived data caching** - Cache by content hash; never recompute identical work
4. **Always online** - Reference by path, not copy; fetch when needed
5. **Content-addressed deduplication** - Store by hash, not by producer
6. **Atomic operations** - Write to temp, then rename; never write in place

## Source

Derived from [Anthropic's research](https://www.anthropic.com/research/redesigning-source-control) on Scion, their in-house source-control system.

## License

MIT - see LICENSE file.
