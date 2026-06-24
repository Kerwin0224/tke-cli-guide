# 新增游离普通节点（tccli）

> 对照官方：[新增游离普通节点](https://cloud.tencent.com/document/product/457/32203) · page_id `32203`

## 概述

**游离普通节点**指不归属任何节点池、直接挂在集群下的 Worker 节点。适用场景：临时算力补充、特殊机型需求、或测试用途。官方提供两种方式：

1. **新建离散节点**：通过 `CreateClusterInstances` 按指定机型购买 CVM 并初始化加入集群。
2. **添加已有 CVM**：通过 `AddExistedInstances` 将现有 CVM 实例添加到集群（要求同 VPC、未被其他集群占用）。

两种方式的启动脚本目录均为 `/usr/local/qcloud/tke/userscript`，容器数据目录默认 `/var/lib/docker`（或 `/var/lib/containerd`）。

## 前置条件

- 集群状态为 `Running`，子网 IP 充足。
- [环境准备](../../../../环境准备.md)：`tccli` 已配置。
- 添加已有 CVM 要求：与集群**同 VPC**、未被其他集群绑定、安全组规则允许与集群通信。
- 建议长期节点使用节点池管理，游离节点适合临时需求。

## 控制台与 CLI 参数映射

| 控制台操作 | tccli API | 幂等 |
|------------|-----------|------|
| 新建离散普通节点 | `CreateClusterInstances`（`--cli-input-json` + `--waiter`） | 否 |
| 添加已有 CVM 到集群 | `AddExistedInstances` | 否 |
| 查询 CVM 配置 | `DescribeClusterInstances` | 是 |
| 主机名模式 | `InstanceAdvancedSettings` / 启动配置字段（自动：`tke_<集群id>_worker`） | 是 |
| 数据盘挂载 | `RunInstancePara` 中磁盘与挂载路径 | 否 |

## 操作步骤

---

### 场景 A：新建离散普通节点

以 `CreateClusterInstances` 创建 1 台新的按量 CVM 并自动初始化加入集群。`RunInstancePara` 为 **JSON 字符串**，含机型、网络、安全组、磁盘等 CVM 购买参数。

#### 准备 JSON 入参

```json
{
  "ClusterId": "<ClusterId>",
  "RunInstancePara": "{\"InstanceType\":\"S5.MEDIUM2\",\"InstanceChargeType\":\"POSTPAID_BY_HOUR\",\"InstanceCount\":1,\"Placement\":{\"Zone\":\"ap-guangzhou-3\",\"ProjectId\":0},\"SystemDisk\":{\"DiskType\":\"CLOUD_PREMIUM\",\"DiskSize\":50},\"SecurityGroupIds\":[\"<SecurityGroupId>\"],\"VirtualPrivateCloud\":{\"VpcId\":\"<VpcId>\",\"SubnetId\":\"<SubnetId>\"}}",
  "InstanceAdvancedSettings": {
    "Unschedulable": 0,
    "MountTarget": "/var/lib/containerd"
  }
}
```

| `RunInstancePara` 字段 | 必填 | 说明 |
|------------------------|------|------|
| `InstanceType` | 是 | CVM 机型，如 `S5.MEDIUM2` |
| `InstanceChargeType` | 是 | 计费类型：`POSTPAID_BY_HOUR`（按量）或 `PREPAID`（包年包月） |
| `InstanceCount` | 否 | 创建数量，默认 1 |
| `Placement.Zone` | 是 | 可用区，须与 `SubnetId` 所在 Zone 一致 |
| `SystemDisk` | 是 | 系统盘类型与大小 |
| `SecurityGroupIds` | 是 | 安全组 ID 数组 |
| `VirtualPrivateCloud` | 是 | VPC 与子网信息 |

#### 提交创建

```bash
tccli tke CreateClusterInstances --cli-input-json file://CreateClusterInstances.json --region ap-guangzhou --output json
```

```json
{ "InstanceIdSet": ["ins-example"], "RequestId": "a1b2c3d4-e5f6-7890-abcd-ef1234567890" }
```

#### 轮询节点状态

```bash
tccli tke DescribeClusterInstances --ClusterId <ClusterId> --region ap-guangzhou --output json \
  --filter "InstanceSet[?contains(InstanceId,'ins-example')].{InstanceId:InstanceId,InstanceState:InstanceState,InstanceRole:InstanceRole}"
```

```json
[
  { "InstanceId": "ins-example", "InstanceState": "running", "InstanceRole": "WORKER" }
]
```

# expected: exit 0, contains "running"

节点主机名默认遵循 `tke_<集群id>_worker` 规则。

---

### 场景 B：添加已有 CVM 到集群

将账号下现有 CVM 实例加入集群管理。前提条件：
- CVM 与集群**同 VPC**。
- CVM **未被其他集群占用**（一个 CVM 只能加入一个 TKE 集群）。
- CVM 的安全组规则允许与 Master 通信（10250 端口等）。

```bash
tccli tke AddExistedInstances --generate-cli-skeleton --output json > AddExistedInstances.json
```

编辑 JSON：

```json
{
  "ClusterId": "<ClusterId>",
  "InstanceIds": ["<InstanceId>"],
  "InstanceAdvancedSettings": {
    "Unschedulable": 0
  },
  "EnhancedService": {
    "SecurityService": { "Enabled": true },
    "MonitorService": { "Enabled": true }
  },
  "LoginSettings": {
    "Password": "<password>"
  },
  "SecurityGroupIds": ["<SecurityGroupId>"],
  "OsName": "TencentOS Server 3.1 (TK4)",
  "HostName": "tke-worker-free"
}
```

```bash
tccli tke AddExistedInstances --cli-input-json file://AddExistedInstances.json --region ap-guangzhou --output json
```

```json
{
  "FailedInstanceIds": [],
  "FailedReasons": [],
  "SuccInstanceIds": ["ins-example"],
  "TimeoutInstanceIds": [],
  "RequestId": "b2c3d4e5-f6a7-8901-bcde-f12345678901"
}
```

# expected: exit 0, "FailedInstanceIds" is empty

> 添加已有节点时，CVM 系统盘会被重装格式化，数据盘默认格式化为 `ext4` 并挂载到 `/var/lib/docker`（或 `/var/lib/containerd`）。若数据盘已有数据请提前备份。

---

### 节点初始化验证

节点加入集群后，数据面验证节点状态（需 APIServer 可达）：

```bash
kubectl get nodes -o wide
```

```text
NAME                STATUS   ROLES    AGE   VERSION   INTERNAL-IP
tke_cls-example_worker   Ready    <none>   2m    v1.32.2   172.24.0.10
```

控制面验证：

```bash
tccli tke DescribeClusterInstances --ClusterId <ClusterId> --region ap-guangzhou --output json \
  --filter "InstanceSet[?InstanceRole=='WORKER' && InstanceId=='<InstanceId>'].{InstanceId:InstanceId,InstanceState:InstanceState,FailedReason:FailedReason}"
```

```json
{
  "TotalCount": 0,
  "InstanceSet": [],
  "InstanceId": "<InstanceId>",
  "InstanceRole": "<InstanceRole>",
  "FailedReason": "<FailedReason>",
  "InstanceState": "<InstanceState>",
  "DrainStatus": "<DrainStatus>",
  "InstanceAdvancedSettings": "<InstanceAdvancedSettings>"
}
```

> **可达性要求**：kubectl 需 APIServer 端点可达。公网端点可能受 CAM 策略限制（如拒绝外网访问），内网端点需通过 IOA/VPN 连通 VPC。若 kubectl 不可达，`DescribeClusterInstances` 中 `InstanceState: running` 可确认节点已加入集群。

## 验证

### Control plane (tccli)

- `DescribeClusterInstances` 中包含新 `InstanceId`，`InstanceState: running`，`InstanceRole: WORKER`。
- `FailedReason` 为空或仅含 `=Ready:True`。

### Data plane (kubectl)

- `kubectl get nodes` 中节点 `Ready=True`（需 APIServer 可达）。

## 清理

游离节点删除（`InstanceDeleteMode: terminate` 会销毁 CVM）：

```bash
tccli tke DeleteClusterInstances --cli-input-json '{"ClusterId":"<ClusterId>","InstanceIds":["<InstanceId>"],"InstanceDeleteMode":"terminate"}' --region ap-guangzhou --output json
```

```json
{ "SuccInstanceIds": ["ins-example"], "RequestId": "c3d4e5f6-a7b8-9012-cdef-123456789012" }
```

`InstanceDeleteMode` 可选值：`terminate`（销毁 CVM）、`retain`（仅移出集群，保留 CVM）。

## 排障

| 现象 | 原因 | 处理 |
|------|------|------|
| 已有 CVM 添加失败 | CVM 不在集群 VPC 内，或已被其他集群绑定 | 确认 CVM 与集群同 VPC；如已加入其他集群，需先移出 |
| 节点初始化超时 | 安全组 10250 端口未放通、路由表不通 | 检查安全组规则（kubelet 10250 端口），确认子网路由表正确 |
| `CreateClusterInstances` 返回机型库存不足 | 选的机型在当前可用区售罄 | 换可用区或换机型（如 S5→SA2） |
| 新节点 `InstanceState` 长期为 `initializing` | 节点初始化脚本执行缓慢或镜像拉取失败 | 检查 VPC DNS；确认 `UserScript` 无长时间阻塞操作 |
| `AddExistedInstances` 返回 `FailedReasons` 含 CVM 状态错误 | CVM 不在运行中状态 | 确认 CVM `InstanceState: RUNNING` |

## 下一步

- [创建节点池](../创建节点池/tccli%20操作.md)：推荐长期使用节点池统一管理同质节点
- [移出节点](../../常用操作/移出节点/tccli%20操作.md)：将游离节点移出集群

## 控制台替代

控制台 → **集群** → 目标集群 → **节点管理** → **新建节点** → 选择**新建游离节点**或**添加已有节点**，按向导填写机型、网络、安全组后创建。
