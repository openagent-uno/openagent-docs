# Memory & Vault

OpenAgent's long-term memory is a plain **Obsidian-compatible markdown vault** — one `.md` file per note, with YAML frontmatter, `[[wikilinks]]`, and tags. The agent reads and writes those files directly via the `vault` MCP. Open the same folder in Obsidian and graph view, backlinks, and plugins all work, untouched.

The markdown is the source of truth. On top of it sits a code-enforced **quality system** (shipped in v0.14.12): an incremental SQLite + FTS5 index, a quality gate that grades every note, a mechanical doctor, derived artifacts (`llms.txt`, `_showcase/`), link-rewriting move/rename, and a git-backed history. The index is a cache — delete it and it rebuilds from the markdown.

::: tip
For the gate rules, the index, the doctor, git provenance, the `vault-gate` MCP tools, and the `openagent vault` CLI, see **[Vault Quality System](./vault-quality.md)**.
:::

## Configuration

```yaml
memory:
  db_path: "~/.openagent/openagent.db"   # SQLite — operational state (see Architecture §9)
  vault_path: "~/.openagent/memories"    # Obsidian markdown vault (git-backed)
```

Quality-system options live under `memory.vault.*` and `memory.vault_reminder.*` — see [Vault Quality System → Configuration](./vault-quality.md#configuration).

## How It Works

- The agent writes notes as `.md` files inside `vault_path/`
- The `vault` MCP exposes `list_directory`, `read_note`, `write_note`, `patch_note`, `search_notes`, `manage_tags`, etc.; the `vault-gate` MCP adds `vault_gate`, `vault_doctor`, `vault_rename_note`, `vault_backlinks`, and friends
- The system auto-commits every change to git with provenance trailers, and a full-text index keeps search fast as the vault grows
- Open the same folder in Obsidian desktop for graph view, backlinks, and plugin support

## Note Format

```markdown
---
title: Deploy Wardrobe Service
type: reference
tags: [k8s, wardrobe, ovh]
created: 2026-04-07
---

# Deploy Wardrobe Service

rsync + docker build + k3s import...
See also: [[server-architecture]]
```

`[[wikilinks]]`, tags and YAML frontmatter are standard Obsidian conventions.

## System Prompt

OpenAgent injects a framework-level system prompt that covers:

- How to use the vault (`vault` / `vault-gate` tools, wikilinks, tags)
- Prefer MCP tools over shell scripts
- Act autonomously (tool calls are pre-approved)
- Be concise

Your `system_prompt` in `openagent.yaml` should stay short — just identity and a pointer to the vault. Everything else belongs as `.md` notes.
