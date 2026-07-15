---
doc_type: Troubleshooting
subtype: 7B
---
# TCR 故障排查

## 从这里开始

```bash
tccli tcr DescribeInstances --region ap-guangzhou
# expected: { "TotalCount": "≥0", "Registries": [...] }
```

正常输出： 每个实例 `Status: "Running"`。任何非 Running 状态（`Creating`/`Deleting`/异常）或实例完全不出现在列表中，见 [状态机](reference/states.md) 或下方。

## 诊断工具箱

| 命令 | 检查什么 | 何时使用 |
|---------|---------------|------------|
| `tccli tcr DescribeInstances --Registryids '["<ID>"]'` | 实例状态 + 域名 + 规格 | 怀疑实例异常时首先执行 |
| `tccli tcr DescribeExternalEndpointStatus --RegistryId "<ID>"` | 公网访问开关状态 | docker login 超时 |
| `tccli tcr DescribeInternalEndpoints --RegistryId "<ID>"` | 内网 VPC 链接状态 | VPC 内 docker pull 失败 |
| `tccli tcr DescribeInstanceToken --RegistryId "<ID>"` | Token 列表 + 启用状态 | docker login 认证失败 |
| `tccli tcr DescribeSecurityPolicies --RegistryId "<ID>"` | 公网白名单列表 | 403 Forbidden |
| `tccli tcr DescribeNamespaces --RegistryId "<ID>"` | 命名空间列表 | 确认仓库是否存在 |
| `tccli tcr DescribeRepositories --RegistryId "<ID>"` | 仓库列表 + 详情 | push/pull 找不到仓库 |
| `tccli tcr DescribeImages --RegistryId "<ID>" --NamespaceName "<NS>" --RepositoryName "<REPO>"` | 镜像 Tag 列表 | 确认镜像是否推送成功 |

## 常见问题

### docker login 失败: unauthorized

**可能原因**： Token 不存在、已禁用、或已过期。

**诊断**：

```bash
tccli tcr DescribeInstanceToken --region ap-guangzhou --RegistryId "<REGISTRY_ID>"
# expected: 列出所有 Token。检查目标 Token 的 Enabled 状态和过期时间
```

如果 Token 被禁用 (`Enabled: false`) 或已过期:

**修复**：

```bash
# 方案 1: 重新启用 Token
tccli tcr ModifyInstanceToken --region ap-guangzhou \
  --RegistryId "<REGISTRY_ID>" \
  --TokenId "<TOKEN_ID>" \
  --Enable true
# expected: exit 0

# 方案 2: 创建新临时 Token（temp 默认 1 小时过期；长期用 longterm）
tccli tcr CreateInstanceToken --region ap-guangzhou \
  --RegistryId "<REGISTRY_ID>" \
  --TokenType temp \
  --Desc "Replacement Token"
# expected: {"Username":"<USER>","Token":"<TOKEN>","ExpTime":<TS>}
# ⚠️ 临时 Token 1 小时过期；长期凭证用控制台或服务账号
```

**验证**：

> docker CLI（镜像传输，非 tccli；TCCLI 不提供 docker daemon 操作能力）
```bash
docker login <REGISTRY_DOMAIN> --username <USERNAME> --password <TOKEN>
# expected: Login Succeeded
```

### docker login 超时或连接被拒绝

**可能原因**： 公网访问未开启，或客户端网络无法访问 Registry 域名。

**诊断**：

```bash
# 1. 检查公网端点
tccli tcr DescribeExternalEndpointStatus --region ap-guangzhou --RegistryId "<REGISTRY_ID>"
# expected: 查看 Status 是否为 "Opened"

# 2. 检查网络连通性
curl -v https://<REGISTRY_DOMAIN>/v2/
# expected: HTTP 401 (未授权但可达) 而不是连接超时
```

**修复**：

```bash
# 如果 Status 不是 "Opened"
tccli tcr ManageExternalEndpoint --region ap-guangzhou \
  --RegistryId "<REGISTRY_ID>" \
  --Operation Create
# expected: exit 0
```

如果开启后仍然超时，检查客户端到 Registry 域名的 443 端口连通性。如果你在防火墙/代理后:

```bash
# 配置 docker daemon 代理
# 编辑 /etc/docker/daemon.json 添加 HTTP_PROXY
```

**验证**：

```bash
tccli tcr DescribeExternalEndpointStatus --region ap-guangzhou --RegistryId "<REGISTRY_ID>"
# expected: Status: "Opened"

curl -s -o /dev/null -w "%{http_code}" https://<REGISTRY_DOMAIN>/v2/
# expected: 401 (端点可达)
```

### docker push/pull 返回 403 Forbidden

**可能原因**： 当前 IP 不在公网访问白名单中，或仓库权限不足。

**诊断**：

```bash
# 0. 先确认公网端点（Closed 时 DescribeSecurityPolicies 报 ResourceNotFound: Failed to get security group id）
tccli tcr DescribeExternalEndpointStatus --region ap-guangzhou --RegistryId "<REGISTRY_ID>"
# expected: Status=Opened；若 Closed → 先 ManageExternalEndpoint --Operation Create

# 1. 检查白名单
tccli tcr DescribeSecurityPolicies --region ap-guangzhou --RegistryId "<REGISTRY_ID>"
# expected: SecurityPolicySet 列表；Closed 端点 → ResourceNotFound（非「无白名单」）

# 2. 检查当前公网 IP
curl -s ifconfig.me
# expected: 你的公网 IP

# 对比: IP 是否在白名单中的某个 CidrBlock 范围内
```

**修复**：

```bash
# 添加当前 IP 到白名单
tccli tcr CreateSecurityPolicy --region ap-guangzhou \
  --RegistryId "<REGISTRY_ID>" \
  --CidrBlock "<YOUR_IP>/32" \
  --Description "Added via troubleshooting"
# expected: exit 0
```

**验证**：

> docker CLI（镜像传输，非 tccli；TCCLI 不提供 docker daemon 操作能力）
```bash
docker pull <REGISTRY_DOMAIN>/<NAMESPACE>/<REPO>:<TAG>
# expected: 镜像拉取成功，不再 403
```

### 实例状态不是 Running

**可能原因**： 实例初始化未完成、欠费、或实例异常。

**诊断**：

```bash
tccli tcr DescribeInstances --region ap-guangzhou --Registryids '["<REGISTRY_ID>"]'
# expected: 查看 Status 和 DescribeInstanceStatus 中的过程信息
```

**修复**：

| 状态 | 动作 |
|--------|------|
| `Creating` | 等待 3-5 分钟，新创建的实例需初始化 |
| `Deleting` | 实例正在删除，等待完成 |
| 非 Running 且非过渡态 | 查 `DescribeInstanceStatus` 的 `Conditions[].Reason`；欠费则充值，超 1 小时未恢复提工单 |

**验证**：

```bash
tccli tcr DescribeInstances --region ap-guangzhou --Registryids '["<REGISTRY_ID>"]'
# expected: Status: "Running"
```

### 镜像推送成功但 DescribeImages 查不到

**可能原因**： 推送到了错误的命名空间或仓库。

**诊断**：

```bash
# 列出所有命名空间和仓库
tccli tcr DescribeNamespaces --region ap-guangzhou --RegistryId "<REGISTRY_ID>"
tccli tcr DescribeRepositories --region ap-guangzhou --RegistryId "<REGISTRY_ID>"
# expected: 找到你推送的目标 Repository
```

**修复**：

> docker CLI（镜像传输，非 tccli；TCCLI 不提供 docker daemon 操作能力）
```bash
# 用正确的路径重新推送
docker tag <IMAGE> <REGISTRY_DOMAIN>/<CORRECT_NAMESPACE>/<CORRECT_REPO>:<TAG>
docker push <REGISTRY_DOMAIN>/<CORRECT_NAMESPACE>/<CORRECT_REPO>:<TAG>
# expected: push 成功
```

**验证**：

```bash
tccli tcr DescribeImages --region ap-guangzhou \
  --RegistryId "<REGISTRY_ID>" \
  --NamespaceName "<NAMESPACE>" \
  --RepositoryName "<REPO>"
# expected: ImageInfoList 包含你的 Tag
```

## 升级

如果以上步骤无法解决问题，收集以下信息:

```bash
# 1. 实例信息
tccli tcr DescribeInstances --region ap-guangzhou --Registryids '["<REGISTRY_ID>"]' > tcr-info.json
# expected: 文件写入成功，JSON 含 Registries[]

# 2. 访问配置
tccli tcr DescribeExternalEndpointStatus --region ap-guangzhou --RegistryId "<ID>" > tcr-endpoint.json
# expected: 文件写入成功，含 Status（Opened/Closed）
tccli tcr DescribeSecurityPolicies --region ap-guangzhou --RegistryId "<ID>" > tcr-policies.json
# expected: 文件写入成功；公网 Closed 时可能 ResourceNotFound

# 3. 操作日志 (CloudAudit)
# 从控制台获取或: tccli cloudaudit LookUpEvents --LookupAttributes '[{"AttributeKey":"ResourceName","AttributeValue":"<REGISTRY_ID>"}]'

# 4. 失败操作的 RequestId
# 从之前失败的 API 响应中获取
```

提交到: [腾讯云工单系统](https://console.cloud.tencent.com/workorder)，附带以上文件和 RequestId。

## 下一步

- [实例状态机](reference/states.md) — `Creating`/`Running`/`Deleting` 状态枚举
- [错误码](reference/error-codes.md) — docker CLI 错误（`unauthorized`/`denied`）+ TCR 特有码
- [访问控制](access/manage.md) — Token/白名单/VPC 内网配置
- [推送拉取镜像](images/push-pull.md) — 完整 push/pull 链路与失败模式
- [访问管理](instances/manage-access.md) — 公网/内网端点开启

