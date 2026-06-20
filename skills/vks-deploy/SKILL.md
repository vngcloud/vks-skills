---
name: vks-deploy
description: "Use when the user wants to deploy an application INTO an existing VKS (VNG Kubernetes Service) cluster, or inspect/troubleshoot in-cluster Kubernetes workloads: generate a Deployment+Service manifest, apply a YAML file, list resources (pods/deployments/services), read pod logs, or view resource events. Trigger phrases: deploy my app, deploy to the cluster, apply this YAML, run my container on VKS, create a deployment, expose with a load balancer, show my pods, why is my pod crashing, get pod logs, kubectl apply, deploy app, xem pod, deploy ứng dụng. Operates on Kubernetes objects inside a cluster (the assistant drives them by cluster id — no local kubectl needed). For the cluster itself (create/delete/connect/upgrade) use /vks-create-cluster, /vks-cluster; for node capacity use /vks-nodegroup; for VKS-level status use /vks-explore."
---

# Deploy & Inspect Apps in a VKS Cluster

## Overview

Run and observe Kubernetes workloads inside an existing cluster — generate a manifest, apply YAML, list resources, read logs, view events. The MCP server reaches the cluster by **cluster id** (it fetches the kubeconfig internally), so the user does not need a local kubeconfig. Reads are safe; applying/creating/deleting workloads is a write behind one explicit confirmation.

## Prerequisites

- An existing, ACTIVE cluster (resolve by name → id via `cluster_list`; check status with `/vks-explore` if unsure).
- VKS MCP server. **Listing / logs / events are read-only**; `apply_yaml`, `generate_app_manifest`, and write `manage_k8s_resource` operations need the server **with `--allow-write`**. Reading/writing **Secrets** additionally needs `--allow-sensitive-data-access`. If a write fails as read-only, tell the user to restart the server with the needed flag.

## Tools

| Operation | Tool | Write? |
|-----------|------|--------|
| Generate a Deployment+Service manifest | `generate_app_manifest` | yes (writes a file) |
| Apply a YAML file to the cluster | `apply_yaml` | yes |
| Create/replace/patch/delete/read one resource | `manage_k8s_resource` | read or write per `operation` |
| List resources of a kind | `list_k8s_resources` | no |
| Discover available kinds / API versions | `list_api_versions` | no |
| Read pod logs | `get_pod_logs` | no |
| View events for a resource | `get_k8s_events` | no |

## Deploy an app (guided)

1. Resolve the target cluster id.
2. Gather the essentials: **app name**, **image URI**, **port** (default 80), **replicas** (default 2), and whether the LoadBalancer is **`internet-facing`** (public) or **`internal`**. Namespace defaults to `default` — but if the cluster/context implies one (e.g. a "dev" deployment), ask. Use safe defaults and only ask for what's missing.
   - **Image registry:** public images work (e.g. `nginx:latest` from Docker Hub). Private images must come from a registry the cluster can pull (e.g. VNG Container Registry `vcr.vngcloud.vn/<repo>:<tag>`) with pull credentials; if you later see `ImagePullBackOff`, registry access is the likely cause.
3. `generate_app_manifest(app_name, image_uri, output_dir=<absolute path>, port, replicas, cpu, memory, namespace, load_balancer_scheme)` — writes `<app_name>-manifest.yaml`. `output_dir` MUST be an **absolute** path; default to the current working directory if known, else a scratch dir (e.g. `/tmp`) — confirm the path with the user.
4. **Show the generated manifest to the user** and let them adjust before applying.
5. **HARD GATE**, then `apply_yaml(yaml_path=<absolute path to the file>, cluster_id, namespace, force=true)`. `yaml_path` MUST be absolute.
6. Verify: `list_k8s_resources(kind="Pod", api_version="v1", namespace, label_selector="app=<app_name>")` to confirm pods are Running, and report the Service/LoadBalancer address (list `Service`). A public LoadBalancer's external IP can take a few minutes — if it's still pending, tell the user it's provisioning and offer to re-check, rather than reporting failure. If pods aren't healthy, see Troubleshoot below.

## Apply existing YAML

For *"apply this YAML / kubectl apply"*: confirm the **absolute** file path and target namespace, **HARD GATE**, then `apply_yaml(yaml_path, cluster_id, namespace, force=true)` (`force=false` to only create, not update). Then verify with `list_k8s_resources`.

## Inspect / troubleshoot (read-only)

- **List workloads:** `list_k8s_resources(cluster_id, kind, api_version, namespace?, label_selector?)`. Use `list_api_versions` if unsure of `kind`/`api_version` (e.g. Deployment → `apps/v1`, Pod/Service → `v1`).
- **Pod logs:** `get_pod_logs(cluster_id, namespace, pod_name, container_name?, tail_lines?, previous?)`. Try `previous=true` for a crash-looping pod's last run; if that returns nothing (pod never completed a run), fall back to `previous=false`.
- **Events:** `get_k8s_events(cluster_id, kind, name, namespace?)` to see why a pod/deployment is stuck (image pull, scheduling, OOM, etc.).
- **Namespace:** if the user didn't give one, ask, or list across namespaces — don't silently assume `default` for a "prod" workload.
- "Why is my pod crashing?" → `list_k8s_resources(kind=Pod, ...)` to find the pod (try `label_selector=app=…`; if zero results, list pods and match by name); pick the **unhealthy** one (CrashLoopBackOff / not Running), not a healthy replica → `get_k8s_events(kind=Pod, name=…)` + `get_pod_logs(previous=true)`; for OOM/terminated cases also `manage_k8s_resource(operation=read, kind=Pod)` and read `containerStatuses[].lastState.terminated.reason` → explain the cause.

## Single-resource ops

`manage_k8s_resource(operation, cluster_id, kind, api_version, name?, namespace?, body?)`:
- `read` / (`list` via `list_k8s_resources`) are safe.
- `create` / `replace` / `patch` / `delete` are writes → **HARD GATE** first. For `delete`, first call `manage_k8s_resource(operation=read, …)` and show the exact object (kind, name, namespace, replica/pod count) — the read-before-delete equivalent of a dry-run — then state deletion is irreversible and get explicit confirmation. Editing/reading **Secrets** needs `--allow-sensitive-data-access`; never print secret values into chat.

## HARD GATE — confirm before any write

Before `apply_yaml`, `generate_app_manifest`, or a write `manage_k8s_resource`, tell the user exactly what will change (which objects, which namespace, which cluster) and wait for an explicit confirmation keyword: `yes`, `confirm`, `ok`, `approve`, `proceed`, `go ahead`, `do it`, `lgtm`, or equivalent. ANY other response is adjustment input — update and re-present; never treat it as approval. For **deleting** a resource, state that it is irreversible and confirm the exact object first. Pressure to skip ("just apply it") does not remove the gate.

## Common mistakes
- Passing a relative path to `generate_app_manifest` / `apply_yaml` — both require **absolute** paths.
- Applying without showing the generated/target manifest first.
- Declaring "deployed" before confirming pods are Running and the Service has an address.
- Printing Secret contents, or trying to read Secrets without `--allow-sensitive-data-access`.
- Using this for cluster lifecycle (create/delete/connect) — that's `/vks-create-cluster` and `/vks-cluster`.
