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

- The **VKS MCP server** (`greenode-mcp`) configured in your Claude client.
  - For create/scale/delete operations, run it with `--allow-write`.
- VNG Cloud credentials in `~/.greenode/` (run `grn configure`) including `project_id`, or env vars `GRN_CLIENT_ID` / `GRN_CLIENT_SECRET` / `GRN_PROJECT_ID`.

## Install

This repo is a Claude Code plugin (manifest in `.claude-plugin/`). Three ways to use it:

**1. As a plugin via the marketplace (recommended)**
```
/plugin marketplace add vngcloud/vks-skills
/plugin install vks-skills@vngcloud
/reload-plugins
```
Skills are then available namespaced, e.g. `/vks-skills:vks-create-cluster`.

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
    "vngcloud": { "source": { "source": "github", "repo": "vngcloud/vks-skills" }, "autoUpdate": true }
  },
  "enabledPlugins": { "vks-skills@vngcloud": true }
}
```

The `"vngcloud"` key must match the `name` field in this repo's `.claude-plugin/marketplace.json`. For a **private** marketplace repo, background auto-update needs `GITHUB_TOKEN`/`GH_TOKEN` in the environment.

## Use with Cursor / Codex (and other SKILL.md tools)

`SKILL.md` is a cross-agent standard — these skills run unmodified in Cursor, Codex, Gemini CLI, etc. Two things differ from Claude Code:

**1. Installation = copy `skills/` (the plugin marketplace is Claude-Code-only).** Keep the directory layout intact — `vks-create-cluster` and `vks-nodegroup` reference `vks/references/resource-defaults.md` by relative path, so the four skill folders must stay siblings.

| Tool | Put the `skills/` folders in |
|------|------------------------------|
| Cursor | `.cursor/skills/` (project) — then reload the workspace |
| Codex | `~/.agents/skills/` (or Codex's skills dir) |

**2. Configure the `greenode-mcp` MCP server in that tool** — the skills call its tools, so without it they have nothing to drive. Use `--allow-write` to enable create/scale/delete. Credentials come from `~/.greenode/` (run `grn configure`); the `env` block below is only needed if you don't use `~/.greenode`. Adjust `command`/`args` to however you run `greenode-mcp` (e.g. `uv run --directory <repo>/src/vks-mcp-server vks-mcp-server`).

**Cursor** — `.cursor/mcp.json`:
```json
{
  "mcpServers": {
    "greenode-mcp": {
      "command": "vks-mcp-server",
      "args": ["--allow-write"],
      "env": {
        "GRN_CLIENT_ID": "<client-id>",
        "GRN_CLIENT_SECRET": "<client-secret>",
        "GRN_PROJECT_ID": "<project-id>"
      }
    }
  }
}
```

**Codex** — `~/.codex/config.toml` (or `codex mcp add greenode-mcp -- vks-mcp-server --allow-write`):
```toml
[mcp_servers.greenode-mcp]
command = "vks-mcp-server"
args = ["--allow-write"]

[mcp_servers.greenode-mcp.env]
GRN_CLIENT_ID = "<client-id>"
GRN_CLIENT_SECRET = "<client-secret>"
GRN_PROJECT_ID = "<project-id>"
```

In Cursor invoke a skill from the slash-command menu; in Codex it loads skills automatically (run `/mcp` to confirm the server is connected).

## Design

These skills implement the design at `docs/superpowers/specs/2026-06-20-vks-assistant-skills-design.md` (in the working repo). The shared default-picking logic lives in `skills/vks/references/resource-defaults.md` and is consumed by both `vks-create-cluster` and `vks-nodegroup`.

## License

Apache-2.0
