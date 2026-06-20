# VKS Skills — VKS Assistant

A set of Claude skills that turn natural language into VKS (VNG Kubernetes Service) operations, so anyone — even without deep Kubernetes or VNG Cloud experience — can create and manage clusters by describing what they want.

The skills orchestrate the [VKS MCP server](https://github.com/vngcloud/greenode-mcp) (`greenode-mcp`) tools. They play the role of `eksctl`/`gcloud`: discover available resources, pick sensible defaults, present a single clear plan, and act only after explicit confirmation.

## Skills

| Skill | Use it for |
|-------|------------|
| **`vks`** | Reference & router — platform overview, auth setup, naming/network rules, "which skill do I need?" |
| **`vks-create-cluster`** | Create a cluster end-to-end from a natural-language request, ending with kubeconfig |
| **`vks-explore`** | Read-only status: list/inspect clusters, node groups, nodes, events, k8s versions |
| **`vks-nodegroup`** | Create / scale / update / delete node groups, upgrade node-group version |

## Requirements

- The **VKS MCP server** (`greenode-mcp`) configured in your AI client (Claude Code / Cursor / Codex).
  - For create/scale/delete operations, run it with `--allow-write` (read-only otherwise).
- VNG Cloud credentials (see below).

## Credentials

The MCP server authenticates to VKS with a VNG Cloud IAM token derived from a **client_id / client_secret** (an IAM service account), plus a **project_id** (required for resource discovery — vpc/subnet/flavor/sshkey/secgroup). Two ways to provide them:

### Option A — `grn` CLI (recommended)

Install the [GreenNode CLI](https://github.com/vngcloud/greenode-cli) (`grn`) and run:

```bash
grn configure
```

It prompts for client_id, client_secret, default region, and project_id (auto-detected if left blank), then writes them to `~/.greenode/credentials` and `~/.greenode/config`. The MCP server reads `~/.greenode/` automatically — **you don't need any env vars in the MCP config**. Use `grn configure --profile <name>` for multiple environments, and select one with `GRN_PROFILE=<name>`.

### Option B — environment variables

Useful for CI / containers, or to override the files. These take priority over `~/.greenode/`:

| Var | Purpose |
|-----|---------|
| `GRN_CLIENT_ID` | IAM service-account client id |
| `GRN_CLIENT_SECRET` | IAM service-account client secret |
| `GRN_PROJECT_ID` | project id (required for discovery tools) |
| `GRN_DEFAULT_REGION` | region (e.g. `HCM-3`, `HAN`) |
| `GRN_PROFILE` | which `~/.greenode` profile to use |

Get the client_id / client_secret from a VNG Cloud IAM service account; verify auth anytime with the `get_access_token` tool.

## Install

This repo is a Claude Code plugin (manifest in `.claude-plugin/`). Three ways to use it:

**1. As a plugin via the marketplace (recommended)**
```
/plugin marketplace add vngcloud/vks-skills
/plugin install vks-skills@greennode
/reload-plugins
```
Note: `vngcloud/vks-skills` is the GitHub repo path; `@greennode` is the marketplace name (from `marketplace.json`). Skills are then available namespaced, e.g. `/vks-skills:vks-create-cluster`.

**2. Local plugin directory (for trying it out)**
```
claude --plugin-dir /path/to/vks-skills
```

**3. Manual copy**
Copy the `skills/` subfolders into your Claude skills directory (e.g. `~/.claude/skills/`). Keep the directory layout intact — `vks-create-cluster` and `vks-nodegroup` reference `skills/vks/references/resource-defaults.md` by relative path.

Each skill is a standard `SKILL.md` with `name` + `description` frontmatter; skills are auto-discovered from `skills/`.

### Team / org auto-install

To push these skills to a whole team, copy [`examples/team-settings.json`](examples/team-settings.json) into your **project's** `.claude/settings.json` (or your org's managed settings). When a teammate trusts the repo, Claude Code prompts them to add the marketplace and install the plugin:

```json
{
  "extraKnownMarketplaces": {
    "greennode": { "source": { "source": "github", "repo": "vngcloud/vks-skills" }, "autoUpdate": true }
  },
  "enabledPlugins": { "vks-skills@greennode": true }
}
```

The `"greennode"` key must match the `name` field in this repo's `.claude-plugin/marketplace.json` (the marketplace name); the repo path stays `vngcloud/vks-skills`. For a **private** marketplace repo, background auto-update needs `GITHUB_TOKEN`/`GH_TOKEN` in the environment.

## Use with Cursor / Codex (and other SKILL.md tools)

`SKILL.md` is a cross-agent standard — these skills run unmodified in Cursor, Codex, Gemini CLI, etc. Two things differ from Claude Code:

**1. Installation = copy `skills/` (the plugin marketplace is Claude-Code-only).** Keep the directory layout intact — `vks-create-cluster` and `vks-nodegroup` reference `vks/references/resource-defaults.md` by relative path, so the four skill folders must stay siblings.

| Tool | Put the `skills/` folders in |
|------|------------------------------|
| Cursor | `.cursor/skills/` (project) — then reload the workspace |
| Codex | `~/.agents/skills/` (or Codex's skills dir) |

**2. Configure the `greenode-mcp` MCP server in that tool** — the skills call its tools, so without it they have nothing to drive. Use `--allow-write` to enable create/scale/delete. If you ran `grn configure` (Credentials → Option A), the server reads `~/.greenode/` automatically and the configs below need **no `env` block**. Adjust `command`/`args` to however you run `greenode-mcp` (e.g. `uv run --directory <repo>/src/vks-mcp-server vks-mcp-server`).

**Cursor** — `.cursor/mcp.json`:
```json
{
  "mcpServers": {
    "greenode-mcp": {
      "command": "vks-mcp-server",
      "args": ["--allow-write"]
    }
  }
}
```

**Codex** — `~/.codex/config.toml` (or `codex mcp add greenode-mcp -- vks-mcp-server --allow-write`):
```toml
[mcp_servers.greenode-mcp]
command = "vks-mcp-server"
args = ["--allow-write"]
```

If you prefer env vars (Credentials → Option B) instead of `grn configure`, add them — Cursor: an `"env": { "GRN_CLIENT_ID": "…", "GRN_CLIENT_SECRET": "…", "GRN_PROJECT_ID": "…" }` block in the server entry; Codex: a `[mcp_servers.greenode-mcp.env]` table with the same keys.

In Cursor invoke a skill from the slash-command menu; in Codex it loads skills automatically (run `/mcp` to confirm the server is connected).

## Design

These skills implement the design at `docs/superpowers/specs/2026-06-20-vks-assistant-skills-design.md` (in the working repo). The shared default-picking logic lives in `skills/vks/references/resource-defaults.md` and is consumed by both `vks-create-cluster` and `vks-nodegroup`.

## License

Apache-2.0
