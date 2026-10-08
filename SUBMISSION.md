# daily.dev Marketplace submission

Paste these into Marketplace → Submit.

## Name

Upgrade Blast Radius

## Short pitch

Before you bump a dependency, see exactly where it breaks. Pulls migration gotchas from daily.dev posts and comment threads, maps each one to the file:line call sites in your repo with a Graphify code graph, and gives you a verdict. Read only: nothing changes until you do.

## About page

**What it does**

You ask your agent "is it safe to upgrade httpx?" or "what breaks if I bump next to 16?". Upgrade Blast Radius answers two questions:

1. **What changed.** It searches daily.dev for migration guides, breaking-change posts and release write-ups for that library and version, reads the top posts and their comment threads, and writes down each concrete change with its source link.
2. **Where it hits your code.** It reads a [Graphify](https://github.com/Graphify-Labs/graphify) graph of your repo. Graphify records every third-party import with its file and line, so the agent knows every file that uses the library. It then searches only those files for each changed API.

You get a report: a verdict (Safe, Needs changes or Risky), a table of each gotcha with the file:line it hits and the post it came from, the gotchas it checked and found nothing for, and what daily.dev did not cover.

**What makes it different from a plain upgrade summary**

A summary tells you what changed in the library. This tells you which of those changes touch your code, and where. Changes that don't touch your code are listed as checked, with the search term used, so you can trust the result.

**Rules it follows**

- Read only. It never edits your code, lockfile or manifests. It saves the source posts to a daily.dev bookmark list only if you say yes.
- Every gotcha links to a daily.dev post or comment, and every location comes from the graph or a search. When daily.dev has nothing on a version, the report says so instead of filling the gap.
- Your daily.dev token is only sent to api.daily.dev.

**What you need**

- daily.dev API access: the daily.dev MCP server (`https://api.daily.dev/mcp`) or a personal token from Settings → API (Plus).
- A Graphify graph of your repo. If you don't have one, the agent offers to build it: `uv tool install graphifyy`, then `graphify extract . --code-only`. That build runs locally with tree-sitter and needs no LLM API key.

Tested on Python and TypeScript repos. Other languages Graphify parses record imports the same way, but have not been tested with this skill yet.

## Link

https://github.com/SyedFahad7/upgrade-blast-radius

## Agent instructions (SKILL.md)

Paste the contents of `SKILL.md` from this repo.
