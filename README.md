# kb-skill

A Claude Code plugin that drives the [obsidian-kb MCP server](https://github.com/huangsron/obsidianMcp) — a personal knowledge base backed by an Obsidian vault, shared across multiple apps/projects via 9 MCP tools.

## What it does

When you ask a question or want to record something, this skill walks Claude through the right MCP tools in the right order:

1. **`kb_lookup`** — try the cache trigger index first; if hit, quote the answer verbatim.
2. **`kb_search`** — full-text search, scoped to the current project via tags derived from `cwd`.
3. **`kb_read`** — read full notes when search hits.
4. **`kb_create`** — auto-classify and persist result-shaped answers (bug fixes, configs, commands, API endpoints). Discussion/exploration notes ask first.
5. **`kb_log_append`** — keep a chronological trail.

Plus `kb_update_frontmatter`, `kb_list_by_tag`, `kb_reindex_cache`, `kb_index_rebuild`.

## Prerequisites

You need the **obsidian-kb MCP server** running and reachable. See [huangsron/obsidianMcp](https://github.com/huangsron/obsidianMcp). Quick version:

```bash
git clone https://github.com/huangsron/obsidianMcp
cd obsidianMcp
cp .env.example .env  # fill in MCP_AUTH_TOKEN (openssl rand -hex 32)
docker compose up -d
```

The plugin assumes the server is at `http://127.0.0.1:8787/mcp`.

## Install

```bash
# 1. Add this repo as a Claude Code plugin marketplace
/plugin marketplace add huangsron/kb-skill

# 2. Install the plugin
/plugin install kb-skill@kb-skill
```

After install, **you must create `.mcp.json` with your token**, because tokens are not stored in the repo. Find the installed plugin directory (usually `~/.claude/plugins/cache/<marketplace>/<plugin>/`), then:

```bash
cp .mcp.json.example .mcp.json
# Edit .mcp.json — replace <YOUR_TOKEN_HERE> with the value of MCP_AUTH_TOKEN from your obsidian-kb server's .env
```

Restart Claude Code. Verify with:

```
/plugin
```

You should see `kb-skill` listed and the `obsidian-kb` MCP server connected.

## Usage

Trigger the skill in two ways:

- **Slash command**: `/kb` followed by what you want to do
- **Direct phrases**: "查知識庫…", "記到知識庫…", "kb lookup…", "kb search…"

The skill stays out of the way on generic phrases like "查一下" — those are intentionally NOT triggers.

## Cross-AP context

The KB is shared across projects. The skill auto-derives an AP tag from `basename(cwd)` and uses it as a filter in `kb_search`:

- `cwd = D:\work\local\sron\obsidianMcp` → search with `tags: ["obsidianMcp"]`
- `cwd = D:\work\local\sron\tes` → search with `tags: ["tes"]`

If a search returns nothing, it widens automatically (drops the tag filter).

## What gets auto-saved vs asked

| Kind | Behavior |
|------|----------|
| Bug fixes, configs, commands, API endpoints | Auto `kb_create` |
| Discussions, trade-offs, exploration | Ask first |
| Chit-chat / topic already in vault | Don't save |

## Updating

```bash
/plugin marketplace update kb-skill
/plugin update kb-skill@kb-skill
```

## Files

```
kb-skill/
├── .claude-plugin/
│   ├── plugin.json           # plugin manifest
│   └── marketplace.json      # marketplace index (single-plugin)
├── .mcp.json.example         # copy to .mcp.json after install, fill token
├── skills/kb/SKILL.md        # the actual skill instructions
└── README.md
```

`.mcp.json` is gitignored; tokens never enter version control.

## License

MIT
