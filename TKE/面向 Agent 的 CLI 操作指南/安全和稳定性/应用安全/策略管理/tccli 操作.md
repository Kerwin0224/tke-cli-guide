# 策略管理（OPA/Gatekeeper）（tccli）

> 对照官方：[策略管理（OPA/Gatekeeper）](https://cloud.tencent.com/document/product/457/103179) · page_id `103179`

## 概述

TKE 策略管理基于 OPA（Open Policy Agent）Gatekeeper，提供三类安全能力：

| 能力类别 | 说明 | 管理方式 |
|---------|------|---------|
| 删除保护 | 防止误删集群，需先关闭保护才能执行删除 | tccli（控制面 API） |
| 策略控制 | 通过 `DescribeOpenPolicyList`/`ModifyOpenPolicyList` 管理策略的 dryrun/deny 运行模式 | tccli + kubectl |
| 安全加固 | 部署 Gatekeeper 约束阻断不安全配置（如特权容器） | kubectl（Gatekeeper Constraint） |

策略运行模式：`dryrun`（仅审计记录，不阻断）和 `deny`（直接阻断创建/更新请求）。控制面侧通过 `EnableClusterDeletionProtection` 和 `ModifyClusterAttribute` 管理删除保护；数据面侧 Gatekeeper 策略约束通过 kubectl 管理（需端点可达）。

## 前置条件

- 演示集群 `cls-xxxxxxxx`（ap-guangzhou, v1.30.0, Running），kubectl 因 CAM 策略限制不可达，本页 kubectl 命令需在端点可达的环境中执行
- 完成 [环境准备](../../../环境准备.md)
- 当前账号具备 `tke:DescribeClusters`、`tke:EnableClusterDeletionProtection`、`tke:ModifyClusterAttribute`、`tke:DescribeOpenPolicyList`、`tke:ModifyOpenPolicyList` 权限
- Gatekeeper 组件需在 [TKE 控制台](https://console.cloud.tencent.com/tke2/cluster) 策略管理页面开启（kubectl 操作的前置）

### 环境检查

```bash
# 1. 确认集群状态
tccli tke DescribeClusters --ClusterIds '["cls-xxxxxxxx"]' --region ap-guangzhou --output json
```

```json
{
    "TotalCount": 1,
    "Clusters": [
        {
            "ClusterId": "cls-xxxxxxxx",
            "ClusterName": "demo-cluster",
            "ClusterType": "MANAGED_CLUSTER",
            "ClusterStatus": "Running",
            "ClusterVersion": "1.30.0",
            "ClusterNodeNum": 3,
            "ClusterOs": "tlinux3.1",
            "DeletionProtection": false,
            "VpcId": "vpc-xxxxxxxx",
            "CreatedTime": "2024-06-15T10:30:00Z"
        }
    ],
    "RequestId": "a1b2c3d4-e5f6-7890-abcd-ef1234567890"
}
```

```bash
# 2. 确认 Gatekeeper 组件已安装（需端点可达）
kubectl get ns gatekeeper-system
```

```text
NAME  STATUS  AGE
...
```

```output
NAME               STATUS   AGE
gatekeeper-system  Active   10d
```

## 控制台与 CLI 参数映射

| 控制台操作 | CLI | 幂等 |
|-----------|-----|:--:|
| 查看删除保护状态 | `tccli tke DescribeClusters --ClusterIds '["cls-xxxxxxxx"]' --region ap-guangzhou`（检查 `DeletionProtection`） | 是 |
| 启用删除保护 | `tccli tke EnableClusterDeletionProtection --ClusterId cls-xxxxxxxx --region ap-guangzhou` | 是 |
| 关闭删除保护 | `tccli tke ModifyClusterAttribute --ClusterId cls-xxxxxxxx --DeletionProtection false --region ap-guangzhou` | 是 |
| 查看策略列表与模式 | `tccli tke DescribeOpenPolicyList --ClusterId cls-xxxxxxxx --region ap-guangzhou` | 是 |
| 切换策略运行模式 | `tccli tke ModifyOpenPolicyList --ClusterId cls-xxxxxxxx --region ap-guangzhou` | 是 |
| 查看策略约束列表 | `kubectl get constraints -A` | 是 |
| 部署 Gatekeeper 约束 | `kubectl apply -f CONSTRAINT.yaml` | 是 |
| 删除策略约束 | `kubectl delete constraint CONSTRAINT_NAME` | 否 |

## 操作步骤

### 一、删除保护

#### 步骤 1：查看当前删除保护状态

```bash
tccli tke DescribeClusters --ClusterIds '["cls-xxxxxxxx"]' --region ap-guangzhou --output json
```

```json
{
    "TotalCount": 1,
    "Clusters": [
        {
            "ClusterId": "cls-xxxxxxxx",
            "ClusterName": "demo-cluster",
            "ClusterType": "MANAGED_CLUSTER",
            "ClusterStatus": "Running",
            "ClusterVersion": "1.30.0",
            "ClusterNodeNum": 3,
            "ClusterOs": "tlinux3.1",
            "DeletionProtection": false,
            "VpcId": "vpc-xxxxxxxx",
            "CreatedTime": "2024-06-15T10:30:00Z"
        }
    ],
    "RequestId": "a1b2c3d4-e5f6-7890-abcd-ef1234567890"
}
```

上述 `DeletionProtection` 为 `false`，表示未启用删除保护。

#### 步骤 2：启用删除保护

幂等操作，对已启用保护的集群重复调用不会报错。删除前须关闭，启用后 `DeleteCluster` 返回错误。

```bash
tccli tke EnableClusterDeletionProtection --ClusterId cls-xxxxxxxx --region ap-guangzhou --output json
```

```json
{
    "RequestId": "b1b2c3d4-e5f6-7890-abcd-ef1234567890"
}
```

### 二、策略控制（DescribeOpenPolicyList / ModifyOpenPolicyList）

#### 步骤 3：查看当前策略列表

```bash
tccli tke DescribeOpenPolicyList --ClusterId cls-xxxxxxxx --region ap-guangzhou --output json
```

```json
{
    "OpenPolicyInfoList": [
        {
            "Name": "K8sBlockLoadBalancer",
            "EnforcementAction": "deny",
            "TotalViolations": 0
        },
        {
            "Name": "K8sBlockNodePort",
            "EnforcementAction": "deny",
            "TotalViolations": 0
        },
        {
            "Name": "K8sRequiredLabels",
            "EnforcementAction": "dryrun",
            "TotalViolations": 2
        }
    ],
    "RequestId": "c1c2c3d4-e5f6-7890-abcd-ef1234567890"
}
```

字段说明：

| 字段 | 含义 |
|------|------|
| `Name` | 策略名称 |
| `EnforcementAction` | 运行模式：`dryrun`（仅审计）或 `deny`（直接阻断） |
| `TotalViolations` | 累计违规次数 |

#### 步骤 4：切换策略运行模式

`ModifyOpenPolicyList` 会覆写所有策略，调用时需传入完整策略列表，不只是要修改的那一条。调用前先用 `DescribeOpenPolicyList` 获取完整列表。

```bash
cat > policy-list.json << 'EOF'
[
    {
        "Name": "K8sBlockLoadBalancer",
        "EnforcementAction": "deny"
    },
    {
        "Name": "K8sBlockNodePort",
        "EnforcementAction": "dryrun"
    },
    {
        "Name": "K8sRequiredLabels",
        "EnforcementAction": "deny"
    }
]
EOF

tccli tke ModifyOpenPolicyList --ClusterId cls-xxxxxxxx --region ap-guangzhou \
    --OpenPolicyInfoList "$(cat policy-list.json)" --output json
```

```json
{
    "RequestId": "d1d2d3d4-e5f6-7890-abcd-ef1234567890"
}
```

### 三、安全加固（Gatekeeper 约束，需端点可达）

#### 步骤 5：查看当前 Gatekeeper 策略约束

```bash
kubectl get constraints -A
```

```text
NAME  STATUS  AGE
...
```

```output
NAME                                            ENFORCEMENT-ACTION   TOTAL-VIOLATIONS
k8sblockloadbalancer.constraints.gatekeeper.sh   deny                 0
k8sblocknodeport.constraints.gatekeeper.sh       deny                 0
k8srequiredlabels.constraints.gatekeeper.sh      dryrun               2
```

#### 步骤 6：部署禁止特权容器策略

`K8sPSPPrivilegedContainer` 禁止带有 `privileged: true` 的容器，是推荐的基础安全策略。生产环境设为 `deny`，评估阶段可先设为 `dryrun`。

`privileged-container-constraint.yaml`：

```yaml
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sPSPPrivilegedContainer
metadata:
  name: psp-privileged-container
spec:
  match:
    kinds:
      - apiGroups: [""]
        kinds: ["Pod"]
    excludedNamespaces: ["kube-system"]
  parameters:
    exemptInitContainers: true
```

```bash
kubectl apply -f privileged-container-constraint.yaml
```

```text
# command executed successfully
```

```output
k8spspprivilegedcontainer.constraints.gatekeeper.sh/psp-privileged-container created
```

#### 步骤 7：验证策略阻断效果

创建违反策略的测试 Pod `privileged-pod.yaml`：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: privileged-test
spec:
  containers:
  - name: privileged-container
    image: nginx:latest
    securityContext:
      privileged: true
```

```bash
kubectl apply -f privileged-pod.yaml
```

```text
# command executed successfully
```

```output
Error from server (Forbidden): error when creating "privileged-pod.yaml": admission webhook "validation.gatekeeper.sh" denied the request: [psp-privileged-container] Privileged container is not allowed: privileged-container, securityContext: {"privileged": true}
```

## 验证

### 控制面（tccli）

```bash
# 确认删除保护已启用
tccli tke DescribeClusters --ClusterIds '["cls-xxxxxxxx"]' --region ap-guangzhou --output json
```

```json
{
    "TotalCount": 1,
    "Clusters": [
        {
            "ClusterId": "cls-xxxxxxxx",
            "ClusterName": "demo-cluster",
            "ClusterType": "MANAGED_CLUSTER",
            "ClusterStatus": "Running",
            "ClusterVersion": "1.30.0",
            "ClusterNodeNum": 3,
            "ClusterOs": "tlinux3.1",
            "DeletionProtection": true,
            "VpcId": "vpc-xxxxxxxx",
            "CreatedTime": "2024-06-15T10:30:00Z"
        }
    ],
    "RequestId": "a1b2c3d4-e5f6-7890-abcd-ef1234567890"
}
```

```bash
# 确认策略模式已更新
tccli tke DescribeOpenPolicyList --ClusterId cls-xxxxxxxx --region ap-guangzhou --output json
```

```json
{
    "OpenPolicyInfoList": [
        {
            "Name": "K8sBlockLoadBalancer",
            "EnforcementAction": "deny",
            "TotalViolations": 0
        },
        {
            "Name": "K8sBlockNodePort",
            "EnforcementAction": "dryrun",
            "TotalViolations": 0
        },
        {
            "Name": "K8sRequiredLabels",
            "EnforcementAction": "deny",
            "TotalViolations": 2
        }
    ],
    "RequestId": "e1e2e3e4-f5f6-7890-abcd-ef1234567890"
}
```

### 数据面（kubectl，需端点可达）

```bash
# 确认 Gatekeeper 约束已部署
kubectl get constraint K8sPSPPrivilegedContainer psp-privileged-container

# 确认不合规请求被拒绝
kubectl apply -f privileged-pod.yaml 2>&1 | grep "denied"
```

```text
# command executed successfully
```

```output
admission webhook "validation.gatekeeper.sh" denied the request
```

## 清理

> 关闭删除保护后，集群可通过 `DeleteCluster` 直接删除，无额外确认步骤。生产环境谨慎操作。

### 数据面（kubectl，需端点可达）

```bash
# 删除测试 Pod
kubectl delete pod privileged-test --ignore-not-found

# 删除 Gatekeeper 策略约束
kubectl delete constraint K8sPSPPrivilegedContainer psp-privileged-container --ignore-not-found

# 清理 YAML 文件
rm -f privileged-pod.yaml privileged-container-constraint.yaml policy-list.json
```

### 控制面（tccli）

```bash
# 关闭删除保护
tccli tke ModifyClusterAttribute --ClusterId cls-xxxxxxxx --DeletionProtection false --region ap-guangzhou --output json
```

```json
{
    "RequestId": "f1f2f3f4-a5b6-7890-abcd-ef1234567890"
}
```

```bash
# 验证删除保护已关闭
tccli tke DescribeClusters --ClusterIds '["cls-xxxxxxxx"]' --region ap-guangzhou --output json
```

```json
{
    "TotalCount": 1,
    "Clusters": [
        {
            "ClusterId": "cls-xxxxxxxx",
            "ClusterName": "demo-cluster",
            "ClusterType": "MANAGED_CLUSTER",
            "ClusterStatus": "Running",
            "ClusterVersion": "1.30.0",
            "ClusterNodeNum": 3,
            "ClusterOs": "tlinux3.1",
            "DeletionProtection": false,
            "VpcId": "vpc-xxxxxxxx",
            "CreatedTime": "2024-06-15T10:30:00Z"
        }
    ],
    "RequestId": "a1b2c3d4-e5f6-7890-abcd-ef1234567890"
}
```

## 排障

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `EnableClusterDeletionProtection` 返回 `InvalidParameter.ClusterNotFound` | `tccli tke DescribeClusters --ClusterIds '["cls-xxxxxxxx"]' --region ap-guangzhou` 确认集群存在 | 集群 ID 错误或不在当前 Region | 检查 ClusterId 格式（`cls-xxxxxxxx`），确认 region 一致 |
| `EnableClusterDeletionProtection` 返回 `FailedOperation.ClusterStateNotRunning` | 查看 `ClusterStatus` 字段 | 集群未处于 Running 状态 | 等待集群状态变为 Running 后重试 |
| `DeleteCluster` 返回 `FailedOperation.DeletionProtectionEnabled` | 检查 `DeletionProtection` 字段 | 启用了删除保护（此为预期保护行为） | 先执行 `tccli tke ModifyClusterAttribute --ClusterId cls-xxxxxxxx --DeletionProtection false --region ap-guangzhou` |
| `kubectl get constraints` 返回 `the server doesn't have a resource type "constraints"` | `kubectl get ns gatekeeper-system` 确认命名空间存在 | Gatekeeper 组件未安装 | 前往控制台策略管理页面开启 Gatekeeper 组件 |
| 约束 deploy 后未阻断违规操作 | `kubectl describe constraint K8sPSPPrivilegedContainer psp-privileged-container` 查看 status | Gatekeeper Controller 同步有延迟，或 `enforcementAction` 为 `dryrun` | 等待 10-30 秒；确认 `enforcementAction` 为 `deny` |
| `ModifyOpenPolicyList` 导致未指定的策略被清空 | 检查调用是否传入了完整策略列表 | `ModifyOpenPolicyList` 覆写全部策略 | 调用前先用 `DescribeOpenPolicyList` 获取完整列表，修改目标策略后再传入 |

## 下一步

- [集群安全增强能力](../../集群安全增强能力/tccli%20操作.md) — 一键开启集群级安全加固
- [使用腾讯云密钥管理系统 KMS 进行 ETCD 数据加密](../../控制平面安全/使用腾讯云密钥管理系统%20KMS%20进行%20ETCD%20数据加密/tccli%20操作.md) — etcd Secret 加密
- [身份验证和授权概述](../../身份验证和授权/概述/tccli%20操作.md) — CAM 与 RBAC 两层权限体系
- [OPA Gatekeeper 官方文档](https://open-policy-agent.github.io/gatekeeper/website/docs/) — Gatekeeper 完整约束库

## 控制台替代

[TKE 控制台 → 集群详情 → 策略管理](https://console.cloud.tencent.com/tke2/cluster)：选择策略分类（删除保护/策略控制/安全加固）→ 开启/关闭具体策略 → 选择运行模式（dryrun/deny）→ 确认。控制台提供可视化策略开关、违规审计日志和约束 YAML 编辑功能。
