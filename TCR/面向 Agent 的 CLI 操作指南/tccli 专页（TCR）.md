# tccli 专页（TCR）

> 页型 **T0** · tccli patterns for **`tccli tcr`** task pages · 不含 tcrctl

Task pages under **面向 Agent 的 CLI 操作指南** reuse the contracts below. See [环境准备](./环境准备.md) for install and credentials.

## Read actions (Describe* / Get*)

Use a one-line command with `--output json`:

```bash
tccli tcr DescribeInstances --region ap-guangzhou --output json
```

For namespace and repository reads:

```bash
tccli tcr DescribeNamespaces --RegistryId <RegistryId> --region ap-guangzhou --output json
tccli tcr DescribeRepositories --RegistryId <RegistryId> --NamespaceName <NamespaceName> --region ap-guangzhou --output json
```

## Complex write standard (JSON bridge)

Create/modify actions with multiple parameters (e.g. **CreateInstance**) use the **JSON bridge** pattern:

1. Submit the request with **`--cli-input-json file://…`** pointing at a curated minimal file in `examples/`.
2. Poll completion with `DescribeInstances` or `DescribeInstanceStatus` until the target status is reached.

```bash
tccli tcr CreateInstance --cli-input-json file://examples/CreateTcrInstance.min.json --region ap-guangzhou --output json
```

## Personal edition actions

TCR 个人版 uses a separate set of actions with `Personal` suffix — no `RegistryId` parameter needed:

```bash
tccli tcr CreateNamespacePersonal --Namespace <namespace-name> --region ap-guangzhou --output json
tccli tcr CreateRepositoryPersonal --RepoName "<namespace>/<repo>" --region ap-guangzhou --output json
```

## Discover full parameter shapes (reference only)

To inspect the full API parameter tree when extending JSON examples:

```bash
tccli tcr CreateInstance --generate-cli-skeleton --output json > /tmp/CreateInstance.skeleton.json
```

Prefer curated `examples/*.min.json` on task pages; merge only the fields you need from a skeleton.
