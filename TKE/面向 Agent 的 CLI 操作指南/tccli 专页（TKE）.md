# tccli 专页（TKE）

> 页型 **T0** · tccli patterns for **`tccli tke`** task pages · 不含 tkectl

Task pages under **面向 Agent 的 CLI 操作指南** reuse the contracts below. See [环境准备](./环境准备.md) for install and credentials.

## Read actions (Describe*)

Use a one-line command with `--output json`. Optional result shaping via `--filter` (JMESPath):

```bash
tccli tke DescribeClusters --region ap-guangzhou --output json
# Optional filter (JMESPath via --filter):
# tccli tke DescribeClusters --region ap-guangzhou --output json --filter 'TotalCount'
```

## Complex write standard (JSON bridge + waiter)

Async create/update actions with large payloads (e.g. **CreateCluster**) use the **JSON bridge** pattern — not bare `--output json` stubs:

1. Submit the request with **`--cli-input-json file://…`** pointing at a curated minimal file in `examples/` (e.g. `examples/CreateCluster.min.json`).
2. Poll completion with **`--waiter`** on a follow-up Describe* call until the target field reaches the expected value.

```bash
tccli tke CreateCluster --cli-input-json file://examples/CreateCluster.min.json --region ap-guangzhou --output json
tccli tke DescribeClusters --ClusterIds '["<cluster-id>"]' --region ap-guangzhou --output json --waiter "{'expr':'Clusters[0].ClusterStatus','to':'Running'}"
```

Edit placeholder IDs in `examples/*.min.json` before running; files are repo-managed minimal payloads aligned with official product/API semantics.

## Discover full parameter shapes (reference only)

To inspect the full API parameter tree when extending JSON examples, use **`--generate-cli-skeleton`** as a **T0 reference** — task pages do not rely on skeleton output at runtime:

```bash
tccli tke CreateCluster --generate-cli-skeleton --output json > /tmp/CreateCluster.skeleton.json
```

Prefer curated `examples/*.min.json` on task pages; merge only the fields you need from a skeleton.
