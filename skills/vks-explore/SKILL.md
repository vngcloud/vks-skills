---
name: vks-explore
description: "Use when the user wants to VIEW or CHECK the status of VKS (VNG Kubernetes Service) resources without changing anything — list clusters, inspect a cluster, see its node groups, list nodes, read events, or list available Kubernetes versions. Trigger phrases: what clusters do I have, list my clusters, show cluster X, is my cluster ready, what's the status of, why is my cluster/node group failing, show events, list node groups, how many nodes, what k8s versions are available, cluster nào đang chạy. Read-only and safe — never modifies anything. To create a cluster use /vks-create-cluster; to change node groups use /vks-nodegroup."
---

# Explore VKS (Read-Only)

## Overview

Answer status questions about VKS in plain language. This skill is **strictly read-only** — it never creates, updates, or deletes anything, so it needs no confirmation and works even when the MCP server runs without `--allow-write`. Other VKS skills call into these same tools to show current state before a change.

## Tools (all read-only)

| Question | Tool |
|----------|------|
| What clusters exist? | `cluster_list` |
| Details of one cluster? | `cluster_get` (cluster id) |
| What happened to a cluster? (debugging) | `cluster_get_events` |
| What node groups in a cluster? | `nodegroup_list` |
| Details of one node group? | `nodegroup_get` |
| What nodes in a node group? | `nodegroup_list_nodes` |
| What Kubernetes versions are available? | `cluster_versions_list` |
| Is auth working / which region & endpoints? | `get_access_token` |

## How to respond

1. **Resolve the target.** If the user names a cluster (not an id), call `cluster_list` and match by name. Exactly one match → use it. **No match → say so and list the available clusters** (offer `/vks-create-cluster` if they meant to create it). Multiple partial matches → list and ask. (Same procedure for node groups via `nodegroup_list`.)
2. **Call the read tool(s) and translate the status.** For "is it ready?" use `cluster_get` and report the status plainly using the status table in `../vks/references/resource-defaults.md` (CREATING / ACTIVE / UPDATING / ERROR / DELETING / …); report any unlisted status verbatim.
3. **Summarize, don't dump.** Lead with the answer ("3 clusters, all ACTIVE"), then the supporting table. The tools already return markdown tables — keep them, but add a one-line takeaway.
4. **Chain for "why is it broken?"** Combine `cluster_get` (status) + `cluster_get_events` (recent events) and explain the likely cause. For a node-group problem, use `nodegroup_get` + `nodegroup_list_nodes`.

## Examples

- *"What clusters do I have?"* → `cluster_list` → "You have 2 clusters: `prod` (ACTIVE), `staging` (ACTIVE)." + table.
- *"Is my-cluster ready?"* → resolve id via `cluster_list` → `cluster_get` → "Not yet — status is CREATING. I'll check again if you'd like."
- *"Why won't staging come up?"* → `cluster_get` + `cluster_get_events` → explain the failing event.
- *"What k8s versions can I use?"* → `cluster_versions_list` → highlight the recommended STABLE one.

## Common mistakes
- Asking for a cluster id when the user gave a name — resolve it via `cluster_list`.
- Returning a raw table with no plain-language answer on top.
- Suggesting or performing a change — that belongs to `/vks-create-cluster` or `/vks-nodegroup`. If the user wants to act, route them there.
