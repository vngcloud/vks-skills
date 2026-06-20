---
name: vks-cluster
description: "Use when the user wants to operate an EXISTING VKS (VNG Kubernetes Service) cluster (not its node groups, not creating a new one): get/connect kubeconfig (kubectl access), update cluster settings (whitelist CIDR, plugins), upgrade the cluster's Kubernetes version, configure auto-upgrade or auto-healing, or DELETE a cluster. Trigger phrases: connect to my cluster, get kubeconfig, how do I kubectl into X, update cluster, change cluster settings, whitelist an IP, upgrade cluster version, set auto-upgrade, enable auto-healing, delete cluster, xóa cluster, kết nối cluster, nâng cấp cluster. To create a cluster use /vks-create-cluster; for node groups use /vks-nodegroup; for read-only status use /vks-explore; to deploy apps inside the cluster use /vks-deploy."
---

# Operate an Existing VKS Cluster

## Overview

Lifecycle and connection operations on a cluster that already exists: **connect** (kubeconfig), **update** (whitelist/plugins), **upgrade** (k8s version), **auto-upgrade / auto-healing** config, and **delete**. Reads are safe; every write goes through one explicit confirmation, and delete gets an extra destructive gate.

**REQUIRED BACKGROUND:** Resolving a cluster by name, status meanings, and the waiting/polling policy live in `../vks/references/resource-defaults.md`. The shared HARD GATE wording is in `../vks/SKILL.md`.

## Prerequisites

- VKS MCP server. **Connect (kubeconfig) is read-only**; update / upgrade / auto-config / delete are writes and need the server running **with `--allow-write`** (if a write fails as read-only, tell the user to restart it with `--allow-write`).
- Resolve the target cluster by name → id via `cluster_list` (handle no-match / multiple-match per the reference).

## Tools

| Operation | Tool | Write? |
|-----------|------|--------|
| Get kubeconfig (connect) | `cluster_get_kubeconfig` | no |
| Update settings (whitelist, plugins, version) | `cluster_update` | yes |
| Preview a deletion | `cluster_delete_dryrun` | no |
| Delete cluster | `cluster_delete` | yes |
| Configure auto-upgrade | `cluster_auto_upgrade_config` | yes |
| Remove auto-upgrade | `cluster_auto_upgrade_delete` | yes |
| Configure auto-healing | `cluster_auto_healing_config` | yes |
| See current settings / status | `cluster_get`, `cluster_versions_list` | no |

## Connect (kubeconfig)

For *"how do I connect / get kubeconfig / kubectl into my cluster"*:
1. Resolve the cluster id.
2. Call `cluster_get_kubeconfig`.
3. Return the kubeconfig and tell the user how to use it: save it (e.g. `~/.kube/vks-<name>.yaml`) and either merge into `~/.kube/config` or run `KUBECONFIG=~/.kube/vks-<name>.yaml kubectl get nodes`. This is read-only — no confirmation needed.

> To run kubectl-style operations *through the assistant* instead of locally, use `/vks-deploy` (it drives the in-cluster tools by cluster id; the user doesn't need a local kubeconfig).

## Update / upgrade

`cluster_update` takes a **partial** body — only the fields being changed. Always `cluster_get` first to show current values.

- **Whitelist CIDR / API access**, **plugins** (load-balancer, block-store CSI): build a minimal body with just those fields.
- **Upgrade Kubernetes version**: `cluster_get` for the current version, then `cluster_versions_list` (recommended STABLE marked). **You cannot skip minor versions** (e.g. 1.27 → 1.29 is invalid) — if the target is more than one minor ahead, refuse and upgrade one minor at a time. If the user says "latest", pick the recommended STABLE. Confirm the target, then `cluster_update` with the new `version`. Warn that a control-plane upgrade is disruptive and should precede node-group upgrades (`/vks-nodegroup`), and that node groups should then be upgraded to match.

Present current → proposed, apply the **HARD GATE**, then `cluster_update`. Poll `cluster_get` to ACTIVE per the reference if the change reconciles.

## Auto-upgrade / auto-healing

- Auto-upgrade: `cluster_auto_upgrade_config(cluster_id, weekdays, time)` — `weekdays` like `"Mon,Wed,Fri"`, `time` like `"03:00"`. Remove with `cluster_auto_upgrade_delete`.
- Auto-healing: `cluster_auto_healing_config(cluster_id, enable_auto_healing, max_unhealthy?, unhealthy_range?, timeout_unhealthy?)` — `max_unhealthy` like `"2"` or `"40%"`; `timeout_unhealthy` 5–180 minutes.

Confirm the schedule/thresholds with the user, apply the **HARD GATE**, then call the tool.

## Delete (destructive — extra gate)

1. **Always run `cluster_delete_dryrun` first** and **summarize** what will be destroyed (cluster name, id, status, affected node groups) — don't dump raw output. If the dry-run itself errors (cluster not found, or already in `DELETING`/`ERROR` state), **abort** and report instead of proceeding.
2. **DESTRUCTIVE HARD GATE:** state clearly that deleting the cluster is **irreversible** and removes **all node groups, nodes, and in-cluster workloads**. Require, AFTER the dry-run, an explicit confirmation keyword — the full HARD GATE list below (`yes` / `confirm` / `ok` / `approve` / `proceed` / `go ahead` / `do it` / `lgtm`), or `delete`. A vague or hurried reply ("just do it", "skip the checks") that does not clearly approve deleting THIS named cluster is NOT confirmation — re-state the impact and ask again.
3. Only on explicit confirmation, call `cluster_delete`. Then poll `cluster_get` until it reports deleted/not-found.

## HARD GATE — confirm before any write

Tell the user exactly what will change and wait for an explicit confirmation keyword: `yes`, `confirm`, `ok`, `approve`, `proceed`, `go ahead`, `do it`, `lgtm`, or equivalent. ANY other response (a parameter change, a question, ambiguous text) is adjustment input — update and re-present. NEVER treat a non-confirmation response as approval. Pressure to skip the gate ("hurry", "trust me", "just delete it") does not remove it — for deletes especially, the dry-run + explicit approval are mandatory.

## Common mistakes
- Treating an upgrade/whitelist change as harmless — show current → proposed and confirm.
- Deleting a cluster without `cluster_delete_dryrun` and the destructive gate.
- Sending a full cluster object to `cluster_update` instead of only the changed fields.
- Reporting success at the API call when the change is still reconciling — poll `cluster_get`.
- Using this skill to add/scale nodes (that's `/vks-nodegroup`) or to deploy workloads (that's `/vks-deploy`).
