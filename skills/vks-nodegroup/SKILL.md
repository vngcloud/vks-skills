---
name: vks-nodegroup
description: "Use when the user wants to change the worker capacity of an EXISTING VKS (VNG Kubernetes Service) cluster: add a node group, scale a node group up/down, change its size/labels/taints, delete a node group, or upgrade a node group's Kubernetes version. Trigger phrases: add a node group, create node pool, scale my nodes, add more nodes, change node count, resize node group, delete node group, upgrade node group version, thêm node, scale node. Creating a node group uses the same guided flow as cluster creation. To create a whole new cluster use /vks-create-cluster; to just view status use /vks-explore."
---

# Manage VKS Node Groups

## Overview

Add and operate node groups on an existing cluster. **Creating** a node group uses the same Hybrid guided flow as cluster creation (discover → default → plan → confirm). **Scale / update / delete / upgrade** each present a clear plan and require one explicit confirmation.

**REQUIRED BACKGROUND:** For node-group *creation*, read `../vks/references/resource-defaults.md` — same discovery tools, default-picking, flavor-by-need, and name rules as cluster creation.

## Prerequisites

- VKS MCP server running **with `--allow-write`** (all operations here are writes). If an operation fails as read-only, tell the user to restart it with `--allow-write`.
- The target **cluster** and **node group** — resolve names → ids per "Resolving a cluster / node group by name" in `../vks/references/resource-defaults.md` (handle no-match / multiple-match; if the user says a generic word like "workers" and there are several node groups, list them and ask).

## Tools

| Operation | Tool |
|-----------|------|
| Create node group | `nodegroup_create` (after `cluster_versions_list` + discovery) |
| Scale / change size, labels, taints, autoscale | `nodegroup_update` |
| Upgrade k8s version of a node group | `nodegroup_upgrade_version` |
| Preview a deletion | `nodegroup_delete_dryrun` |
| Delete node group | `nodegroup_delete` |
| Inspect current state (before changing) | `nodegroup_list`, `nodegroup_get`, `nodegroup_list_nodes` |

## Create a node group (Hybrid)

1. Resolve the target cluster (by name → id via `cluster_list` if needed). Inherit the cluster's image type and SSH key where sensible.
2. Get the node-group **name** (5–15 chars; propose a fix if invalid).
3. Discover + default per the reference: `flavor_list` (default small, or by `need` if the user wants to choose), `secgroup_list`, `sshkey_list`; disk SSD/100GB; `os` ubuntu; `numNodes` 1 (or 3 if prod/HA).
4. Present ONE plan table marking `[auto]` vs `[you chose]`.
5. **HARD GATE** (below), then `nodegroup_create`.
6. Poll `nodegroup_get` until ACTIVE; report progress.

## Scale / update

1. Show current state first (`nodegroup_get`) so the change is concrete ("currently 2 nodes → 5").
2. Build the partial `nodegroup_update` body with only the changed fields (e.g. `{"numNodes": 5}`) — see "Node-group update body" in the reference for writable fields, the autoscaler caveat, and the scale-down (node drain) warning. Present the before→after.
3. **HARD GATE**, then `nodegroup_update`. Poll `nodegroup_get` to ACTIVE per the reference's waiting policy if the change provisions nodes.

## Upgrade version

1. `cluster_versions_list` to show valid targets; confirm the target version.
2. **HARD GATE**, then `nodegroup_upgrade_version`.

## Delete (extra caution)

1. Always run `nodegroup_delete_dryrun` first and show the user exactly what will be removed (name, id, node count).
2. **HARD GATE with an explicit destructive warning** — state that deletion is irreversible and removes all nodes in the group.
3. Only on explicit confirmation, call `nodegroup_delete`.

## HARD GATE — confirm before any operation

Tell the user exactly what will change and wait for an explicit confirmation keyword: `yes`, `confirm`, `ok`, `approve`, `proceed`, `go ahead`, `do it`, `lgtm`, or equivalent. ANY other response (a parameter change, a question, ambiguous text) is adjustment input — update and re-present. NEVER treat a non-confirmation response as approval. For deletes, require the confirmation after showing the dry-run.

## Common mistakes
- Scaling/deleting without first showing current state — the user can't judge the change.
- Deleting without `nodegroup_delete_dryrun` and a destructive warning.
- Asking for raw flavor/sshkey/secgroup IDs instead of resolving via discovery tools.
- Reporting success at the API call when the node group is still provisioning — poll `nodegroup_get` to ACTIVE.
