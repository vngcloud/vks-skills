---
name: vks
description: "VKS (VNG Kubernetes Service) reference and router. Use for general questions about VKS, getting started, authentication/credentials setup, regions, naming and network rules, or deciding which VKS skill to use. Trigger phrases include: what is VKS, how do I use VNG Kubernetes, help me with VKS, get started with VKS, how do I set up VKS credentials, which VKS skill should I use, VKS overview, explain clusters/node groups. This is a reference guide — to actually create a cluster use /vks-create-cluster, to view status use /vks-explore, to manage node groups use /vks-nodegroup."
---

# VKS Assistant — Reference & Router

## Overview

VKS (VNG Kubernetes Service) is GreenNode/VNG Cloud's managed Kubernetes. These skills let you create and operate clusters by describing what you want in plain language — the assistant discovers available resources, picks safe defaults, and confirms before acting. You do not need to know raw resource IDs.

All operations go through the **VKS MCP server** (`greenode-mcp`) tools.

## Which skill do I need?

| You want to… | Use |
|--------------|-----|
| Create a Kubernetes cluster | `/vks-create-cluster` |
| See what clusters/node groups/nodes/events/versions exist | `/vks-explore` |
| Connect (kubeconfig), update/upgrade, auto-config, or delete a cluster | `/vks-cluster` |
| Add, scale, update, delete a node group, or upgrade its version | `/vks-nodegroup` |
| Deploy an app into a cluster, or inspect pods/logs/events | `/vks-deploy` |
| Understand the platform, set up auth, or learn the rules | this skill |

If a request is vague ("help me with Kubernetes on VNG"), ask one question to route: *create something new*, *check status*, *operate an existing cluster*, or *deploy an app*.

## Core concepts (for newcomers)

- **Cluster** — a managed Kubernetes control plane plus one or more node groups.
- **Node group** — a set of identical worker VMs (same flavor, disk, image). A cluster needs at least one.
- **VPC / subnet** — the private network the nodes live in (from VNG Cloud's vServer).
- **Flavor** — the VM size (vCPU/RAM/GPU). **SSH key / security group** — node access & firewall.

## Prerequisites

### 1. VKS MCP server configured
The `greenode-mcp` server must be available in your Claude client.
- Read-only operations (list/get/discover) work by default.
- **Create / scale / update / delete require the server to run with `--allow-write`.** If a write fails because the server is read-only, tell the user to restart it with `--allow-write`.

### 2. Credentials (`~/.greenode/`)
VKS authenticates with a **VNG Cloud IAM bearer token** (the MCP server obtains it via client-credentials). vServer discovery calls use the **same token** — no `portal-user-id` header is needed.

Set up with `grn configure`, which writes `~/.greenode/credentials` and `~/.greenode/config`. Resolution order (highest first):

1. Env vars: `GRN_CLIENT_ID`, `GRN_CLIENT_SECRET`, `GRN_PROJECT_ID`, `GRN_DEFAULT_REGION`
2. `~/.greenode/credentials` and `~/.greenode/config` (selected by `GRN_PROFILE`, default `default`)

**`project_id` is required for resource discovery** (vpc/subnet/flavor/sshkey/secgroup). `grn configure` auto-detects it; if missing, the discovery tools return a clear error — set `GRN_PROJECT_ID` or re-run `grn configure`.

Verify auth at any time with the `get_access_token` tool.

## Regions

| Region | Notes |
|--------|-------|
| `HCM-3` | default |
| `HAN` | Hanoi |

Override per call via the `region` argument on most tools, or set `GRN_DEFAULT_REGION`.

## Naming & network rules

- Cluster name: 5–20 chars, lowercase letters/digits/hyphens, start & end alphanumeric.
- Node group name: 5–15 chars, same character rules.
- Network type defaults to `CILIUM_NATIVE_ROUTING` (no pod CIDR needed). `CALICO` / `CILIUM_OVERLAY` require a `cidr`.

Full default-picking and discovery logic: `references/resource-defaults.md` (shared by the create and node-group skills).

## Interaction principles (all VKS skills)

- **Guide-first for reads, confirm-once for writes.** Read-only queries need no confirmation. Every write presents one complete plan and waits for explicit approval.
- **Never auto-decide a parameter the user cares about** — auto-pick safe defaults, show them marked as `[auto]`, and let the user change any of them.
- **Resolve names → IDs** via discovery tools; never make the user hunt for raw IDs.
- **Reply in the language the user used** (e.g. answer Vietnamese prompts in Vietnamese).
- **Never put secrets in chat.**

## HARD GATE — confirmation before any write

Before any tool that creates, updates, deletes, scales, or upgrades, tell the user exactly what will happen and wait for an explicit confirmation keyword: `yes`, `confirm`, `ok`, `approve`, `proceed`, `go ahead`, `do it`, `lgtm`, or equivalent. If the user responds with ANYTHING ELSE (a parameter change, a question, a correction, or ambiguous/hedged text like "looks about right I think"), treat it as adjustment input. If they requested a change, update the plan; if no change was requested, re-present the same plan unchanged. Either way, ask again for an explicit keyword. NEVER interpret a non-confirmation response as approval.

## Available VKS skills
- `/vks-create-cluster` — create a cluster end-to-end (discovery → plan → confirm → create → kubeconfig)
- `/vks-explore` — read-only status of clusters, node groups, nodes, events, versions
- `/vks-cluster` — operate an existing cluster: connect (kubeconfig), update/upgrade, auto-upgrade/healing, delete
- `/vks-nodegroup` — create / scale / update / delete node groups, upgrade node-group version
- `/vks-deploy` — deploy apps into a cluster and inspect workloads (manifests, apply YAML, pods, logs, events)
