# VKS Resource Defaults & Discovery

Shared logic for `vks-create-cluster` and `vks-nodegroup`. This is how the assistant behaves like `eksctl`/`gcloud`: discover what exists, auto-pick safe defaults, and ask the user only the few decisions that matter.

## Discovery tools (read-only, always available)

Resolve resource IDs by calling these MCP tools — never ask the user to paste raw IDs (accept them if offered, but don't require them):

| Tool | Returns | Item ID field → create-body field |
|------|---------|-----------------------------------|
| `vpc_list` | VPCs/networks | `id` → `vpcId` |
| `subnet_list` (needs `vpc_id`) | subnets of a VPC | `uuid` → `subnetId` |
| `flavor_list` (optional `need`) | flavors + suggested group | `flavorId` → `flavorId` |
| `sshkey_list` | SSH keys | `id` → `sshKeyId` |
| `secgroup_list` | security groups | `id` → `securityGroups[]` |
| `cluster_versions_list` | k8s versions (recommended marked) | → `version` + `releaseChannel` |

Accept names OR IDs from the user and resolve to the ID via these tools.

### Caching & freshness
Discovery results are cached briefly by the MCP server (TTL tiered by volatility: flavors/versions ~30 min; vpc/subnet/secgroup ~2 min; **ssh keys ~30 s**). The cache refreshes lazily when the TTL expires, and on server restart. It does **not** see resources created outside the MCP server (e.g. in the VNG Cloud console). Each discovery tool takes a **`refresh: true`** flag to bypass the cache. Pass `refresh: true` when:
- resuming after the user just created a resource in the console (especially an **SSH key** — see "No SSH key"),
- the user says "I just created X, check again".
Otherwise rely on the cache (don't set `refresh` on every call).

## Default-picking policy

| Parameter | Default the assistant picks | Ask the user only when |
|-----------|-----------------------------|------------------------|
| Region | config default (HCM-3) | user names a different region |
| VPC | the only VPC if exactly one | more than one → present them and ask (see "Presenting choices") |
| Subnet | the default/only subnet of the chosen VPC | more than one → present and ask |
| Security group | the `default` group for the project | user wants a specific group |
| Network type | `CILIUM_NATIVE_ROUTING` (no CIDR needed) | user wants `CALICO`/`CILIUM_OVERLAY` → then `cidr` is required |
| K8s version | latest STABLE (the recommended one) | user pins a version |
| Image type (`os`) | `ubuntu` | user wants `flatcar` |
| Disk type | `SSD` | user names another; if the API rejects `SSD`, ask for a `vtype-…` id |
| Disk size | `100` GB | user states a size (20–5000) |
| Node count | `1` for a default group; suggest `3` if the user says prod/HA | user states a count (0–10) |
| SSH key | the single configured key (VKS uses one) | none exists → guide to create one (see "No SSH key"); do not invent one |
| Plugins | load-balancer + block-store CSI enabled | user wants them off |
| Cluster name | required — if not given, ask for it (the only mandatory question) | always |
| Node group name | auto-propose `<cluster-name>-ng`; if that exceeds 15 chars or is invalid, use `default-ng` | user names one |

### Default flavor (when the user expresses no preference)
There is no hard-coded flavor id. Resolve the default at runtime: call `flavor_list need=Dev/test`; if non-empty, pick the **smallest** by vCPU then RAM. If that group is empty, call `flavor_list` unfiltered and pick the smallest. Show the chosen flavor in the plan as `[auto]` so the user can change it.

### Disk type (no discovery tool)
There is **no disk-type discovery tool**. Default `diskType` to `"SSD"` and show it as `[auto]`. If `cluster_create_validate` / the API rejects `"SSD"`, the value must be a vServer volume-type id (`vtype-…`) — ask the user to get it from the VNG Cloud console (vServer → Volumes / Volume types). Do not guess a `vtype-…` id.

## Flavor selection — by deployment need

If the user does not accept the default flavor, ask **"what will this run?"** then call `flavor_list` and show only the matching group:

| User's need | `need` filter / group |
|-------------|-----------------------|
| Dev / test / learning | `Dev/test` |
| Normal web / app | `Cân bằng` |
| Data / high CPU | `Compute` |
| High memory | `RAM cao` |
| AI / GPU | `AI/GPU` |

The user can always type a flavor name or `flavorId` directly if they already know it.

## Name validation

- Cluster: `^[a-z0-9][a-z0-9\-]{3,18}[a-z0-9]$` (5–20 chars, lowercase/digits/hyphens, start & end alphanumeric).
- Node group: `^[a-z0-9][a-z0-9-]{3,13}[a-z0-9]$` (5–15 chars).
- If a requested name fails, propose the closest valid form (e.g. `My_Cluster` → `my-cluster`) and confirm.

## Network types

- `CILIUM_NATIVE_ROUTING` — default; **no** `cidr` needed. Best for "just give me a cluster".
- `CALICO`, `CILIUM_OVERLAY` — require a pod `cidr` (e.g. `10.96.0.0/12`). Only use when the user asks.

## Create-body shape (CreateClusterComboDto / CreateNodeGroupDto)

Always run `cluster_create_validate` before `cluster_create`. A minimal cluster body:

```json
{
  "name": "<cluster-name>",
  "version": "<recommended version>",
  "releaseChannel": "STABLE",
  "networkType": "CILIUM_NATIVE_ROUTING",
  "enablePrivateCluster": false,
  "vpcId": "<from vpc_list .id>",
  "subnetId": "<from subnet_list .uuid>",
  "nodeGroups": {
    "name": "<ng-name>",
    "numNodes": 1,
    "flavorId": "<from flavor_list .flavorId>",
    "diskSize": 100,
    "diskType": "SSD",
    "os": "ubuntu",
    "enablePrivateNodes": false,
    "securityGroups": ["<from secgroup_list .id>"],
    "sshKeyId": "<from sshkey_list .id>"
  }
}
```

A node-group body is the `nodeGroups` object above, posted to a cluster via `nodegroup_create`.

## Presenting choices (when more than one option)
When a discovery tool returns several candidates and there's no safe single default (multiple VPCs, multiple subnets, a flavor-by-need list, multiple security groups), **show a short numbered list with the distinguishing attributes before asking** — never ask "which VPC?" with no context. Minimums: VPC → name + CIDR; subnet → name + CIDR; flavor → name + vCPU/RAM (+GPU); security group → name + description. The user may answer with a number, a name, or an id.

## Resolving a cluster / node group by name
The user usually names a cluster, not an id.
1. `cluster_list` and match the name. **Exactly one match** → use it. **No match** → say so and list the available clusters; do not guess. **Multiple partial matches** → list them and ask which.
2. For a node group, `nodegroup_list` on the cluster. If the user said a generic word like "workers" and there's exactly one node group, use it. If several, list them by name and ask — VKS node groups have names, not roles.

## No SSH key
VKS needs exactly one SSH key for node access and cannot create one for the user. If `sshkey_list` is empty, stop and tell the user to create a key pair in the VNG Cloud console (vServer → SSH Keys, e.g. `https://hcm-3.console.vngcloud.vn` → vServer → SSH Key; `han-1` console for HAN), then resume by saying `continue` or re-issuing the request. On resume, call `sshkey_list` with **`refresh: true`** (the new key was created in the console, so the short cache won't show it yet). Keep the already-gathered choices and resume from where you stopped — don't restart discovery from scratch.

## Node-group update body (nodegroup_update)
`nodegroup_update` takes a **partial** body — send only the fields being changed. Writable fields: `numNodes` (0–10), `securityGroups`, `labels`, `taints`, `autoScaleConfig`. To scale, send `{"numNodes": <n>}`. If the node group has `autoScaleConfig` enabled, a manual `numNodes` may be overridden by the autoscaler — surface this to the user before scaling. Show current → target before confirming, and warn that scaling down drains/removes nodes (workload disruption).

## Waiting for ACTIVE (no waiter tool)
Poll `cluster_get` / `nodegroup_get` about every 30 seconds. Report each status change rather than spinning silently. Timeout: ~15 min for a cluster, ~10 min for a node group. On timeout or an ERROR status, fetch `cluster_get_events` / inspect the node group, surface the likely cause, and tell the user to check the console — don't keep polling forever.

## Cluster / node-group statuses
Common states: `CREATING`, `ACTIVE` (ready), `UPDATING`/`RECONCILING` (a change is applying), `ERROR` (failed — pull events), `DELETING`. Report the raw status verbatim if it isn't in this list, and explain in plain language.

## Safety rules

- Never ask the user to paste a secret into chat.
- Mask sensitive values in any echoed output.
- Every write (create / update / delete / scale / upgrade) goes through a single explicit confirmation gate — see the hard-gate block in the calling skill.
- If the MCP server is read-only (no `--allow-write`), creating/scaling will fail — tell the user to restart it with `--allow-write`.
