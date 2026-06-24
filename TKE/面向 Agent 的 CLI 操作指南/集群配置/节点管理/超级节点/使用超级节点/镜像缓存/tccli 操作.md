# 镜像缓存

> 对照官方：[镜像缓存](https://cloud.tencent.com/document/product/457/65908) · page_id `65908` · tccli ≥3.1.107 · API 2018-05-25

## 概述

**镜像缓存**加速超级节点上 Pod 的镜像拉取速度。TKE 提供两种互补的镜像缓存机制：**数据面 `cbs-reuse-key` Annotation**（kubectl）和 **控制面 ImageCache CRD**（tccli）。

| 缓存方式 | 配置位置 | 触发方式 | 适用场景 |
|---------|---------|---------|---------|
| `cbs-reuse-key` Annotation | Pod `.metadata.annotations` | Pod 启动时被动匹配已有磁盘 | 同镜像多次启动，动态复用已有 CBS 磁盘上的镜像层 |
| ImageCache CRD（手动） | tccli `CreateImageCache` | 预创建快照，Pod 启动时主动挂载 | 预缓存指定镜像，批量加速大规模 Pod 启动 |
| ImageCache CRD（自动） | 超级节点后端自动触发 | 首次创建 EKSCI 实例时自动生成 | 无需手动管理，首次使用后自动缓存 |

### 两者关系

`cbs-reuse-key` 和 ImageCache CRD **互补而非互斥**：

- `cbs-reuse-key` 是**数据面机制**：在 Pod 声明中嵌入缓存复用键，Pod 启动时自动匹配已有磁盘上的镜像层，适合同镜像反复创建的场景。
- ImageCache CRD 是**控制面机制**：预创建镜像缓存资源，提前将镜像下载到 CBS 快照，创建 EKSCI 实例时直接挂载快照避免拉取。更可预测（预缓存 vs 被动累积），适合批量 Pod 启动场景。
- **常见误解**：在已有 `cbs-reuse-key` 工作机制的集群中，误以为 ImageCache CRD 是替代品。两者可共存：`cbs-reuse-key` 适合动态缓存，ImageCache CRD 适合预缓存。

### 手动 vs 自动镜像缓存

| 类型 | `ImageCacheType` | 创建方式 | 适用场景 |
|------|:--:|------|------|
| 手动 | `manual` | `tccli tke CreateImageCache` | 精确控制缓存的镜像列表和大小 |
| 自动 | `auto` | 超级节点后端自动生成 | 首次创建 EKSCI 实例时自动触发 |

通过 `DescribeImageCaches` 返回的 `ImageCacheType` 字段区分。

### ImageCache 生命周期

创建镜像缓存的事件链：**创建云硬盘 → 创建 EKSCI 实例拉取镜像 → 创建快照 → 删除 EKSCI → 删除云硬盘 → Status=Ready**。

`Events` 字段记录了完整创建过程：`DiskCreated` → `EksCiCreated` → `SnapCreated` → `EksCiDeleted` → `DiskDeleted` 五个事件依次出现即为正常流程。若中间阶段失败，可从 Events 定位具体阶段。

`RetentionDays=0` 表示永不过期，生产环境建议设置合理保留天数（如 7-30 天），避免快照累积产生费用。

## 前置条件

- [环境准备](../../../../../环境准备.md)

### 环境检查

```bash
# 1. 检查 tccli 版本
tccli --version
# expected: tccli version >= 1.0.0

# 2. 检查 kubectl 已安装且可连接集群
kubectl version --client
# expected: 显示 Client Version

# 3. 检查 CAM 权限 — 需要以下 Action 名
#    tke:DescribeClusterVirtualNodePools, tke:DescribeClusterKubeconfig
#    tke:CreateImageCache, tke:DescribeImageCaches, tke:GetMostSuitableImageCache
# 验证：
tccli tke DescribeClusterVirtualNodePools --region <Region> --ClusterId <ClusterId>
# expected: exit 0
```

### 资源检查

```bash
# 4. 确认超级节点池已创建且状态正常
tccli tke DescribeClusterVirtualNodePools --region <Region> \
    --ClusterId <ClusterId>
# expected: NodePoolSet 非空，LifeState 为 normal

# 5. 获取 kubeconfig
tccli tke DescribeClusterKubeconfig --region <Region> --ClusterId <ClusterId> \
    | jq -r '.Kubeconfig' > ~/.kube/config-super
kubectl --kubeconfig ~/.kube/config-super cluster-info
# expected: Kubernetes control plane 可达
```

```json
{
  "TotalCount": "<TotalCount>",
  "NodePoolSet": [
    {
      "NodePoolId": "<NodePoolId>",
      "Name": "<Name>",
      "LifeState": "<LifeState>",
      "SubnetIds": ["<SubnetId>"]
    }
  ],
  "RequestId": "..."
}
```

**预期输出**（`DescribeClusterKubeconfig`）：

```json
{
    "Kubeconfig": "<base64-encoded-kubeconfig>",
    "RequestId": "..."
}
```

## 控制台与 CLI 参数映射

| 控制台操作 | tccli / kubectl | 幂等 |
|-----------|-----------|:--:|
| 查看超级节点池 | `DescribeClusterVirtualNodePools` | 是 |
| 获取 kubeconfig | `DescribeClusterKubeconfig` | 是 |
| 创建镜像缓存 | `CreateImageCache` | 否 |
| 查看镜像缓存（按 ID） | `DescribeImageCaches --ImageCacheIds` | 是 |
| 查看镜像缓存（按名称） | `DescribeImageCaches --ImageCacheNames` | 是 |
| 列出所有镜像缓存（分页） | `DescribeImageCaches --Offset --Limit` | 是 |
| 查询最适镜像缓存 | `GetMostSuitableImageCache` | 是 |
| 更新镜像缓存 | `UpdateImageCache` | 是 |
| 删除镜像缓存 | `DeleteImageCaches` | 否 |
| 声明镜像缓存 Annotation | `kubectl apply -f` | 是 |
| 查看 Pod 启动时间 | `kubectl describe pod` | 是 |

## 操作步骤

### Part A：cbs-reuse-key Annotation（kubectl）

适用于同镜像反复创建 Pod、依赖已有 CBS 磁盘上镜像层的场景。

#### 步骤 A1：确认超级节点就绪

```bash
tccli tke DescribeClusterVirtualNodePools --region <Region> \
    --ClusterId <ClusterId>
# expected: exit 0，NodePoolSet 中目标节点池 LifeState 为 normal
```

**预期输出**：

```json
{
  "TotalCount": "<TotalCount>",
  "NodePoolSet": [
    {
      "NodePoolId": "<NodePoolId>",
      "Name": "<Name>",
      "LifeState": "normal",
      "SubnetIds": ["<SubnetId>"]
    }
  ],
  "RequestId": "..."
}
```

#### 步骤 A2：创建带镜像缓存 Annotation 的 Pod

##### 选择依据

- `cbs-reuse-key` 是镜像缓存复用的核心 Annotation。相同 key 的 Pod 按顺序复用同一块 CBS 磁盘，后续 Pod 可避免重新拉取已有镜像层。
- **缓存 key 命名**：按镜像名或业务维度命名，如 `"nginx-cache"` 或 `"app-v1-base"`。不同镜像不应使用相同 key，否则缓存内容不匹配。
- **首次拉取**：使用新 key 的第一个 Pod 仍会正常拉取镜像。后续使用相同 key 的 Pod 才享受缓存加速。

`pod-image-cache.yaml`：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: <PodName>
  annotations:
    eks.tke.cloud.tencent.com/cpu: "1"
    eks.tke.cloud.tencent.com/mem: "2Gi"
    eks.tke.cloud.tencent.com/cbs-reuse-key: "<CacheKey>"
spec:
  containers:
  - name: <ContainerName>
    image: nginx:1.27-alpine
    resources:
      requests:
        cpu: "1"
        memory: "2Gi"
```

```bash
kubectl --kubeconfig ~/.kube/config-super apply -f pod-image-cache.yaml
# expected: pod/<PodName> created
```

| 占位符 | 说明 | 约束 | 获取方式 |
|--------|------|------|---------|
| `<PodName>` | Pod 名称 | 须遵循 K8s 命名规范 | 自定义 |
| `<CacheKey>` | 镜像缓存复用键 | 同一组共享缓存的 Pod 使用相同 key | 自定义 |
| `<ContainerName>` | 容器名称 | 须遵循 K8s 命名规范 | 自定义 |

#### 步骤 A3：验证缓存效果

首次创建使用新 key 的 Pod 时，须等待镜像拉取完成。后续使用相同 key 的 Pod 启动时间应明显缩短。

```bash
# 查看首个 Pod 启动时间（含镜像拉取）
kubectl --kubeconfig ~/.kube/config-super describe pod <PodName> | grep -A10 "Events"
# expected: Events 中包含 "Pulling image" 和 "Started container" 事件
# 首个 Pod：Pulling image 到 Started container 间隔较长（取决于镜像大小）
```

**预期输出**（首个 Pod Events）：

```text
Events:
  Type    Reason          Age   From               Message
  ----    ------          ----  ----               -------
  Normal  Scheduled       30s   default-scheduler  Successfully assigned ...
  Normal  Pulling         28s   kubelet            Pulling image "nginx:1.27-alpine"
  Normal  Pulled          10s   kubelet            Successfully pulled image ...
  Normal  Created         9s    kubelet            Created container <ContainerName>
  Normal  Started         8s    kubelet            Started container <ContainerName>
```

`pod-image-cache-2.yaml`：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: <PodName2>
  annotations:
    eks.tke.cloud.tencent.com/cpu: "1"
    eks.tke.cloud.tencent.com/mem: "2Gi"
    eks.tke.cloud.tencent.com/cbs-reuse-key: "<CacheKey>"
spec:
  containers:
  - name: <ContainerName>
    image: nginx:1.27-alpine
    resources:
      requests:
        cpu: "1"
        memory: "2Gi"
```

```bash
# 创建第二个使用相同缓存 key 的 Pod（名称不同）
kubectl --kubeconfig ~/.kube/config-super apply -f pod-image-cache-2.yaml
# expected: pod/<PodName2> created

# 查看第二个 Pod 启动时间
kubectl --kubeconfig ~/.kube/config-super describe pod <PodName2> | grep -A10 "Events"
# expected: 相较于首个 Pod，Pulling image 到 Started container 间隔明显缩短
```

**预期输出**（第二个 Pod Events — 缓存命中，无镜像拉取）：

```text
Events:
  Type    Reason          Age   From               Message
  ----    ------          ----  ----               -------
  Normal  Scheduled       5s    default-scheduler  Successfully assigned ...
  Normal  Created         3s    kubelet            Created container <ContainerName>
  Normal  Started         2s    kubelet            Started container <ContainerName>
```

### Part B：ImageCache CRD（tccli）

适用于预缓存指定镜像、批量加速大规模 Pod 启动的场景。镜像缓存创建后通过 CBS 快照的形式持久化。

#### 步骤 B1：创建镜像缓存

##### 选择依据

- **手动创建**（`ImageCacheType=manual`）：可精确控制缓存的镜像列表和大小，推荐生产环境使用。
- **`--AutoCreateEip true`** 必须显式开启，否则 EKSCI 实例无外网访问能力，无法从公网镜像仓库拉取镜像。
- **`RetentionDays`**：0 表示永不过期。生产环境建议设置 7-30 天，避免快照累积产生费用。

##### 最小参数版

```bash
tccli tke CreateImageCache \
    --VpcId <VpcId> \
    --SubnetId <SubnetId> \
    --Images '["nginx:1.27-alpine"]' \
    --ImageCacheName <ImageCacheName> \
    --region <Region> \
    --AutoCreateEip true
# expected: exit 0，返回 ImageCacheId
```

**预期输出**：

```json
{
    "ImageCacheId": "imc-xxxxxx",
    "RequestId": "..."
}
```

##### 完整参数版（含标签）

```bash
tccli tke CreateImageCache \
    --VpcId <VpcId> \
    --SubnetId <SubnetId> \
    --Images '["nginx:1.27-alpine","busybox:1.36"]' \
    --ImageCacheName <ImageCacheName> \
    --ImageCacheSize 20 \
    --RetentionDays 7 \
    --region <Region> \
    --AutoCreateEip true
# expected: exit 0，返回 ImageCacheId
```

**预期输出**：

```json
{
    "ImageCacheId": "imc-yyyyyy",
    "RequestId": "..."
}
```

| 占位符 | 说明 | 约束 | 获取方式 |
|--------|------|------|---------|
| `<VpcId>` | VPC ID | 须与超级节点池同一 VPC | `DescribeClusters` 查询集群详情 |
| `<SubnetId>` | 子网 ID | 须与超级节点池同一子网 | `DescribeClusters` 查询集群详情 |
| `<ImageCacheName>` | 镜像缓存名称 | 长度 1-63 字符 | 自定义 |
| `<Region>` | 地域 | 须与集群相同地域 | — |

##### 参数说明

| 参数 | 类型 | 必填 | 取值与约束 | 说明 | 错误后果 |
|------|------|:--:|------|------|------|
| `--Images` | Array of String | 是 | 格式 `["img:tag", ...]`，每项为有效镜像引用 | 需缓存的镜像列表 | 镜像名或标签拼写错误导致缓存创建成功但 Pod 拉取时镜像不匹配 |
| `--VpcId` | String | 是 | 须与超级节点池同一 VPC | VPC ID | 填写错误导致 EKSCI 实例无法访问集群网络，镜像缓存创建后无法使用 |
| `--SubnetId` | String | 是 | 须与超级节点池同一子网 | 子网 ID | 填写错误导致 EKSCI 实例无法分配到正确子网，创建失败 |
| `--ImageCacheName` | String | 否 | 长度 1-63 字符 | 镜像缓存名称，便于后续按名称检索 | 缺省时只能通过 ID 检索，影响运维效率 |
| `--ImageCacheSize` | Integer | 否 | 范围 20-32768（GiB），默认 20 | 缓存磁盘大小（GiB） | 设置过小导致镜像层放不下，缓存创建失败；设置过大浪费磁盘空间和费用 |
| `--RetentionDays` | Integer | 否 | 范围 0-65535，0=永不过期 | 保留天数 | 设为 0 导致快照永不自动删除，按月持续产生 CBS 快照费用 |
| `--AutoCreateEip` | Boolean | 否 | true/false，默认 false | 是否自动创建 EIP，用于拉取公网镜像 | 设为 false 且未提供 ExistedEipId，EKSCI 实例无外网访问能力，镜像拉取失败 |
| `--ExistedEipId` | String | 否 | 与 AutoCreateEip 二选一，须为有效 EIP ID | 已有 EIP ID | EIP 已被释放或 ID 拼写错误导致镜像拉取失败 |
| `--SecurityGroupIds` | Array of String | 否 | 格式 `["sg-xxx"]`，须为有效安全组 ID | 安全组 ID 列表 | 安全组规则过严导致 EKSCI 实例无法访问镜像仓库，拉取超时 |
| `--ImageRegistryCredentials` | Array | 否 | 私有镜像仓库凭证数组 | 私有镜像仓库凭证 | 凭证过期或错误导致私有镜像拉取失败 |
| `--Tags` | Array of Tag | 否 | 格式 `[{"Key":"k","Value":"v"}]` | 标签列表，用于资源分类和计费追踪 | 缺省不阻塞创建，但影响资源管理效率 |

#### 步骤 B2：查看镜像缓存

##### 按 ID 查看详情

```bash
tccli tke DescribeImageCaches \
    --ImageCacheIds '["<ImageCacheId>"]' \
    --region <Region>
# expected: exit 0，返回指定镜像缓存的完整信息
```

**预期输出**：

```json
{
    "TotalCount": 1,
    "ImageCaches": [
        {
            "ImageCacheId": "imc-xxxxxx",
            "ImageCacheName": "<ImageCacheName>",
            "ImageCacheSize": 20,
            "Images": ["nginx:1.27-alpine"],
            "CreationTime": "...",
            "ExpireDateTime": "...",
            "SnapshotId": "snap-xxxxxx",
            "Status": "Ready",
            "RetentionDays": 0,
            "ImageRegistryCredentials": [],
            "ImageCacheType": "manual",
            "Snapshotter": "overlay",
            "Events": [
                {"Type": "Normal", "Reason": "DiskCreated", "Message": "create disk disk-xxxxxx successfully"},
                {"Type": "Normal", "Reason": "EksCiCreated", "Message": "create EksCi eksci-xxxxxx successfully"},
                {"Type": "Normal", "Reason": "SnapCreated", "Message": "create snapshot snap-xxxxxx successfully"},
                {"Type": "Normal", "Reason": "EksCiDeleted", "Message": "delete EksCi eksci-xxxxxx successfully"},
                {"Type": "Normal", "Reason": "DiskDeleted", "Message": "delete disk disk-xxxxxx successfully"}
            ]
        }
    ],
    "RequestId": "..."
}
```

##### 按名称查询

```bash
tccli tke DescribeImageCaches \
    --ImageCacheNames '["<ImageCacheName>"]' \
    --region <Region>
# expected: exit 0，返回匹配的镜像缓存列表
```

**预期输出**：

```json
{
    "TotalCount": 1,
    "ImageCaches": [
        {
            "ImageCacheId": "imc-xxxxxx",
            "ImageCacheName": "<ImageCacheName>",
            "Status": "Ready"
        }
    ],
    "RequestId": "..."
}
```

##### 分页列出所有镜像缓存

```bash
tccli tke DescribeImageCaches \
    --region <Region> \
    --Limit 5 \
    --Offset 0
# expected: exit 0，返回镜像缓存分页列表
```

**预期输出**：

```json
{
    "TotalCount": 2,
    "ImageCaches": [
        {
            "ImageCacheId": "imc-aaaaaa",
            "ImageCacheName": "",
            "Status": "Ready",
            "Images": ["alpine:3.20", "nginx:1.27-alpine"],
            "ImageCacheType": "auto"
        },
        {
            "ImageCacheId": "imc-bbbbbb",
            "ImageCacheName": "<ImageCacheName>",
            "Status": "Ready",
            "Images": ["nginx:1.27-alpine"],
            "ImageCacheType": "manual"
        }
    ],
    "RequestId": "..."
}
```

#### 步骤 B3：查询最适合的镜像缓存

在创建 Pod 前，可查询是否存在已包含目标镜像的缓存，避免重复创建。

##### 命中场景

```bash
tccli tke GetMostSuitableImageCache \
    --Images '["nginx:1.27-alpine"]' \
    --region <Region>
# expected: exit 0，Found=true 且返回匹配的 ImageCache
```

**预期输出**：

```json
{
    "Found": true,
    "ImageCache": {
        "ImageCacheId": "imc-xxxxxx",
        "ImageCacheName": "<ImageCacheName>",
        "ImageCacheSize": 20,
        "Images": ["nginx:1.27-alpine"],
        "Status": "Ready",
        "SnapshotId": "snap-xxxxxx",
        "ImageCacheType": "manual",
        "Snapshotter": "overlay"
    },
    "RequestId": "..."
}
```

##### 未命中场景

```bash
tccli tke GetMostSuitableImageCache \
    --Images '["nonexistent:latest"]' \
    --region <Region>
# expected: exit 0，Found=false
```

**预期输出**：

```json
{
    "Found": false,
    "ImageCache": null,
    "RequestId": "..."
}
```

#### 步骤 B4：更新镜像缓存（权限受限）

> **说明**：当前账号的 CAM 策略未授予 `tke:UpdateImageCache` 权限，即使资源带有 `billing` 标签也无法绕过。以下仅为命令格式和预期输出说明，实际执行将返回 `UnauthorizedOperation.CamNoAuth`。如需执行更新操作，请联系 CAM 管理员在策略中增加 `tke:UpdateImageCache` 权限。

**命令格式**：

```bash
tccli tke UpdateImageCache \
    --ImageCacheId <ImageCacheId> \
    --ImageCacheName <NewName> \
    --RetentionDays 3 \
    --region <Region>
```

**预期输出**（正常情况）：

```json
{
    "RequestId": "..."
}
```

**实际错误**（当前 CAM 策略阻止）：

```
UnauthorizedOperation.CamNoAuth: you are not authorized to perform
operation (tke:UpdateImageCache)
resource (qcs::tke:ap-guangzhou:uin/...:imagecache/<ImageCacheId>)
has no permission
```

#### 步骤 B5：删除镜像缓存（权限受限）

> **说明**：当前账号的 CAM 策略未授予 `tke:DeleteImageCaches` 权限。以下仅为命令格式和预期输出说明。如需清理镜像缓存，请通过控制台手动删除或联系 CAM 管理员增加 `tke:DeleteImageCaches` 权限。

**命令格式**：

```bash
tccli tke DeleteImageCaches \
    --ImageCacheIds '["<ImageCacheId>"]' \
    --region <Region>
```

**预期输出**（正常情况）：

```json
{
    "RequestId": "..."
}
```

**实际错误**（当前 CAM 策略阻止）：

```
UnauthorizedOperation.CamNoAuth: you are not authorized to perform
operation (tke:DeleteImageCaches)
resource (qcs::tke:ap-guangzhou:uin/...:imagecache/<ImageCacheId>)
has no permission
```

## 验证

### 控制面（tccli）

```bash
# 查看镜像缓存列表，确认 Status 为 Ready
tccli tke DescribeImageCaches --region <Region> --Limit 10 --Offset 0
# expected: exit 0，ImageCaches 中 Status 为 Ready 的资源可用于 Pod 加速
```

```json
{
    "TotalCount": "<TotalCount>",
    "ImageCaches": [
        {
            "ImageCacheId": "imc-xxxxxx",
            "Status": "Ready",
            "ImageCacheType": "<manual|auto>"
        }
    ],
    "RequestId": "..."
}
```

### 数据面（kubectl）

```bash
# 验证 Pod 状态和缓存 Annotation
kubectl --kubeconfig ~/.kube/config-super describe pod <PodName>
# expected: Annotations 包含 cbs-reuse-key
```

**预期输出**：

```text
Annotations:  eks.tke.cloud.tencent.com/cbs-reuse-key: <CacheKey>
              eks.tke.cloud.tencent.com/cpu: 1
              eks.tke.cloud.tencent.com/mem: 2Gi
Status:       Running
```

```bash
# 查看启动耗时
kubectl --kubeconfig ~/.kube/config-super get pod <PodName> -o json \
    | jq '.status.conditions[] | select(.type=="Ready")'
# expected: 首个 Pod 启动时间较长（含镜像拉取），后续 Pod 显著缩短
```

**预期输出**：

```json
{
  "type": "Ready",
  "status": "True",
  "lastProbeTime": null,
  "lastTransitionTime": "<Timestamp>"
}
```

| 验证维度 | 命令 | 预期 |
|---------|------|------|
| 节点池就绪 | `DescribeClusterVirtualNodePools --ClusterId <ClusterId>` | `LifeState: "normal"` |
| 镜像缓存就绪 | `DescribeImageCaches --ImageCacheIds '["<ImageCacheId>"]'` | `Status: "Ready"` |
| 镜像缓存查询 | `GetMostSuitableImageCache --Images '["<image>"]'` | `Found: true` |
| Pod Running | `kubectl get pod <PodName>` | `STATUS: "Running"` |
| Annotation 生效 | `kubectl describe pod <PodName>` | Annotations 含 `cbs-reuse-key` |
| 缓存加速 | 对比首/次 Pod 启动耗时 | 后续 Pod 无 "Pulling image" 或时间显著缩短 |

## 清理

> **警告**：
> - 删除镜像缓存后关联的 CBS 快照会被释放，使用该快照的 EKSCI 实例将无法再享用缓存加速。
> - `RetentionDays=0`（永不过期）的镜像缓存不会自动清理，快照按月计费，务必在测试完成后删除。
> - 如果 `DeleteImageCaches` 被 CAM 拒绝，需通过控制台或联系管理员手动清理。
> - 删除带有镜像缓存 Annotation 的 Pod 不会清除底层 CBS 缓存数据。缓存数据由超级节点后端管理，不产生额外费用。如需彻底清理缓存，删除所有使用该缓存 key 的 Pod 后，等待系统自动回收（通常数小时内）。

### kubectl 清理

```bash
# 删除测试 Pod
kubectl --kubeconfig ~/.kube/config-super delete pod <PodName>
# expected: pod "<PodName>" deleted

# 如有第二个测试 Pod，一并删除
kubectl --kubeconfig ~/.kube/config-super delete pod <PodName2>
# expected: pod "<PodName2>" deleted

# 验证 Pod 已删除
kubectl --kubeconfig ~/.kube/config-super get pod <PodName>
# expected: Error from server (NotFound): pods "<PodName>" not found
```

**预期输出**：

```text
Error from server (NotFound): pods "<PodName>" not found
```

### tccli 清理（如权限允许）

```bash
# 删除镜像缓存（需 CAM 权限，当前账号可能被拒绝）
tccli tke DeleteImageCaches \
    --ImageCacheIds '["<ImageCacheId>"]' \
    --region <Region>
# expected（权限允许时）: exit 0，返回 RequestId
```

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| Pod 创建失败，Events 显示 `Failed to create pod sandbox` | `kubectl describe pod <PodName> \| tail -20` | 镜像拉取失败：镜像名/标签错误，或镜像仓库不可达 | 确认镜像名和标签正确；检查镜像仓库网络可达 |
| `kubectl get nodes -l node.kubernetes.io/instance-type=eklet` 无节点 | `tccli tke DescribeClusterVirtualNodePools --region <Region> --ClusterId <ClusterId>` | 超级节点池尚未创建 | 先执行 [新建超级节点](../新建超级节点/tccli%20操作.md) |
| `CreateImageCache` 失败 | 查看返回的错误码 | `VpcId` 或 `SubnetId` 填写错误 | `DescribeClusters` 查询集群详情获取正确的 VpcId 和 SubnetId |
| `CreateImageCache` 返回成功但 Status 长时间未 Ready | `DescribeImageCaches --ImageCacheIds` 查看 Events 字段 | EKSCI 实例无外网访问能力 | 创建时必须加 `--AutoCreateEip true` 或 `--ExistedEipId` |
| `UpdateImageCache` 返回 `UnauthorizedOperation.CamNoAuth` | — | CAM 策略未授予 `tke:UpdateImageCache` 权限 | 联系 CAM 管理员在策略中增加 `tke:UpdateImageCache` 权限 |
| `DeleteImageCaches` 返回 `UnauthorizedOperation.CamNoAuth` | — | CAM 策略未授予 `tke:DeleteImageCaches` 权限 | 联系 CAM 管理员增加 `tke:DeleteImageCaches` 权限，或通过控制台手动删除 |

### 创建成功但缓存未生效

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| 后续 Pod 仍需重新拉取镜像 | `kubectl describe pod <PodName2> \| grep -A10 Events` 查看是否有 `Pulling image` | `cbs-reuse-key` Annotation key 拼写错误或值不一致 | 核对 Annotation Key 为 `eks.tke.cloud.tencent.com/cbs-reuse-key`（全小写，中划线分隔），且两个 Pod 使用相同 value |
| 首个 Pod 使用新 key 仍然慢 | — | 此为目标行为：新 key 的首次使用须拉取完整镜像 | 无。首个 Pod 拉取镜像后，缓存才可供后续 Pod 复用 |
| 不同镜像使用相同 key | `kubectl describe pod <PodName>` 查看镜像名 | 缓存 key 对应特定镜像层，不同镜像混用同一 key 会导致缓存不命中 | 为不同镜像使用不同的缓存 key |
| 镜像更新后缓存命中旧版本 | `kubectl describe pod <PodName2> \| grep Image` 查看实际镜像 | 缓存 key 未变更，复用旧的镜像层 | 更新镜像后变更缓存 key（如加版本后缀），强制重新拉取 |
| `RetentionDays=0` 导致快照费用持续产生 | — | 永不过期策略，快照按月计费 | 生产环境设置 `--RetentionDays` 为 7 或合理值 |

## 下一步

- [超级节点 Annotation 说明](https://cloud.tencent.com/document/product/457/44173) — 完整 Annotation 列表
- [指定资源规格](https://cloud.tencent.com/document/product/457/44174) — CPU/内存/系统盘规格配置
- [超级节点可调度 Pod 说明](https://cloud.tencent.com/document/product/457/74015) — 可调度规格范围
- [调度 Pod 至超级节点](https://cloud.tencent.com/document/product/457/74016) — 调度策略配置

## 控制台替代

[TKE 控制台 → 工作负载 → 新建 → 高级设置 → 镜像缓存](https://console.cloud.tencent.com/tke2/cluster)：在控制台表单中创建镜像缓存资源，或为 Pod 填写镜像缓存复用键。
