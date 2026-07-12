# Agentic Support Hiero — Project Rules

This repository **is** a Claude Code plugin: it bundles Hiero- and Hedera-focused skills and the Hedera documentation MCP server, and distributes them through its own marketplace.

## Purpose of this project

The goal is to give any project that works with Hiero or Hedera a ready-made base of domain knowledge and tooling. When working on this repository, keep in mind that every skill you write here will be used across many different projects.

## Repository Structure

The repository root is the plugin root (and the marketplace root).

- `.claude-plugin/plugin.json` — plugin manifest (name, version, metadata)
- `.claude-plugin/marketplace.json` — marketplace catalog (lists this plugin, `source: "./"`)
- `skills/` — reusable Claude Code skills, flat (`skills/<name>/SKILL.md`); discovered automatically
- `.mcp.json` — MCP servers shipped with the plugin (the Hedera documentation server)
- `README.md` — public-facing documentation for users of this plugin
- `CHANGELOG.md` — release history

## Skills

- `hedera-info` — background knowledge about the Hedera public network (HBAR, network services, Council, ecosystem).
- `hiero-info` — background knowledge about the Hiero open-source DLT project under LF Decentralized Trust.
- `hiero-solo` — deploy and manage local/multi-node Hiero/Hedera test networks with the `solo` CLI.
- `hedera-mcp` — how to use the Hedera hosted MCP server (`hedera-testnet` in `.mcp.json`): auth, the `RETURN_BYTES` signing model, and Testnet workflow.
- `hedera-mirror-queries` — how to query Hedera data via the mirror node REST API or the `@hiero-enterprise/mirror` TypeScript package: data map, pagination/timestamp patterns, unit gotchas.

## Distribution

The plugin is installed from inside Claude Code:

```
/plugin marketplace add hiero-hackers/agentic-support-hiero
/plugin install agentic-support-hiero@hiero-hackers
```

Skills are namespaced under the plugin, e.g. `/agentic-support-hiero:hiero-solo`.

**Releasing:** bump `version` in **both** `.claude-plugin/plugin.json` and the `agentic-support-hiero` entry in `.claude-plugin/marketplace.json`, update `CHANGELOG.md`, tag the release, and validate with `claude plugin validate ./ --strict`. Users only receive an update when the version is bumped.

A plugin-root `CLAUDE.md` is not loaded as context, so ship guidance as skills — not in this file.

## Writing Guidelines

### For skills (`skills/<name>/SKILL.md`)

- Each skill must have valid frontmatter with a `name` (kebab-case, matching its directory) and a `description`, plus a clear H1 title.
- Keep skills focused on a single task. Do not combine unrelated actions into one skill.
- Reference bundled files with `${CLAUDE_PLUGIN_ROOT}/…`, never with `../` traversal or absolute paths — the plugin is copied to a cache on install.
- Preserve the `metadata` frontmatter block (`source`, `author`) so provenance tooling keeps working.
- Prefer information from the `hedera-docs` MCP server and official sources over prior knowledge; the network's CLI syntax, ports, and releases change frequently.

## Quality Standards

- All files must be valid Markdown with no formatting errors.
- Use consistent heading levels (H1 for title, H2 for sections, H3 for subsections).
- No trailing whitespace, no tabs for indentation in Markdown files.
- Run `claude plugin validate ./ --strict` after structural changes.

## What NOT to include

- No project-specific configurations (build scripts, CI pipelines, IDE settings).
- No absolute file paths or machine-specific references.
- No secrets, credentials, or environment-specific values.
