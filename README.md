# Upgrade Blast Radius

An agent skill for the [daily.dev Marketplace](https://daily.dev/marketplace). Before you bump a dependency, it finds out **what changed** from daily.dev posts and comment threads, and **where that hits your code** from a [Graphify](https://github.com/Graphify-Labs/graphify) graph of your repo. Then it gives you a verdict.

```
You: is it safe to move this repo from httpx to httpx2?
Agent: (searches daily.dev, reads the graph, searches the importing files)
       Verdict: Risky. 6 except clauses catch standalone httpcore errors...
```

It is read only. It never edits your code, lockfile or manifests.

## How it works

1. **Inputs.** Library, current version (read from your lockfile) and target version or replacement package.
2. **Call sites.** Graphify records every third-party import as a node, with an `imports` edge that carries the importing file and line. A short script in the skill lists every file that imports the library.
3. **Gotchas.** The agent searches daily.dev (`/search/posts`), reads the top posts and their comments, and writes down each concrete change with its source link.
4. **Match.** For each changed API, it runs a whole-word search over only the importing files. Environment changes (TLS, drivers, language version) are checked in Dockerfiles, CI and version pins.
5. **Report.** Verdict, a table mapping each gotcha to file:line and its source post, likely hits (labeled as inference), gotchas checked with no hits, and what daily.dev did not cover.
6. **Optional.** Saves the source posts to a daily.dev bookmark list, only if you say yes.

## Setup

**daily.dev**: connect the daily.dev MCP server (`https://api.daily.dev/mcp`), or create a personal token at [app.daily.dev/settings/api](https://app.daily.dev/settings/api) (Plus) and set it as `DAILY_DEV_TOKEN`. The skill only sends the token to `api.daily.dev`.

**Graphify**: you need `graphify-out/graph.json` in your repo. If you don't have one, build it. This runs locally with tree-sitter and needs no LLM API key:

```bash
uv tool install graphifyy      # PyPI package name has two y's
graphify extract . --code-only
```

**The skill**: copy [`SKILL.md`](SKILL.md) into your agent's skills folder (for example `~/.claude/skills/upgrade-blast-radius/SKILL.md` or `.cursor/skills/upgrade-blast-radius/SKILL.md`), or paste it from the daily.dev Marketplace.

## Example: httpx-ws, from httpx to httpx2

A real run against [frankie567/httpx-ws](https://github.com/frankie567/httpx-ws) at commit `40ded28` (2026-08-12), a small library that adds WebSockets to HTTPX. The graph came from `graphify extract . --code-only` (352 nodes, about 10 seconds). For this run, the daily.dev posts were found with web search and read through daily.dev's public post pages (`/posts/<slug>.md`), which carry the same summary and Q&A content. Your own runs use the daily.dev API search.

> ## Upgrade Blast Radius: httpx → httpx2
>
> **Verdict: Risky.** httpx-ws is built on httpcore's own stream and exception classes, and daily.dev reports that httpx2 vendors httpcore. No post says what that means for code that imports httpcore directly.
>
> **Where httpx is used:** 5 files (3 library, 2 tests). httpcore is imported directly in 3 files (2 library, 1 test).
>
> | Gotcha | Change | Hits in this repo | Source |
> |---|---|---|---|
> | Certificate trust | httpx2 uses the operating system trust store instead of the certifi bundle. Can break minimal containers without system CA certificates, TLS-inspecting proxies, and custom certifi bundles. Fix: set `SSL_CERT_FILE` or `SSL_CERT_DIR`, or pass an `ssl.SSLContext` via `verify`. | None in code. No `verify`, `cert`, `SSL_CERT_FILE`, `SSLContext` or `certifi` in the 5 files. Applies to whoever deploys code using httpx-ws. | [#494: Python Wrapture](https://daily.dev/posts/494-python-wrapture-vsednyhjv) |
>
> **Likely (inferred):** httpx2 vendors httpcore ([httpx2!](https://daily.dev/posts/httpx2--hgnk03uib)). httpx-ws catches the standalone package's exceptions at `httpx_ws/_api.py:185, 462, 516, 725, 1033, 1087` (`httpcore.WriteError`, `httpcore.ReadError`), and imports `AsyncNetworkStream` and `NetworkStream` from httpcore at `httpx_ws/_api.py:16` and `httpx_ws/transport.py:8`. If httpx2 raises and returns its vendored copies of these classes, those `except` clauses would not catch them. The tests raise standalone httpcore errors in 12 places in `tests/test_api.py` (first at line 65).
>
> **Not applicable:** zstd compression on Python 3.14+ is a new feature, not a breaking change.
>
> **Not covered on daily.dev:** httpx2's import name, whether its public API matches httpx 0.28 beyond TLS, and whether the vendored httpcore classes are exposed for callers.
>
> **Sources:** [#494: Python Wrapture](https://daily.dev/posts/494-python-wrapture-vsednyhjv) (Planet Python, 1 upvote), [httpx2!](https://daily.dev/posts/httpx2--hgnk03uib) (Planet Python), [Why I forked httpx](https://daily.dev/posts/why-i-forked-httpx-cfsr4olye) (Hacker News, background on the forks).

Every location in that report was checked against the source.

## Limits

- It is only as good as daily.dev's coverage of that version. When coverage is thin, the report says so and the verdict leans to Risky.
- "Likely" rows are the agent's inference, and they are labeled that way. Read them as "check this", not "this breaks".
- The graph lists the files that import a library. Code that reaches it only through another package (transitive use) needs a separate run for that package.
- Tested on Python and TypeScript repos. Other languages Graphify parses record imports the same way, but are untested here.

## License

MIT. Graphify is a separate open-source project by Graphify Labs, Apache-2.0, installed from PyPI as `graphifyy`.
