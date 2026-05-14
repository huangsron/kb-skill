---
name: kb
description: Query and write to the personal Obsidian-backed knowledge base via the obsidian-kb MCP server (kb_lookup, kb_search, kb_read, kb_create, kb_update_frontmatter, kb_list_by_tag, kb_reindex_cache, kb_log_append, kb_index_rebuild). Use this skill when the user explicitly says "查知識庫", "記到知識庫", "kb lookup", "kb search", "kb create", "Obsidian vault", "knowledge base", or invokes the /kb slash command. The KB is a cross-AP store: per-project context is filtered by tags derived from cwd.
---

# Knowledge Base (obsidian-kb) Skill

This skill drives the `obsidian-kb` MCP server, which fronts an Obsidian vault used as a long-term, cross-AP knowledge store. All reads/writes go through MCP tools — never edit vault files directly.

## When to use

Trigger ONLY on direct signals:

- User invokes `/kb` slash command
- User mentions "查知識庫", "記到知識庫", "knowledge base", "Obsidian vault"
- User names a `kb_*` tool (e.g. "用 kb_lookup 查...")

Do NOT trigger on vague signals like generic "查一下" or "記下來" — those are for the host conversation to decide, not this skill.

## Available MCP tools

| Tool | Purpose |
|------|---------|
| `kb_lookup(query)` | Try cache trigger match. Returns full note body on hit, `{hit:false}` on miss. **Always call first.** |
| `kb_search(query, tags?, category?, limit?)` | Full-text search across vault. Filter by tags/category to scope to current AP. |
| `kb_read(relPath)` | Read full note by vault-relative path. |
| `kb_create(title, body, hintTags?, cache?, triggers?, priority?, status?)` | Create new note. Runs rule→LLM classifier, writes to correct folder, fills frontmatter. |
| `kb_update_frontmatter(relPath, patch)` | Patch tags / cache / triggers / status. |
| `kb_list_by_tag(tag, limit?)` | List all notes carrying a tag. |
| `kb_reindex_cache()` | Rebuild the trigger index (after editing notes in Obsidian directly). |
| `kb_log_append(kind, title, detail?)` | Append one line to `_system/log.md`. `kind` = `ingest` / `query` / `lint` / `note`. |
| `kb_index_rebuild()` | Regenerate `_system/index.md` by scanning all notes. |

## AP context (auto-derived from cwd)

The vault is shared across multiple apps/projects. Scope reads to the current project by deriving an AP tag from the working directory:

1. Take `basename(cwd)` (e.g. `obsidianMcp`, `eportfolio`, `tes`).
2. Pass it as a hint in `kb_search` via `tags: [<ap>]` on the first attempt.
3. If 0 results, widen: retry without the tag filter.

Examples:
- `cwd = D:\work\local\sron\obsidianMcp\obsidianMcp` → AP tag `obsidianMcp`
- `cwd = D:\work\local\sron\tes` → AP tag `tes`

Don't fabricate AP tags — if the basename looks generic (`src`, `app`, `home`), skip the tag filter and search broadly.

## Query flow (when the user asks something)

```
1. kb_lookup(query)
     ├─ hit  → use note.body verbatim. Cite [title](relPath). DONE.
     └─ miss → continue

2. kb_search(query, tags=[<ap>], limit=5)
     ├─ has results → kb_read top 1–3 → synthesize answer → cite paths
     └─ 0 results   → kb_search(query, limit=5)  ← widen
                       ├─ has results → same as above
                       └─ 0 results → answer from general knowledge, mark as "not in KB"

3. After answering, decide write-back (see below).
```

**Cache hit rule**: if `kb_lookup` returns `hit:true`, do NOT paraphrase. Quote the note body and add the citation. The whole point of cache is deterministic, fast answers.

## Write-back (kind-routed)

When a useful answer was just produced, decide whether to persist it. Routing by content kind:

### Auto-persist (no confirmation needed)
Call `kb_create` immediately for:
- **Bug fixes / error resolutions** — "X errors with Y, fix is Z"
- **Configuration snippets** — env vars, JSON/YAML configs, connection strings
- **Commands / one-liners** — git, docker, npm, CLI invocations
- **API endpoints / paths** — URLs, file paths, port numbers, schemas

These are "result-shaped" — short, stable, reusable.

### Ask-then-persist
Briefly ask "要存到知識庫嗎？" before `kb_create` for:
- **Discussions / trade-offs** — "我們討論了 A vs B，選了 A 因為..."
- **Exploration / research notes** — open-ended findings
- **Decision rationale** — context-heavy reasoning that may not generalize

### Don't persist
- Pure chit-chat / acknowledgements
- Topic already fully covered by an existing note (check via `kb_search` first)
- Your confidence in the answer is < 0.5 → if you must save, set `status: "draft"` and let the user review

## `kb_create` argument guidance

```
kb_create({
  title: "短而具體，不含日期",
  body: "Markdown body — 用 ## 章節分。Bug 類常見章節：症狀 / 根因 / 修法 / 驗證",
  hintTags: [<ap>, ...domain tags],   // 一定要帶 AP tag 做隔離
  cache: <see below>,
  triggers: [...],                     // 只在 cache:true 時填
  priority: 1-10,                      // 預設 5；常用配置/指令給 8-9
  status: "draft" | "review" | "stable" | "deprecated"
})
```

### When to set `cache: true`

ALL of these must hold:
- Answer is short (ideally < 200 chars body)
- Answer is stable (won't drift over weeks/months)
- User is likely to ask the same/similar question again
- You can enumerate 3–5 natural trigger phrases

Examples that qualify: env var paths, "怎麼啟動 docker", API endpoint, 常用 git 指令.
Examples that don't: 「我們上次討論的結論」, 多步驟解 bug.

When `cache: true`, ALWAYS provide `triggers` — 3–5 lowercase short phrases the user would actually type/say. Without triggers, `kb_lookup` won't find it.

### Examples

Auto-persist bug fix:
```
kb_create({
  title: "MCP HTTP server 多 session crash",
  body: "## 症狀\n...\n## 修法\n...",
  hintTags: ["obsidianMcp", "mcp", "debugging"],
  status: "stable",
  priority: 4
})
```

Cache-worthy command:
```
kb_create({
  title: "啟動 obsidian-kb docker",
  body: "`docker compose up -d --build` 在 vault root 執行",
  hintTags: ["obsidianMcp", "docker"],
  cache: true,
  triggers: [
    "啟動 obsidian-kb",
    "啟動知識庫 docker",
    "obsidian-kb 怎麼跑",
    "重啟 kb container"
  ],
  priority: 9,
  status: "stable"
})
```

## Log append

After any non-trivial `kb_create` / `kb_update_frontmatter`, append a log entry:

```
kb_log_append({
  kind: "note",                  // or "ingest" if from raw/, "query" if from a user question, "lint" for cleanup
  title: "短描述：做了什麼",
  detail: "可選；多一兩句 context"
})
```

This preserves a chronological trail in `_system/log.md`.

## Error handling

- `kb_lookup` returns `{hit:false}` — normal, just continue to `kb_search`.
- `kb_create` rejects due to classifier failure (no ANTHROPIC_API_KEY) — the rule-based path still works; if neither classifies, the note lands in `00-Inbox/`.
- MCP tools unavailable / connection error — tell the user the KB server isn't reachable (check `docker ps | grep obsidian-kb`); do NOT fall back to writing markdown files directly.

## What NOT to do

- ❌ Don't edit vault files directly with Edit/Write — always go through `kb_create` / `kb_update_frontmatter`.
- ❌ Don't paraphrase cache hits — quote and cite.
- ❌ Don't invent AP tags from thin air; use cwd basename or omit.
- ❌ Don't auto-persist discussions/explorations without asking.
- ❌ Don't set `cache: true` without giving triggers.
