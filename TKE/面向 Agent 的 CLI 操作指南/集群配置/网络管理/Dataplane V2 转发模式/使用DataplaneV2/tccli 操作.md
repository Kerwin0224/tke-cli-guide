# 使用DataplaneV2
> 对照官方：[使用DataplaneV2](https://cloud.tencent.com/document/product/457/113228) · page_id `113228`

## 概述

DataplaneV2 是 TKE 集群的替代转发模式，启用后使用 eBPF 替代传统 kube-proxy 组件进行 Service 流量转发，消除 iptables 规则膨胀带来的性能损耗。该模式仅在**新建集群**时可选，存量集群不支持切换。

核心特性：
- 替代 kube-proxy，集群中不再安装 kube-proxy DaemonSet
- 仅支持 VPC-CNI 共享网卡多 IP 网络模式
- 仅支持 TencentOS Server 3.1/3.2 (TK4) 操作系统
- 要求 Kubernetes >= v1.24
- 默认最大 500 节点，超出需提工单申请
- 暂不支持 NodeLocalDNSCache

## 前置条件

| 条件 | 验证命令 | 预期结果 |
|------|---------|---------|
| tccli 已安装并配置凭证 | `tccli tke DescribeClusters --region ap-guangzhou --ClusterIds '["<ClusterId>"]'` | 返回集群信息，无鉴权错误 |
| 目标 K8s 版本 >= v1.24 | `tccli tke DescribeClusters --region ap-guangzhou --ClusterIds '["<ClusterId>"]'` | ClusterVersion 字段 >= 1.24.0 |
| 使用 VPC-CNI 网络模式 | `tccli tke DescribeClusters --region ap-guangzhou --ClusterIds '["<ClusterId>"]' --output json` | ClusterNetworkSettings.Cni 为 true |
| 操作系统为 TencentOS Server 3.1/3.2 | 控制台节点列表查看 OS 镜像，或 `tccli tke DescribeClusterInstances --region ap-guangzhou --ClusterId <ClusterId>` | OS 字段为 TencentOS Server 3.1 或 3.2 |
| 已配置 kubectl（如需数据面验证） | `kubectl cluster-info` | Kubernetes control plane 可达 |

> **注意**：DataplaneV2 **仅新建集群时可开启**。若存量集群未启用，需创建新集群并指定参数。

## 控制台与 CLI 参数映射

| 控制台操作 | CLI | 幂等 |
|-----------|-----|------|
| 新建集群时开启 DataplaneV2 | `tccli tke CreateCluster --cli-input-json file://create-cluster-dpv2.json`（ClusterNetworkSettings.DataPlaneV2=true） | 否（新建资源，每次创建新集群） |
| 查看集群是否启用 DataplaneV2 | `tccli tke DescribeClusters --ClusterIds '["<ClusterId>"]'` | 是 |
| 查看节点操作系统 | `tccli tke DescribeClusterInstances --ClusterId <ClusterId>` | 是 |
| 确认 kube-proxy 已不存在 | `kubectl get ds -n kube-system`（数据面） | 是 |

## 操作步骤

### 1. 准备集群创建参数

创建 JSON 参数文件 `create-cluster-dpv2.json`：

```json
{
    "ClusterName": "cls-example",
    "ClusterType": "MANAGED_CLUSTER",
    "ClusterVersion": "1.30.0",
    "VpcId": "<VpcId>",
    "ProjectId": 0,
    "ClusterNetworkSettings": {
        "ClusterCIDR": "10.168.0.0/16",
        "ServiceCIDR": "10.168.252.0/22",
        "Cni": true,
        "Subnets": ["<SubnetId>"],
        "DataPlaneV2": true
    },
    "ClusterBasicSettings": {
        "OsName": "TencentOS Server 3.1 (TK4)"
    },
    "RunInstancesForNode": [
        {
            "NodeRole": "WORKER",
            "RunInstancesPara": [
                "{\"InstanceType\":\"S5.MEDIUM4\",\"ImageId\":\"img-eb30mz89\",\"SystemDisk\":{\"DiskType\":\"CLOUD_PREMIUM\",\"DiskSize\":50}}"
            ],
            "InstanceAdvancedSettings": {
                "SubnetId": "<SubnetId>"
            }
        }
    ]
}
```

> **字段说明**：`DataPlaneV2: true` 为核心开关。`Cni: true` 为 DataplaneV2 的前置要求。`OsName` 须为 TencentOS Server 3.1 (TK4) 或 3.2。

### 2. 创建集群

```bash
tccli tke CreateCluster --region ap-guangzhou --cli-input-json file://create-cluster-dpv2.json
```

```output
{
    "ClusterId": "<ClusterId>",
    "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

> 集群创建为异步操作，约需 5-10 分钟。记录返回的 `ClusterId`。

### 3. 等待集群就绪

```bash
tccli tke DescribeClusters --region ap-guangzhou --ClusterIds '["<ClusterId>"]' --output json
```

```json
{
  "TotalCount": 0,
  "Clusters": [],
  "ClusterId": "<ClusterId>",
  "ClusterName": "<ClusterName>",
  "ClusterDescription": "<ClusterDescription>",
  "ClusterVersion": "<ClusterVersion>",
  "ClusterOs": "<ClusterOs>",
  "ClusterType": "<ClusterType>"
}
```

```output
{
    "TotalCount": 1,
    "Clusters": [
        {
            "ClusterId": "<ClusterId>",
            "ClusterName": "cls-example",
            "ClusterStatus": "Running",
            "ClusterVersion": "1.30.0",
            "ClusterNetworkSettings": {
                "ClusterCIDR": "10.168.0.0/16",
                "VpcId": "<VpcId>",
                "Cni": true,
                "Ipvs": false,
                "ServiceCIDR": "10.168.252.0/22",
                "Subnets": ["<SubnetId>"],
                "DataPlaneV2": true
            }
        }
    ],
    "RequestId": "xxxxx

```json
{
  "TotalCount": 0,
  "Clusters": [],
  "ClusterId": "<ClusterId>",
  "ClusterName": "<ClusterName>",
  "ClusterDescription": "<ClusterDescription>",
  "ClusterVersion": "<ClusterVersion>",
  "ClusterOs": "<ClusterOs>",
  "ClusterType": "<ClusterType>"
}
```xxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

> 确认 `ClusterStatus` 为 `Running`，`ClusterNetworkSettings.DataPlaneV2` 为 `true`。

## 验证

### 控制面（tccli）

验证 DataplaneV2 状态：

```bash
tccli tke DescribeClusters --region ap-guangzhou --ClusterIds '["<ClusterId>"]' --output json
```

确认输出中 `ClusterNetworkSettings.DataPlaneV2` 为 `true`，`Cni` 为 `true`。

验证集群运行状态：

```bash
tccli tke DescribeClusterStatus --region ap-guangzhou --ClusterIds '["<ClusterId>"]'
```

```output
{
    "ClusterStatusSet": [
        {
            "ClusterId": "<ClusterId>",
            "ClusterState": "Running"
        }
    ],
    "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

### 数据面（kubectl）

> **注意**：公网端点创建受 CAM 策略限制（tke:clusterExtranetEndpoint 被 deny）。kubectl 命令需在 VPC 内 CVM 执行，或通过 VPN/专线连接集群内网端点。

确认 kube-proxy DaemonSet 不存在：

```bash
kubectl get daemonset -n kube-system --no-headers
```

```output
# 输出中不应包含 kube-proxy 相关条目
# 预期的 kube-system DaemonSet 仅包含：
#   tke-cni-agent
#   tke-monitor-agent
#   tke-log-agent
```

显式查询 kube-proxy，预期返回 No resources found：

```bash
kubectl get daemonset kube-proxy -n kube-system
```

```output
Error from server (NotFound): daemonsets.apps "kube-proxy" not found
```

验证 Service 流量转发正常（部署测试 Pod 和 Service）：

```bash
kubectl create deployment nginx-test --image=nginx:alpine -n default
kubectl expose deployment nginx-test --port=80 --target-port=80 -n default
kubectl run curl-test --image=curlimages/curl:latest -it --rm --restart=Never -- \
  curl -s -o /dev/null -w "%{http_code}" nginx-test.default.svc.cluster.local
```

```output
200
```

清理测试资源：

```bash
kubectl delete deployment nginx-test -n default
kubectl delete service nginx-test -n default
```

## 清理

### 控制面（tccli）

> **计费警告**：删除集群将释放所有节点（CVM 实例），停止计费。集群内数据（PVC、日志等）将永久丢失，不可恢复。

> **副作用警告**：集群删除不可逆。确认无生产业务在运行后再执行。

```bash
tccli tke DeleteCluster --region ap-guangzhou --ClusterId <ClusterId> --InstanceDeleteMode terminate
```

```output
{
    "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

验证集群已删除：

```bash
tccli tke DescribeClusters --region ap-guangzhou --ClusterIds '["<ClusterId>"]'
```

```output
{
    "TotalCount": 0,
    "Clusters": [],
    "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

### 数据面（kubectl）

kubectl 测试资源已在验证阶段清理。无额外数据面资源需要处理。

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `CreateCluster` 返回 `InvalidParameter.ClusterCIDRConflict` | 检查 CIDR 是否与 VPC 内其他 CIDR 冲突 | ClusterCIDR 或 ServiceCIDR 与已有网络段重叠 | 使用不重叠的 CIDR 段，如 `10.168.0.0/16` 和 `10.168.252.0/22` |
| `CreateCluster` 返回 `InvalidParameter.OsNotSupport` | 检查 OsName 参数值 | 指定了不支持的 OS（如 CentOS、Ubuntu） | 将 `OsName` 改为 `TencentOS Server 3.1 (TK4)` 或 `TencentOS Server 3.2` |
| `CreateCluster` 返回 `FailedOperation.CniNotEnabled` | 检查 Cni 参数 | DataplaneV2 要求 VPC-CNI 已启用 | 确保 `ClusterNetworkSettings.Cni` 为 `true` |
| `CreateCluster` 返回 `InvalidParameter.K8sVersionNotSupport` | 检查 ClusterVersion 参数 | K8s 版本低于 1.24 | 将 `ClusterVersion` 设为 `1.24.0` 或更高 |
| `CreateCluster` 返回 `LimitExceeded.ClusterQuota` | 检查当前集群数量 | 超出集群配额上限 | 提交工单提升配额，或删除无用集群后重试 |
| `DescribeClusters` 返回空结果 | 检查 --ClusterIds 参数格式 | ClusterIds 需为 JSON 数组字符串 | 使用 `--ClusterIds '["<ClusterId>"]'` 格式（单引号包裹） |

### 创建成功但状态异常

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| ClusterStatus 长时间为 `Creating` | `DescribeClusters` 查看状态 | 节点初始化慢或资源不足 | 等待 10-15 分钟；超时后检查节点 CVM 是否正常创建 |
| 节点状态为 NotReady | `kubectl get nodes`（需数据面可达） | 节点 CNI 插件初始化失败或网络不通 | 通过控制台查看节点事件日志，确认 VPC-CNI 组件正常 |
| kube-proxy 仍出现在 DaemonSet 列表中 | `kubectl get ds -n kube-system`（需数据面可达） | DataplaneV2 未正确启用或集群为旧版本创建 | 确认 `ClusterNetworkSettings.DataPlaneV2` 为 `true`；若为 `false` 需重建集群 |
| 添加非 TencentOS 节点失败 | 控制台节点列表查看添加状态 | DataplaneV2 集群禁止添加非 TencentOS 节点 | 仅使用 TencentOS Server 3.1/3.2 镜像创建节点 |
| kubectl 无法连接（公网端点限制） | `kubectl cluster-info` 超时或拒绝连接 | 公网端点受 CAM 策略 deny | 在 VPC 内 CVM 执行 kubectl，或配置 VPN/专线连接内网端点 |

## 下一步

- [集群启用 IPVS](../../集群启用%20IPVS/tccli%20操作.md) -- IPVS 与 DataplaneV2 为互斥转发模式
- [容器集群网络方案选型](../../容器集群网络方案选型/tccli%20操作.md) -- 不同网络模式的对比与选择
- [容器集群网络规划](../../容器集群网络规划/tccli%20操作.md) -- 集群 CIDR 规划最佳实践
- [连接集群](../../../集群管理/连接集群/tccli%20操作.md) -- 配置 kubectl 访问集群

## 控制台替代

控制台路径：登录 [容器服务控制台](https://console.cloud.tencent.com/tke2/cluster/create) -> 新建集群 -> 网络配置 -> 高级设置 -> 开启 **DataplaneV2** 开关。

存量集群查看：控制台 -> 集群列表 -> 选择集群 -> 基本信息 -> 网络配置 -> 转发模式。
