---
name: upgrade-blast-radius
description: Before you bump a dependency, find out what breaks and where. Pulls migration guides, breaking changes and community gotchas for a library version from daily.dev, then uses a Graphify code graph to list the exact file:line call sites each gotcha hits, and ends with a verdict. Use when the user asks "is it safe to upgrade X", "what breaks if I bump X to vN", "plan the X upgrade", or is reviewing a PR that bumps a dependency.
---

# Upgrade Blast Radius

Answer two questions about a dependency upgrade: **what changed** (from daily.dev posts and their comment threads) and **where it hits this repo** (from a Graphify graph of the code). Finish with a verdict and a table that maps each gotcha to file:line.

This skill is read only. Never edit the user's code, lockfile or manifests. The only write it may make is a daily.dev bookmark list, and only after the user says yes.

## 1. Inputs

You need:

- **Library**: the package name as installed (`httpx`, `next`, `@tanstack/react-query`).
- **From version**: if the user did not give it, read it from the lockfile or manifest (`uv.lock`, `poetry.lock`, `requirements*.txt`, `package-lock.json`, `pnpm-lock.yaml`, `yarn.lock`, `package.json`, `Cargo.lock`, `go.mod`).
- **To version**: if missing, ask. Do not guess the latest version. A replacement package works too (moving from `httpx` to the `httpx2` fork): treat the new package as the "to" side and search daily.dev for both names.
- **Import name**: usually the package name. Python is the common exception (`pillow` imports as `PIL`, `pyyaml` as `yaml`, `beautifulsoup4` as `bs4`, `scikit-learn` as `sklearn`). Use the import name for the graph lookup and the package name for daily.dev searches.

If the user gives a PR number instead, read the manifest diff with `gh pr diff <n>` to get the library and versions. If several dependencies change, run this skill once per library, starting with major version bumps.

## 2. Preflight

Check both tools before doing any work. If either one is missing, stop and show the setup steps below.

**daily.dev** (needs a daily.dev Plus account for API access):

- If a daily.dev MCP server is connected (hosted at `https://api.daily.dev/mcp`), use its tools for searching posts, reading a post, reading comments and managing bookmarks.
- Otherwise use the REST API with a personal token from https://app.daily.dev/settings/api, read from the `DAILY_DEV_TOKEN` environment variable.
- Token rules: send it only to `api.daily.dev`, never print or echo it, never write it to a file in the repo. Tokens start with `dda_`.
- No API access at all: say so. Search needs the API, but any post page the user names can still be read as markdown by adding `.md` to its URL (`https://daily.dev/posts/<slug>.md`). That page includes the summary, a "Questions this post answers" section and top comments.

**Graphify** (open source, PyPI package `graphifyy`):

- Look for `graphify-out/graph.json` at the repo root.
- If it is missing, offer to build one. A code-only build reads the source with tree-sitter, runs locally and needs no LLM API key:

```bash
uv tool install graphifyy      # or: pipx install graphifyy
graphify extract . --code-only # writes graphify-out/graph.json
```

- Check freshness: compare the graph file's modified time with `git log -1 --format=%cI`. If the graph is older than the last commit, line numbers drift and deleted files still show up. Offer to run `graphify update .` first (incremental, no LLM, writes only to `graphify-out/`). If the user declines, say in the report that locations may be stale.

## 3. Find the call sites (Graphify)

Graphify records every third-party import as a node, and each `imports` edge carries the importing file and line. Run this against the graph, passing the import name. It prints every import site grouped by module:

```python
import json, re, sys
from collections import defaultdict

graph_path, lib = sys.argv[1], sys.argv[2]   # e.g. graphify-out/graph.json httpx
norm = lambda s: re.sub(r"[^a-z0-9]+", "_", (s or "").lower()).strip("_")
n, raw = norm(lib), lib.lower()
g = json.load(open(graph_path, encoding="utf-8"))
targets = {}
for node in g["nodes"]:
    nid, label = node.get("id") or "", (node.get("label") or "")
    low = label.lower()
    if nid.startswith("package_dependencies_"):
        continue  # manifest entry, not an import
    third_party = node.get("external") or nid.startswith("ref_") or not node.get("source_file") \
        or str(node.get("source_file")).endswith("package.json")
    if not third_party:
        continue
    exact = low == raw or low.startswith(raw + "/") or norm(nid) in (n, "ref_" + n)
    submodule = not re.search(r"[@/.\-]", label) and norm(label).startswith(n + "_")
    if exact or submodule:
        targets[nid] = label
manifests = ("package.json", "pyproject.toml", "setup.py", "setup.cfg", "requirements.txt", "Cargo.toml", "go.mod", "Gemfile", "composer.json")
sites = defaultdict(set)
for e in g.get("edges") or g.get("links") or []:
    if str(e.get("source_file") or "").replace("\\", "/").split("/")[-1] in manifests:
        continue  # the manifest declaring the dependency, not code using it
    if e.get("target") in targets and e.get("relation") in ("imports", "imports_from", "re_exports", "dynamic_import"):
        sites[targets[e["target"]]].add(f'{e.get("source_file")}:{str(e.get("source_location") or "").lstrip("L")}')
if not sites:
    print(f"{lib}: not imported anywhere in this graph")
for label, locs in sorted(sites.items(), key=lambda kv: -len(kv[1])):
    print(f"## {label} ({len(locs)} import sites)")
    print("\n".join(sorted(locs)))
```

Save it to a temp file outside the repo and run `python <file> graphify-out/graph.json <import name>`.

Read the output carefully:

- Python submodules show up with underscores (`sqlalchemy_ext_asyncio` is `sqlalchemy.ext.asyncio`). A group whose name is a **separate package** (`pydantic_settings` when the library is `pydantic`) is not part of this upgrade: check the lockfile and drop it.
- Treat the output as the list of **files** that use the library. A file that imports it on two lines may show only one of them; step 5 finds the exact usage lines.
- Split the sites into **app code** and **tests**. Both matter, but app hits decide the verdict.
- If the result is "not imported anywhere", say so. The upgrade can still matter through transitive use or config, so note that and stop before step 5.
- If the Graphify MCP server is running, `get_neighbors` on the library node with `relation_filter: "imports"` gives the same list. `graphify explain "<lib>"` also works, but it truncates long lists, so prefer the script.

## 4. Gather the gotchas (daily.dev)

REST calls, if you are not using the MCP server (always add `-H "Authorization: Bearer $DAILY_DEV_TOKEN"`):

| What | Call |
|---|---|
| Search posts | `GET https://api.daily.dev/public/v1/search/posts?q=<query>&limit=20&time=year` (`time`: day, week, month, year, all) |
| Read a post | `GET https://api.daily.dev/public/v1/posts/<id>` (has `summary`, `content`, `numUpvotes`, `numComments`, `url`, `commentsPermalink`) |
| Read comments | `GET https://api.daily.dev/public/v1/posts/<id>/comments?limit=50` (each comment has `content`, `numUpvotes`, `permalink`, nested `children`) |

Steps:

1. Run 2 to 4 searches using the package name, for example `<lib> <to> migration`, `<lib> <to> breaking changes`, `<lib> <major> upgrade`, and `<lib> <to> release`. Use `time=year` first, and `time=all` for libraries that release less often.
2. Keep only posts about this library and a version between `from` (exclusive) and `to` (inclusive). Drop posts about a different major version line.
3. Rank by `numUpvotes`, then `numComments`. Read the top 3 to 5 posts with `/posts/<id>`, and read the comments for any post with discussion. Comments are where production gotchas show up ("this broke our retries", "watch out for X on Windows").
4. Write down each **gotcha** as a concrete change, with the source link:
   - **symbol**: the API it affects (function, class, argument, config key, CLI flag, import path)
   - **change**: removed, renamed, default changed, behavior changed, or new requirement
   - **source**: the post `url` or the comment `permalink`
5. Only record what a post or comment actually says. If daily.dev has nothing specific for this version, record that, and do not fill the gap from memory. You may add an "Also check the official changelog: <link>" line, labeled as not from daily.dev.

## 5. Match gotchas to call sites

For each gotcha with a concrete symbol, search **only the files from step 3** for that symbol, as a whole word: `rg -nw "<symbol>" <file> <file> ...`. Without `-w`, short names match unrelated words (`certifi` matches every `certificate`). Search the method name, argument or import path that the post names. If the gotcha names another package (a dependency that was dropped, vendored or replaced), run the step 3 script for that package too.

Gotchas about the **environment** (TLS certificates, environment variables, minimum language version, default drivers, system libraries) rarely show up in source files. For those, also check `Dockerfile*`, CI workflows, deploy config, and version pins (`.python-version`, `requires-python`, `engines`, `rust-version`).

Sort each gotcha into one of five groups:

- **Hit**: the symbol appears in importing files. List every file:line.
- **Likely**: the post states a change, and the code depends on that exact thing, but the post does not spell out the consequence (for example, a dependency is now vendored and the code catches that dependency's exception classes). Label it as your inference.
- **No hit**: searched, not found. Name the search term so the user can trust the result.
- **Too vague to check**: the gotcha is behavioral ("timeouts now stricter"). List the most likely call sites (the importing files that call the library most) instead of claiming a match.
- **Not applicable**: needs a feature or platform the repo does not use, with one line on why.

## 6. Report

Use this shape:

```markdown
## Upgrade Blast Radius: <lib> <from> → <to>

**Verdict:** Safe | Needs changes | Risky
<one sentence on why>

**Where <lib> is used:** <N> import sites in <M> files (<a> app, <b> tests). Graph: graphify-out/graph.json, built <date>.

| Gotcha | Change | Hits in this repo | Source |
|---|---|---|---|
| `<symbol>` | <removed / renamed / default changed>, <fix from the post> | <file:line>, <file:line> | [post](url) |

**Likely (inferred):** <gotcha>: <why this code depends on it>, <file:line>.
**Checked, no hits:** <gotcha>: searched `<term>` in <N> files.
**Too vague to pin down:** <gotcha>: most likely in <files>.
**Not covered on daily.dev:** <what you looked for and did not find>.

**Sources:** <titles with links, upvotes and comment counts>
```

Verdict rules:

- **Safe**: no hits, and no behavioral gotchas touch the import sites.
- **Needs changes**: one or more hits, each with a clear fix given in the posts.
- **Risky**: hits or likely hits with no documented fix, behavioral changes in app code, or daily.dev coverage too thin to judge a major version bump or a package swap.

Never invent a gotcha, a version number, a line number or a fix. Every row in the table must trace to a daily.dev link, and every location must come from the graph or `rg`.

## 7. Optional: save the sources

After the report, ask: "Save these posts to a daily.dev bookmark list called `Upgrade: <lib> <to>`?" Only if the user says yes:

1. `GET /bookmarks/lists` and reuse a list with that name if one exists, or `POST /bookmarks/lists` with `{"name": "Upgrade: <lib> <to>"}`.
2. `POST /bookmarks` with `{"postIds": [...], "listId": "<id>"}`.

Confirm what was saved, with the count.
