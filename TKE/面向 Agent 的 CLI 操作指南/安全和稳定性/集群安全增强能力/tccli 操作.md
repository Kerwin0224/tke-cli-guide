# 集群安全增强能力（tccli）

> 对照官方：[集群安全增强能力](https://cloud.tencent.com/document/product/457/129420) · page_id `129420`

## 概述

TKE 集群安全增强能力是一键开启、全托管的安全加固方案，通过三层防护机制保护集群安全：**MetaServer 阻断**（通过 initContainer iptables 规则阻断对云元数据服务 `169.254.0.23` 的访问）、**准入控制**（基于 Gatekeeper Constraints 拦截不合规操作）、**网络隔离**（通过 NetworkPolicy 限制 Pod 间直接 IP 通信）。开启后 TKE 自动部署并持续调和所有安全资源，支持按命名空间或 Pod 标签灰度发布。

集群安全增强的开启/关闭为控制台独有操作，无对应 tccli API。开启后，可通过 tccli 查询集群状态，通过 kubectl 查看由安全增强管理的 Gatekeeper 约束、NetworkPolicy 和 Assign 资源（需端点可达）。

## 前置条件

- 演示集群 `cls-xxxxxxxx`（ap-guangzhou, v1.30.0, Running），kubectl 因 CAM 策略限制不可达，本页 kubectl 命令需在端点可达的环境中执行
- 完成 [环境准备](../../环境准备.md)
- 集群安全增强能力需在 [TKE 控制台](https://console.cloud.tencent.com/tke2/cluster) 开启（无对应 tccli API）
- 安全增强管理的资源统一携带标签 `app.kubernetes.io/managed-by: tke-security-mode`

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
            "VpcId": "vpc-xxxxxxxx",
            "CreatedTime": "2024-06-15T10:30:00Z"
        }
    ],
    "RequestId": "a1b2c3d4-e5f6-7890-abcd-ef1234567890"
}
```

## 控制台与 CLI 参数映射

| 控制台操作 | CLI | 幂等 |
|-----------|-----|:--:|
| 开启安全增强 | 控制台独有，无 tccli API | — |
| 关闭安全增强 | 控制台独有，无 tccli API | — |
| 查看集群状态 | `tccli tke DescribeClusters --ClusterIds '["cls-xxxxxxxx"]' --region ap-guangzhou` | 是 |
| 查看 Gatekeeper 约束 | `kubectl get constraints -A` | 是 |
| 查看 NetworkPolicy | `kubectl get networkpolicies -A` | 是 |
| 查看 Gatekeeper Assign（Mutation） | `kubectl get assign -A` | 是 |
| 查看安全增强管理资源 | `kubectl get all -A -l app.kubernetes.io/managed-by=tke-security-mode` | 是 |
| 查看 Pod initContainer 注入 | `kubectl describe pod <pod-name>` | 是 |

## 操作步骤

### 1. 确认安全增强已启用（需端点可达）

所有由安全增强管理的资源带有统一标签。

```bash
kubectl get all -A -l app.kubernetes.io/managed-by=tke-security-mode
```

```text
NAME  STATUS  AGE
...
```

```output
NAMESPACE   NAME                 READY   STATUS    RESTARTS   AGE
gatekeeper-system   pod/gatekeeper-controller-manager-xxx   1/1     Running   0          10d
```

### 2. 机制一：MetaServer 阻断（需端点可达）

安全增强通过 Gatekeeper Assign（Mutation）向符合条件的 Pod 注入名为 `security-defense` 的 initContainer，通过 iptables 规则阻断对 `169.254.0.23` 的访问。

```bash
kubectl get assign -A
```

```text
NAME  STATUS  AGE
...
```

```output
NAME                        AGE
addinitcontainer            10d
```

```bash
kubectl get assign addinitcontainer -o yaml
```

```text
NAME  STATUS  AGE
...
```

查看 Pod 的 initContainer 注入情况：

```bash
kubectl describe pod <pod-name> -n <namespace> | grep -A 10 "Init Containers"
```

```text
Name:         ...
Status:       Running
...
```

预期输出包含名为 `security-defense` 的 initContainer。

### 3. 机制二：准入控制（Gatekeeper Constraints，需端点可达）

安全增强部署以下四条 Gatekeeper 约束（deny 模式）：

| 约束名称 | 类型 | 作用 |
|----------|------|------|
| k8spspprivilegedcontainer | Gatekeeper Constraints（deny） | 禁止创建特权容器 |
| k8spspautomountserviceaccounttokenpod | Gatekeeper Constraints（deny） | 禁止 Pod 自动挂载 ServiceAccount Token |
| blockvolumemountpath | Gatekeeper Constraints（deny） | 限制卷挂载路径，防止覆盖敏感目录 |
| blockmountablevolumetype | Gatekeeper Constraints（deny） | 限制可挂载的卷类型 |

```bash
kubectl get constraints -A
```

```text
NAME  STATUS  AGE
...
```

```output
NAME                                                    ENFORCEMENT-ACTION   TOTAL-VIOLATIONS
k8spspprivilegedcontainer.constraints.gatekeeper.sh     deny                 0
k8spspautomountserviceaccounttokenpod                   deny                 0
blockvolumemountpath                                    deny                 0
blockmountablevolumetype                                deny                 0
```

### 4. 机制三：网络隔离（NetworkPolicy，需端点可达）

安全增强部署名为 `security-mode-block-container-subnet` 的 NetworkPolicy，阻止 Pod 间通过 Pod IP 直接通信，仅允许通过 Service CIDR 和 DNS 访问。

```bash
kubectl get networkpolicies -A
```

```text
NAME  STATUS  AGE
...
```

```output
NAMESPACE   NAME                                      POD-SELECTOR   AGE
default     security-mode-block-container-subnet      <none>         10d
```

```bash
kubectl describe networkpolicy security-mode-block-container-subnet -n default
```

```text
Name:         ...
Status:       Running
...
```

### 5. 灰度发布控制

安全增强支持按 Namespace 或 Pod Label 进行灰度发布。特定命名空间或带有特定标签的 Pod 优先启用安全策略。系统命名空间默认豁免。

**豁免的系统命名空间：** `kube-system`、`kube-public`、`kube-node-lease`、`gatekeeper-system`、`tke-backup`、`crane-system`

## 验证

### 控制面（tccli）

```bash
# 确认集群仍正常运行
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
            "VpcId": "vpc-xxxxxxxx",
            "CreatedTime": "2024-06-15T10:30:00Z"
        }
    ],
    "RequestId": "a1b2c3d4-e5f6-7890-abcd-ef1234567890"
}
```

### 数据面（kubectl，需端点可达）

```bash
# 检查安全增强核心组件状态
kubectl get constraints -A
kubectl get networkpolicies -A | grep security-mode
kubectl get assign -A

# 检查 Gatekeeper 审计日志（violations）
kubectl get constraints -A -o json | jq '.items[].status.totalViolations'
```

```text
NAME  STATUS  AGE
...
```

```output
0
0
0
0
```

## 清理

无需清理。集群安全增强能力的开启和关闭为控制台独有操作，无法通过 CLI 或 kubectl 完成。若需关闭，请通过控制台操作。

若仅需删除测试资源（需端点可达）：

```bash
kubectl delete pod security-test-pod --ignore-not-found
rm -f test-privileged.yaml
```

> 被误删的约束会被系统在数秒内自动调和（reconcile）重新创建。

## 排障

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| Pod 无法访问元数据服务 `169.254.0.23` | 这是预期行为，安全增强的 MetaServer 阻断功能正常工作 | `security-defense` initContainer iptables 规则已生效 | 无需修复，业务应改用 CAM 角色或实例 RAM 获取凭证 |
| 安全增强开启后 Pod 创建失败 | `kubectl get constraints` 查看 violation 计数 | Pod 违反 Gatekeeper 约束（如特权容器、自动挂载 Token） | 修改 Pod Spec 去除不合规字段 |
| Pod 间网络不通 | `kubectl describe networkpolicy security-mode-block-container-subnet -n default` 检查策略 | NetworkPolicy 阻止 Pod IP 直连 | 改用 Service 而非 Pod IP 通信 |
| 灰度命名空间的 Pod 未注入 initContainer | 检查命名空间标签或 Pod 标签是否匹配灰度配置 | 标签不匹配灰度规则 | 在控制台调整灰度发布范围 |
| 安全增强未生效 | 确认在 TKE 控制台已开启「集群安全增强」；`kubectl get pods -n gatekeeper-system` 检查 Pod 状态 | 未开启功能或 Gatekeeper 组件未就绪 | 在控制台开启功能；等待 `gatekeeper-system` 命名空间 Pod 全部 Running |
| 约束被误删除 | 等待数秒后 `kubectl get constraints` 重新查看 | 系统会自动调和重建 | 无需手动干预 |

## 下一步

- [策略管理（OPA/Gatekeeper）](../应用安全/策略管理/tccli%20操作.md) — 自定义 Gatekeeper 约束与策略
- [安全容器（Kata）](../安全容器/tccli%20操作.md) — VM 级别强隔离
- [使用腾讯云密钥管理系统 KMS 进行 ETCD 数据加密](../控制平面安全/使用腾讯云密钥管理系统%20KMS%20进行%20ETCD%20数据加密/tccli%20操作.md) — etcd Secret 加密

## 控制台替代

[TKE 控制台 → 集群详情](https://console.cloud.tencent.com/tke2/cluster) → 安全 → 集群安全增强 → 点击「开启」按钮。开启后可通过控制台查看安全策略覆盖范围、Gatekeeper 审计日志和违规统计数据，支持按命名空间或 Pod 标签配置灰度发布范围。
