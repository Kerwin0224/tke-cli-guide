---
doc_type: Quickstart
fused: true
---

# TKE：创建第一个托管集群

> 官方文档: [TKE 产品文档](https://cloud.tencent.com/document/product/457) | 控制台: [TKE 集群创建](https://console.cloud.tencent.com/tke2/cluster/create?rid=1)
>
> ⚠️ **计费警告**: 创建托管集群（MANAGED_CLUSTER）即开始计收**集群管理费**（L5 为最低等级）。
> 完成本 Quickstart 后须执行 [Step 3: 删除集群](#step-3-删除集群清理) 以避免持续计费。
>
> **适用角色**: DevOps / SRE — 用 TCCLI 管理 TKE 集群。
>
> **相关文档**: 本文 → [TKE 概览](../tke/index.md) → [创建集群详解](../tke/clusters/create.md)
>
> **时间估计**：集群创建通常需要数分钟；实际时长受地域与后端资源影响。

---

## 触发条件

- 需用 TCCLI 验证创建并管理一个 TKE 集群（创建→查询→删除闭环）— 本篇是最短路径
- 终端执行 `tccli tke DescribeClusters` 返回空或需验证 TCCLI 集群管理能力 — 从 [Step 0 环境检查](#step-0-环境检查) 开始
- 要使用 TCCLI 的 `--waiter` 异步等待 + `--filter` JMESPath 过滤演练集群生命周期 — 本 Quickstart 含完整示例

## 准备工作

逐条验证，每条后附可执行命令。

| # | 条件 | 验证命令 | 预期结果 |
|:--|:-----|:--------|:---------|
| 1 | TCCLI 已安装 (≥ 3.0) | `tccli --version` | `3.x.x` 或更高 |
| 2 | 凭证已配置 | `tccli tke DescribeRegions` | 返回 `"RequestId"`，无 `Error` |
| 3 | 目标地域可用 | `tccli tke DescribeRegions` | 输出含 `ap-guangzhou`、`Status` 非空 |
| 4 | VPC 和子网已就绪 | `tccli vpc DescribeSubnets --region <REGION>` | 某子网 `AvailableIpAddressCount ≥ 10` |
| 5 | 服务角色 `TKE_QCSRole` | `tccli cam GetRole --RoleName TKE_QCSRole --filter "RoleInfo.RoleName" --output text` | 输出 `TKE_QCSRole`（不存在则 RoleNotExist） |
| 6 | 服务角色 `IPAMDofTKE_QCSRole`（本篇默认 VPC-CNI） | `tccli cam GetRole --RoleName IPAMDofTKE_QCSRole --filter "RoleInfo.RoleName" --output text` | 输出 `IPAMDofTKE_QCSRole`（不存在则 RoleNotExist） |
| 7 | kubectl 已安装（验证集群用） | `kubectl version --client` | Client Version 显示版本号 |

```bash
tccli --version
# expected: 3.1.126.1 或更高

tccli tke DescribeRegions \
    --filter "RegionInstanceSet[0].{name:RegionName,status:Status}" --output text
# expected: ap-guangzhou    alluser

# 服务角色（缺则创建会中途失败 / VPC-CNI 不可用；按名 GetRole，勿分页扫 DescribeRoleList）
tccli cam GetRole --RoleName TKE_QCSRole --filter "RoleInfo.RoleName" --output text
# expected: TKE_QCSRole
tccli cam GetRole --RoleName IPAMDofTKE_QCSRole --filter "RoleInfo.RoleName" --output text
# expected: IPAMDofTKE_QCSRole
```

> 未安装 TCCLI → [安装 TCCLI](../getting-started/install.md)。凭证未配置 → [配置凭证](../getting-started/credentials.md)（`tccli configure` 或 `tccli auth login`）。**缺服务角色** → [配置凭证 — 服务角色](../getting-started/credentials.md#服务角色tke--ipamd--as--tcr--可观测)（补 `TKE_QCSRole` + `IPAMDofTKE_QCSRole`）。缺少 VPC/子网 → [准备 VPC 与子网](../getting-started/prepare-vpc.md)。未装 kubectl → [kubectl 安装](https://kubernetes.io/docs/tasks/tools/)（验证集群用，非创建集群必需）。

---

## Step 0: 环境检查 {#step-0-环境检查}

查询当前地域可用的 K8s 版本，确认最新版本号。

```bash
tccli tke DescribeVersions --region <REGION> \
    --filter "VersionInstanceSet[-3:].Version" --output text
# expected (示例，text 为 tab 分隔单行；json 才是数组分行):
# 1.30.0	1.32.2	1.34.1
```

| 占位符 | 含义 | 约束 | 获取方式 |
|--------|------|------|---------|
| `<REGION>` | 目标地域 | 如 `ap-guangzhou` | `tccli tke DescribeRegions` 查看 `RegionName` |

> 本 Quickstart 使用 **1.34.1**（当前最新可用版本）。用 `tccli tke DescribeVersions --region <REGION> --output json` 查看完整版本列表及 `Remark`。

---

## Step 1: 创建集群

### 决策：集群类型

| 特性 | **MANAGED_CLUSTER (托管)** | INDEPENDENT_CLUSTER (独立) |
|:-----|:--------------------------|:---------------------------|
| Master 节点 | 腾讯云托管，不占用户资源 | 用户自管 3 个 Master 节点 |
| 管理费 | 按等级收取（L5 起） | 无管理费，但需付费 Master CVM |
| 运维 | 低 — 升级/备份由云平台处理 | 高 — 自行维护 |
| 高可用 | 内置 HA | 需自行配置 3 节点 |
| SLO | 99.95% | 取决于用户配置 |

> **推荐**: `MANAGED_CLUSTER` — 适用多数场景。

### 创建集群

命令使用 VPC-CNI 网络模式（Pod 直接从 VPC 子网获取 IP），最小规格 L5，无节点。

```bash
tccli tke CreateCluster \
    --region <REGION> \
    --ClusterType MANAGED_CLUSTER \
    --ClusterBasicSettings '{
        "ClusterName": "<CLUSTER_NAME>",
        "ClusterVersion": "<K8S_VERSION>",
        "VpcId": "<VPC_ID>",
        "SubnetId": "<SUBNET_ID>",
        "ClusterLevel": "L5"
    }' \
    --ClusterCIDRSettings '{
        "ServiceCIDR": "10.100.0.0/17",
        "MaxNodePodNum": 64,
        "MaxClusterServiceNum": 32768,
        "EniSubnetIds": ["<SUBNET_ID>"]
    }' \
    --ClusterAdvancedSettings '{
        "ContainerRuntime": "containerd",
        "DeletionProtection": false,
        "NetworkType": "VPC-CNI",
        "VpcCniType": "tke-route-eni",
        "IsNonStaticIpMode": true
    }'
# expected: {"ClusterId":"cls-xxxxxxxx","RequestId":"..."}
```

```json
{
    "ClusterId": "cls-xxxxxxxx",
    "RequestId": "..."
}
```

> 记下响应中的 `ClusterId`，后续步骤以 `<CLUSTER_ID>` 代指（示例输出中的 `cls-xxxxxxxx` 仅为虚构样例）。

**占位符说明**：

| 占位符 | 含义 | 约束 | 获取方式 |
|--------|------|------|---------|
| `<REGION>` | 目标地域 | 如 `ap-guangzhou` | `tccli tke DescribeRegions` |
| `<CLUSTER_NAME>` | 集群名称 | 1-60 字符，字母开头 | 自定义 |
| `<K8S_VERSION>` | 当前地域支持的 Kubernetes 版本 | 从 Step 0 的 `DescribeVersions` 结果中选择 | `tccli tke DescribeVersions --region <REGION>` |
| `<VPC_ID>` | VPC ID | 格式 `vpc-xxxxxxxx` | `tccli vpc DescribeVpcs --region <REGION>` |
| `<SUBNET_ID>` | 子网 ID | 格式 `subnet-xxxxxxxx`，VPC-CNI 模式必填 | `tccli vpc DescribeSubnets --region <REGION>` |
| `<CLUSTER_ID>` | 集群 ID | 格式 `cls-xxxxxxxx` | `CreateCluster` 响应的 `ClusterId` |

**关键参数**：

| 参数 | 说明 |
|:-----|:-----|
| `ClusterVersion` | K8s 版本，Step 0 中 `DescribeVersions` 查得的最新值 |
| `ClusterLevel` | 集群等级，`L5` 为最低（成本最小） |
| `ServiceCIDR` | Service 网段，**不得与 VPC CIDR 重叠**。若报 `CIDR_CONFLICT_WITH_VPC_CIDR` 则换网段 |
| `EniSubnetIds` | VPC-CNI 模式必需，填子网 ID |
| `ClusterCIDR` | **VPC-CNI 模式禁止传此参数**。若同时传 `ClusterCIDR` + `NetworkType: VPC-CNI` → `FailedOperation.Param: VPC-CNI cluster doesn't need clusterCIDR` |
| `NetworkType` | `VPC-CNI`：Pod 直接从 VPC 子网获取 IP。也可选 `GR`（Global Router），此时需传 `ClusterCIDR` |
| `DeletionProtection` | `false` — Quickstart 用完即删 |
| `ContainerRuntime` | `containerd` — 当前默认运行时 |

### 等待就绪

集群创建是异步的，用 `--waiter` 等待状态变为 `Running`。

```bash
tccli tke DescribeClusterStatus \
    --region <REGION> \
    --ClusterIds '["<CLUSTER_ID>"]' \
    --waiter '{"expr":"ClusterStatusSet[0].ClusterState","to":"Running","timeout":600,"interval":10}' \
    --output json
# expected: ClusterState = "Running"
```

```json
{
    "ClusterStatusSet": [
        {
            "ClusterId": "cls-xxxxxxxx",
            "ClusterState": "Running",
            "ClusterInstanceState": "",
            "ClusterBMonitor": false,
            "ClusterInitNodeNum": 0,
            "ClusterRunningNodeNum": 0,
            "ClusterFailedNodeNum": 0,
            "ClusterClosedNodeNum": 0,
            "ClusterClosingNodeNum": 0,
            "ClusterDeletionProtection": false,
            "ClusterAuditEnabled": false
        }
    ],
    "TotalCount": 1,
    "RequestId": "..."
}
```

> **空集群说明**：本 Quickstart 创建时**无 worker**，`ClusterRunningNodeNum=0` 时 `ClusterInstanceState` 常为**空字符串**（非异常）。有 ≥1 worker 且节点健康后才为 `AllNormal`。waiter 只盯 `ClusterState=Running` 即可。

| waiter 参数 | 值 | 说明 |
|:-----------|:----|:-----|
| `expr` | `ClusterStatusSet[0].ClusterState` | 指向状态字段的 JMESPath（注意字段名是 `ClusterState`，非 `ClusterStatus`） |
| `to` | `Running` | 目标状态 |
| `timeout` | `600` | 超时秒数（首次创建较慢） |
| `interval` | `10` | 轮询间隔秒数 |

> `--waiter` 参数必须用 **JSON 双引号**格式，不要用 Python dict 单引号。Shell 中整个 waiter JSON 用单引号包裹即可，内部 JSON key/value 用双引号。

### 验证（控制面创建成功 ≠ 本机 kubectl 已通）

> **边界**：下面维度 1–2 只证明 **托管控制面已 `Running`**。  
> **本机 / 公网 CI 的 kubectl 不会因此自动可用**——集群创建后默认 **无访问端点**；且部分环境对 kubeconfig 默认域名 `cls-*.ccs.tencent-cloud.com` **无法解析（NXDOMAIN）**。  
> 要在本机连 API Server，须另走包含 worker 与端点的扩展路径（**托管优先内网端点**；独立集群才走新接口公网端点），见下方 **「可选：让本机 kubectl 可达」** 与 [管理端点](../tke/networking/endpoints.md)。这不是完成本 Quickstart 最短闭环的必做条件，也不会要求用户为最短闭环额外增加付费 worker。

| 维度 | 命令 | 预期 | 含义 |
|:-----|:-----|:-----|:-----|
| 1. 状态 | `DescribeClusterStatus` → `ClusterState` | `Running` | 控制面就绪 |
| 2. 元数据 | `DescribeClusters` → `ClusterId`/`ClusterName`/`ClusterVersion`/`ClusterLevel` | 与创建参数一致 | 创建参数落库 |
| 3. 域名字段（可选） | `DescribeClusterSecurity` → `Domain` | 常为 `cls-xxxxxxxx.ccs.tencent-cloud.com` | **仅元数据**；**不等于**本机 DNS 可解析或 kubectl 已通 |
| 4. kubeconfig 文件 | `DescribeClusterKubeconfig` → `Kubeconfig` | 有效 YAML 可写盘 | **有凭证文件**；未开端点 + 未 patch `server` 时本机仍会失败 |

```bash
# 维度 1: 状态 = Running，删除保护 = false
tccli tke DescribeClusterStatus --region <REGION> --filter "ClusterStatusSet[?ClusterId=='<CLUSTER_ID>'] | [0].{state:ClusterState,protection:ClusterDeletionProtection}" --output json
# expected: {"state":"Running","protection":false}
```
```json
{"state": "Running", "protection": false}
```

```bash
# 维度 2: 详情字段完整
tccli tke DescribeClusters --region <REGION> --ClusterIds '["<CLUSTER_ID>"]' \
    --filter "Clusters[0].{id:ClusterId,name:ClusterName,ver:ClusterVersion,level:ClusterLevel}" --output json
# expected: id/name/ver/level 与创建参数一致
```
```json
{"id": "cls-xxxxxxxx", "name": "<CLUSTER_NAME>", "ver": "1.34.1", "level": "L5"}
```

```bash
# 维度 3（可选）: Domain 字段存在——不要把它当成「本机已可达」
tccli tke DescribeClusterSecurity --region <REGION> --ClusterId <CLUSTER_ID> \
    --filter "Domain" --output text
# expected: 常为 cls-xxxxxxxx.ccs.tencent-cloud.com（仅字段值；dig 可能 NXDOMAIN）
```

```bash
# 维度 4: 拉取 kubeconfig YAML（凭证敏感；此时通常仍不能本机 kubectl）
tccli tke DescribeClusterKubeconfig --region <REGION> --ClusterId <CLUSTER_ID> \
    --filter "Kubeconfig" --output text > ~/.kube/config-qs
# expected: 文件为有效 YAML；server 可能是 ccs 域名或内网 VIP
```

> ⚠️ kubeconfig 含访问凭证，勿提交到 git 或公开分享。  
> ⚠️ **不要**在未按模式开启端点、未确认对应 `ClusterIntranetEndpoint`/`ClusterExternalEndpoint` 前，把 `kubectl cluster-info` 成功当作本 Quickstart 的完成标准。

### 可选：让本机 kubectl 可达（需先按节点专题添加 worker）

空集群 `Running` 后，kubectl 仍可能不可达——须先按**集群模式**选端点路径（**不可把托管/独立串成同一条**）。**禁止** `CreateClusterEndpointVip`（官方废弃）。完整契约与故障码：[管理端点](../tke/networking/endpoints.md)。

| 集群模式 | 推荐入口 | 访问控制 | 不要做 |
|:---------|:---------|:---------|:-------|
| **托管**（本 Quickstart 默认） | `CreateClusterEndpoint --IsExtranet false` + 客户端已在集群 VPC（同 VPC / 专线 / VPN） | VPC 边界 + 可选 `--SecurityGroup` | **不要**把 `ModifyClusterEndpointSP` 接到新接口公网链；新接口对托管**仅内网**（`DescribeClusterEndpointStatus` 接口描述） |
| **独立** | `CreateClusterEndpoint --IsExtranet true` | 外网 CLB 绑定的**安全组**对客户端来源 IP 放通 TCP 443 | **不要**用 `ModifyClusterEndpointSP` 当独立集群新公网 ACL（该接口仅老托管外网端口） |

| 步 | 动作 | 成功判据 |
|:---|:-----|:---------|
| 1 | 添加 ≥1 worker（节点池或 `CreateClusterInstances`） | 节点 Ready；**无 worker 常无法开公网端点** |
| 2a 托管 | `CreateClusterEndpoint --IsExtranet false --SubnetId ...`（常可加 `--SecurityGroup`） | `DescribeClusterEndpointStatus --IsExtranet false` → **`Status=Created`**（不是 `Running`） |
| 2b 独立公网 | `CreateClusterEndpoint --IsExtranet true`（常需 `--SecurityGroup` + `ExtensiveParameters`） | `DescribeClusterEndpointStatus --IsExtranet true` → **`Status=Created`** |
| 3 | 读取端点地址 | 托管：`ClusterIntranetEndpoint` 非空；独立公网：`ClusterExternalEndpoint` 非空 |
| 4 | 将 kubeconfig 的 `server` 改为 `https://` + 上一步地址 | 不依赖可能 NXDOMAIN 的 `cls-*.ccs.tencent-cloud.com` |
| 5 | `kubectl get --raw=/healthz` 与 `get nodes` | **`ok`**；节点列表或空列表但命令成功 |

占位符：REGION / CLUSTER_ID / SUBNET_ID / SECURITY_GROUP_ID / 已有 worker

```bash
REGION=ap-guangzhou
CLUSTER_ID="<CLUSTER_ID>"
SUBNET_ID="<SUBNET_ID>"
SECURITY_GROUP_ID="<SECURITY_GROUP_ID>"
KUBECONFIG_PATH="${KUBECONFIG_PATH:-kubeconfig-qs.yaml}"
```

#### 1a. 托管：开内网端点（客户端须可达集群 VPC）

```bash
tccli tke CreateClusterEndpoint --region "$REGION" \
  --ClusterId "$CLUSTER_ID" --IsExtranet false \
  --SubnetId "$SUBNET_ID" --SecurityGroup "$SECURITY_GROUP_ID"
# expected: exit 0

tccli tke DescribeClusterEndpointStatus --region "$REGION" \
  --ClusterId "$CLUSTER_ID" --IsExtranet false \
  --waiter '{"expr":"Status","to":"Created","timeout":300,"interval":10}'
# expected: Status=Created
```

#### 1b. 独立集群：开公网端点（本机 / 公网 CI）

```bash
tccli tke CreateClusterEndpoint --region "$REGION" \
  --ClusterId "$CLUSTER_ID" --IsExtranet true \
  --SecurityGroup "$SECURITY_GROUP_ID" \
  --ExtensiveParameters '{"InternetAccessible":{"InternetChargeType":"TRAFFIC_POSTPAID_BY_HOUR","InternetMaxBandwidthOut":1}}'
# expected: exit 0

tccli tke DescribeClusterEndpointStatus --region "$REGION" \
  --ClusterId "$CLUSTER_ID" --IsExtranet true \
  --waiter '{"expr":"Status","to":"Created","timeout":300,"interval":10}'
# expected: Status=Created
```

> 公网路径：检查绑定到外网 CLB 的安全组，仅对客户端来源 IP 放通 TCP 443。`ModifyClusterEndpointSP` **仅**用于托管集群**老**外网端口的 `SecurityPolicies`/`SecurityGroup`，不要接到本段独立集群新公网链。

#### 2. 核对地址

```bash
# 托管内网：看 intranet；独立公网：看 ext
tccli tke DescribeClusterEndpoints --region "$REGION" --ClusterId "$CLUSTER_ID" \
  --filter "{ext:ClusterExternalEndpoint,intranet:ClusterIntranetEndpoint}" --output json
# expected: 所选路径对应字段非空
```

#### 3. kubeconfig + 改写 server

```bash
tccli tke DescribeClusterKubeconfig --region "$REGION" --ClusterId "$CLUSTER_ID" \
  --filter "Kubeconfig" --output text > "$KUBECONFIG_PATH"
# expected: 文件非空，内容为 kubeconfig YAML（含 clusters/users/contexts）

# 托管内网用 ClusterIntranetEndpoint；独立公网用 ClusterExternalEndpoint
EP_FIELD=ClusterIntranetEndpoint   # 独立公网改为 ClusterExternalEndpoint
EP="$(tccli tke DescribeClusterEndpoints --region "$REGION" --ClusterId "$CLUSTER_ID" \
  --filter "$EP_FIELD" --output text | tr -d '[:space:]')"
# expected: EP 非空
test -n "$EP"
case "$EP" in
  https://*) SERVER="$EP" ;;
  *) SERVER="https://${EP}" ;;
esac
python3 - "$KUBECONFIG_PATH" "$SERVER" <<'PY'
import re, sys
path, server = sys.argv[1], sys.argv[2]
text = open(path, encoding="utf-8").read()
new, n = re.subn(r"(?m)^(\s*server:\s*).*$", r"\1" + server, text, count=1)
if n != 1:
    raise SystemExit(f"rewrite server failed: replacements={n}")
open(path, "w", encoding="utf-8").write(new)
print("server ->", server)
PY
# expected: 打印 server -> https://...；kubeconfig 中 server 已改写为所选端点
```

#### 4. 强制确认：/healthz=ok

```bash
kubectl --kubeconfig "$KUBECONFIG_PATH" get --raw=/healthz --request-timeout=20s
# expected: ok
kubectl --kubeconfig "$KUBECONFIG_PATH" get nodes --request-timeout=20s
# expected: 命令成功
```

> **完成判据（缺一不可）**：`Status=Created` + 所选端点地址非空 + 客户端网络可达该入口 + **`/healthz=ok`**。  
> CAM / 空集群错误码与反模式：[管理端点](../tke/networking/endpoints.md)。  
> 加节点：[创建节点池](../tke/nodes/nodepool-create.md) · [新建 CVM 作节点](../tke/nodes/instance-ops.md)。  
> kubeconfig 证书面：[认证配置](../tke/security/auth.md)。

---

## Step 2: 查看集群

### 列表查询

筛选所有 `Running` 状态的集群，以表格形式展示。

```bash
tccli tke DescribeClusters --region <REGION> \
    --filter "Clusters[?ClusterStatus=='Running'].{id:ClusterId,name:ClusterName,ver:ClusterVersion,type:ClusterType}" \
    --output text
# expected: Tab 分隔的集群列表
# 注意：--output text 多键投影列序按投影 key 名字母序（id,name,type,ver），
# 非书写序；要固定列序请用 --output json
```
```text
cls-xxxxxxxx	<CLUSTER_NAME>	MANAGED_CLUSTER	1.34.1
```

### 单集群详情

```bash
tccli tke DescribeClusters --region <REGION> --ClusterIds '["<CLUSTER_ID>"]' \
    --filter "Clusters[0].{id:ClusterId,name:ClusterName,version:ClusterVersion,status:ClusterStatus,runtime:ContainerRuntime,vpc:ClusterNetworkSettings.VpcId}" \
    --output json
# expected: 返回集群完整元数据
```
```json
{
    "id": "cls-xxxxxxxx",
    "name": "<CLUSTER_NAME>",
    "version": "1.34.1",
    "status": "Running",
    "runtime": "containerd",
    "vpc": "<VPC_ID>"
}
```

> 列表查询的状态字段是 `Clusters[].ClusterStatus`（如 `"Running"`），与单集群健康查询 `DescribeClusterStatus` 的 `ClusterState` 字段名不同，注意区分。

---

## Step 3: 删除集群（清理） {#step-3-删除集群清理}

> ⚠️ **不可逆**: 集群删除后元数据**无法恢复**，工作负载全部丢失。
> `--InstanceDeleteMode terminate` 会销毁关联的 CVM 节点。
> **不会自动删除**的关联资源: CBS 云硬盘、EIP、CLB —— 请手动清理。

### 前置检查

确认删除保护已关闭。

```bash
tccli tke DescribeClusterStatus --region <REGION> --filter "ClusterStatusSet[?ClusterId=='<CLUSTER_ID>'] | [0].ClusterDeletionProtection" --output text
# expected: false
```
```text
false
```

若输出 `true`，先关闭保护：

```bash
tccli tke DisableClusterDeletionProtection --region <REGION> --ClusterId <CLUSTER_ID>
# expected: {"RequestId":"..."}
```

### 执行删除

```bash
tccli tke DeleteCluster \
    --region <REGION> \
    --ClusterId <CLUSTER_ID> \
    --InstanceDeleteMode terminate \
    --output json
# expected: {"RequestId":"..."}
```
```json
{"RequestId": "..."}
```

| 参数 | 说明 |
|:-----|:-----|
| `--InstanceDeleteMode terminate` | **推荐**: 销毁关联 CVM 节点，避免孤立计费 |
| `--InstanceDeleteMode retain` | 仅删管理面，CVM 保留为孤立实例（继续计费） |

### 验证删除

```bash
tccli tke DescribeClusters --region <REGION> --ClusterIds '["<CLUSTER_ID>"]' \
    --output json
# expected: TotalCount = 0，集群已从列表消失
```
```json
{"TotalCount": 0, "Clusters": [], "RequestId": "..."}
```

`TotalCount: 0` → 目标集群已删除。删除是异步操作；只执行一次 `DeleteCluster`，随后在查询型 `DescribeClusters` 上使用 TCCLI waiter 等待目标 ID 消失。以下等待最多 10 分钟、每 10 秒查询一次：

```bash
tccli tke DescribeClusters --region <REGION> \
    --ClusterIds '["<CLUSTER_ID>"]' \
    --waiter '{"expr":"TotalCount","to":0,"timeout":600,"interval":10}' \
    --output json
# expected: TotalCount = 0；waiter 超时则命令失败，不把删除判为成功
```

> `to` 使用 JSON 数字 `0`，与 `TotalCount` 的数值类型一致。超时后不要重复执行 `DeleteCluster`；保留首次删除返回的 `RequestId`，用 `tccli tke DescribeTasks --region <REGION> --Filter '[{"Name":"ClusterId","Values":["<CLUSTER_ID>"]},{"Name":"TaskType","Values":["<TASK_TYPE>"]}]' --Latest true`（`ClusterId`/`TaskType` 均在 `Filter` 内，**无**顶层 `--ClusterId`；`TaskType` 必传，取值如 `node_upgrade`/`add_cluster_cidr`/`master_upgrade`）和 `DescribeClusterStatus` 检查任务/状态，持续失败时携 RequestId 提交工单。

---

## 故障恢复

### 命令返回错误 (exit ≠ 0)

| 错误码 / 信号 | 诊断 | 根因 | 修复 |
|:-------------|:-----|:-----|:-----|
| `FailedOperation.Param: VPC-CNI cluster doesn't need clusterCIDR` | 检查 `--ClusterCIDRSettings` 是否含 `ClusterCIDR` | VPC-CNI 模式下 Pod IP 来自 VPC 子网，不需要 `ClusterCIDR` | 移除 `ClusterCIDRSettings` 中的 `ClusterCIDR` 字段 |
| `InvalidParameter.CidrConflictWithVpcCidr` | 对比 `ServiceCIDR` 与 VPC CIDR | Service 网段与 VPC CIDR 重叠 | 换不重叠 CIDR（如 `10.100.0.0/17`） |
| `EniSubnetIds must be set` | 检查是否传了 `EniSubnetIds` | VPC-CNI 模式缺弹性网卡子网 | 在 `ClusterCIDRSettings` 中加 `"EniSubnetIds":["<SUBNET_ID>"]` |
| `DeleteCluster` 返回 exit 252 | 检查命令是否含 `--InstanceDeleteMode` | 缺少必填参数 `InstanceDeleteMode` | 补 `--InstanceDeleteMode terminate` 或 `retain` |
| 删除被 API 拒绝 | `DescribeClusterStatus` 查 `ClusterDeletionProtection` | 删除保护开启中 | 先执行 `tccli tke DisableClusterDeletionProtection --region <REGION> --ClusterId <CLUSTER_ID>` |
| 集群卡 `Creating` > 10min | `tccli tke DescribeTasks --region <REGION> --Filter '[{"Name":"ClusterId","Values":["<CLUSTER_ID>"]},{"Name":"TaskType","Values":["<TASK_TYPE>"]}]' --Latest true` 查任务状态（`TaskType` 必传，如 `node_upgrade`/`add_cluster_cidr`/`master_upgrade`；**无**顶层 `--ClusterId`） | 子网 IP 不足或后端异常 | 换子网重试；若持续失败则提工单并附 `RequestId` |

### 命令成功但状态不对 (exit = 0)

| 现象 | 诊断 | 根因 | 说明 |
|:-----|:-----|:-----|:-----|
| 删除后 `DescribeClusters` 仍有记录 | `DescribeClusters --ClusterIds '["<CLUSTER_ID>"]' --filter "TotalCount" --output text` | 删除是异步操作，短暂窗口内列表未刷新 | 使用上方 10 分钟有界 waiter；超时后用 `DescribeTasks --Filter '[{"Name":"ClusterId","Values":["<CLUSTER_ID>"]},{"Name":"TaskType","Values":["<TASK_TYPE>"]}]' --Latest true` / `DescribeClusterStatus` 诊断并保留 RequestId |
| 删除后 CVM 仍存在 | `tccli cvm DescribeInstances --region <REGION>` | 删除时可能用了 `--InstanceDeleteMode retain` | 手动终止残留 CVM，下次使用 `terminate` 模式 |
| `DescribeClusterSecurity` 返回空字段 | `DescribeClusterStatus` 查 `ClusterState` | 集群非 `Running` 时端点不可用 | 等待集群 `Running` 后再查 |

---

## 收尾确认

```bash
# 目标集群删除判据（仅限本次记录的 CLUSTER_ID）
tccli tke DescribeClusters --region <REGION> --ClusterIds '["<CLUSTER_ID>"]' \
  --filter "TotalCount" --output text
# expected: 0
```

DeleteCluster 不会自动删除所有关联资源。CBS/EIP/CLB 只能按创建阶段记录的资源 ID 或明确标签逐个核对；若本 Quickstart 的空集群未创建这些资源，则无需把账号全局存量清零。无法建立归属时，全局 `TotalCount` 只能作为人工盘点信息，不能作为本次清理通过条件，也不得据此删除无关资源。

> 本 Quickstart 的闭环判据是：目标 `<CLUSTER_ID>` 查询 `TotalCount=0`，并且创建期间记录的、可归属于该集群的资源 ID 均已处理。账号内其他合法 CBS/EIP/CLB 不影响判定。

---

## 下一步

### 上线前检查 {#上线前检查}

Quickstart 闭环后、生产部署前，对照下列项（完整表见 [容器应用部署 Check List](https://cloud.tencent.com/document/product/457/41497)）：

| 类别 | 检查项 | 未做的后果 | 本指南入口 |
|:-----|:-------|:-----------|:-----------|
| 网络 | 创建前规划节点网段与容器网段容量 | 扩容时节点/Pod IP 不足 | [创建集群 — 创建前必读](../tke/clusters/create.md#创建前必读创建后改不了) · [网络管理](../tke/networking/index.md) |
| 网络 | 专线/对等连接/VPN 场景避免网段冲突 | 跨网互通失败 | [准备 VPC](../getting-started/prepare-vpc.md) |
| 安全 | 安全组按推荐放通容器/节点网段 | 节点 NotReady、DNS 失败 | [创建节点池 — 安全组](../tke/nodes/nodepool-create.md#安全组节点加入前) |
| 部署 | 运行时（containerd/docker）创建时选定 | 改运行时仅影响无节点池归属的增量节点 | [创建集群](../tke/clusters/create.md) |
| 部署 | IPVS 仅创建时可开，开后不可关 | 转发模式锁死 | [网络 — 转发模式半常量](../tke/networking/index.md#转发模式半常量与-networktype-正交) |
| 部署 | **新建**用托管集群（独立已停止新建） | 误选独立则无法新建 | [集群管理](../tke/clusters/index.md) |
| 工作负载 | 设 CPU/内存 limit；配存活/就绪探针 | 资源争抢或业务挂了 Pod 仍 Ready | 工作负载 YAML / 控制台 |

- [给集群添加节点](../tke/nodes/nodepool-create.md) — 空集群 Running 后先加工人节点
- [管理端点](../tke/networking/endpoints.md) — **本机/公网 CI 必读**：公网端点 + ACL + CLB 地址改写 kubeconfig
- [认证配置](../tke/security/auth.md) — kubeconfig 获取与轮转
- [创建集群详解](../tke/clusters/create.md) — 完整参数、高级配置（Global Router 模式、双栈、IPVS）
- [查询和过滤集群](../tke/clusters/query.md) — JMESPath 高级过滤技巧
- [删除集群详解](../tke/clusters/delete.md) — 残留资源清理、批量删除
- [TKE 拉取 TCR 镜像](../cross-product/tke-pull-tcr.md) — 跨产品：集群工作负载使用 TCR 私有镜像

---
