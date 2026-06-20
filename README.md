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
/plugin marketplace add vks-team/vks-skills
/plugin install vks-skills@vks-team
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

## Design

These skills implement the design at `docs/superpowers/specs/2026-06-20-vks-assistant-skills-design.md` (in the working repo). The shared default-picking logic lives in `skills/vks/references/resource-defaults.md` and is consumed by both `vks-create-cluster` and `vks-nodegroup`.

## License

Apache-2.0
