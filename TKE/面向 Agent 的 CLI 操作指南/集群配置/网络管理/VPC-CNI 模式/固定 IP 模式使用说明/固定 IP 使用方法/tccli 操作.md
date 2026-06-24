# 固定 IP 使用方法（tccli）

> 对照官方：[固定 IP 使用方法](https://cloud.tencent.com/document/product/457/34994) · page_id `34994`

## 概述

对照[官方 固定 IP 使用方法](https://cloud.tencent.com/document/product/457/34994)，保留产品与限制说明，并用 **tccli + kubectl** 完成控制面 VPC-CNI 开启、数据面 StatefulSet 固定 IP 工作负载创建的完整步骤。

## 前置条件

- [环境准备](../../../../../环境准备.md)：`tccli`、`kubectl`、地域 `ap-guangzhou` 与凭证已配置。
- 新建集群见 [创建集群](../../../../%E9%9B%86%E7%BE%A4%E7%AE%A1%E7%90%86/%E5%88%9B%E5%BB%BA%E9%9B%86%E7%BE%A4/tccli%20%E6%93%8D%E4%BD%9C.md)；下载 kubeconfig 见 [连接集群](../../../../集群管理/连接集群/tccli%20操作.md)。

## 控制台与 CLI 参数映射

| 官方 / 控制台 | 含义 | tccli / 说明 | 幂等 |
|---------------|------|-------------|------|
| 新建 VPC-CNI 并勾选固定 Pod IP | 开启固定 IP 模式 | `EnableVpcCniNetworkType` + `EnableStaticIp=true` | 否 |
| 固定 IP 回收策略 | Pod 销毁后 IP 保留时长 | `--claim-expired-duration`（eniipamd 组件参数） | 否 |
| 创建 StatefulSet（固定 IP） | 工作负载 | `kubectl apply -f sts.yaml`（带固定 IP 注解） | 是 |
| IP 地址范围选择 | 控制台「随机」 | 不支持指定，系统自动分配 | 是 |

## 操作步骤

### 使用场景

固定 IP 适用于依赖固定 IP 的场景：传统架构迁移至容器平台、IP 安全策略、IP 审计与日志查询。不依赖 IP 的工作负载不推荐此模式。

### 能力和限制

**支持：**
- Pod 销毁 IP 保留；Pod 迁移 IP 不变
- 多子网支持
- 固定 IP Pod 可跨普通节点、原生节点、超级节点迁移保留 IP

**限制：**
- 固定 IP Pod 不可跨子网调度（即不支持跨可用区）
- VPC-CNI 集群创建后不可动态开关固定 IP
- GR 模式集群需先关闭当前 VPC-CNI 再重新开启并勾选固定 IP

### 步骤一：开启集群固定 IP 模式

#### 方式 A：新建 VPC-CNI 固定 IP 集群

在 `CreateCluster` 请求中将 `ClusterNetworkSettings` 设为 VPC-CNI，完整 JSON 参见 [创建集群](../../../../%E9%9B%86%E7%BE%A4%E7%AE%A1%E7%90%86/%E5%88%9B%E5%BB%BA%E9%9B%86%E7%BE%A4/tccli%20%E6%93%8D%E4%BD%9C.md)。创建后 `EnableStaticIp` 不可逆。

#### 方式 B：为已有 GR 集群附加 VPC-CNI 固定 IP

```bash
tccli tke EnableVpcCniNetworkType \
  --ClusterId <ClusterId> \
  --SubnetId <SubnetId> \
  --VpcCniType "tke-route-eni" \
  --EnableStaticIp true \
  --region ap-guangzhou
```

```json
{
  "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

# expected: exit 0, contains "RequestId"

```bash
tccli tke DescribeClusters --region ap-guangzhou --output json \
  --filter "Clusters[?ClusterId=='<ClusterId>'] | [0].{ClusterId:ClusterId,ClusterNetworkSettings:ClusterNetworkSettings}"
```

```json
{
  "ClusterId": "<ClusterId>",
  "ClusterNetworkSettings": {
    "ClusterCIDR": "10.168.0.0/16",
    "VpcId": "vpc-example",
    "Cni": true,
    "EnableStaticIp": true,
    "Subnets": ["subnet-example"],
    "VpcCniType": "tke-route-eni"
  }
}
```

# expected: exit 0, contains "EnableStaticIp"

> **注意：** 开启后须在 eniipamd 组件配置中设置 **IP 回收策略**（`--claim-expired-duration`，非固定 IP Pod 不受此策略影响仍立即释放）。

### 步骤二：创建固定 IP StatefulSet 工作负载

固定 IP 仅保证在 **StatefulSet** 生命周期内 Pod 重建 IP 不变。

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  labels:
    k8s-app: busybox
  name: busybox
  namespace: default
spec:
  replicas: 3
  selector:
    matchLabels:
      k8s-app: busybox
      qcloud-app: busybox
  serviceName: ""
  template:
    metadata:
      annotations:
        tke.cloud.tencent.com/networks: "tke-route-eni"
        tke.cloud.tencent.com/vpc-ip-claim-delete-policy: Never
      labels:
        k8s-app: busybox
        qcloud-app: busybox
    spec:
      containers:
      - args:
        - "10000000000"
        command:
        - sleep
        image: busybox
        imagePullPolicy: Always
        name: busybox
        resources:
          limits:
            tke.cloud.tencent.com/eni-ip: "1"
          requests:
            tke.cloud.tencent.com/eni-ip: "1"
```

关键注解说明：

| 注解 | 值 | 作用 |
|------|-----|------|
| `tke.cloud.tencent.com/networks` | `tke-route-eni`（共享网卡）或 `tke-direct-eni`（独占网卡） | 标记为 VPC-CNI 模式 Pod。**纯 VPC-CNI 集群无需此注解** |
| `tke.cloud.tencent.com/vpc-ip-claim-delete-policy` | `Never`（保留 IP）/ `Immediate`（默认，销毁即释放） | Pod 销毁后 IP 是否保留 |

```bash
kubectl apply -f sts-fixed-ip.yaml
```

```text
statefulset.apps/busybox created
```

# expected: exit 0, contains "created"

查看 Pod IP：

```bash
kubectl get pods -l k8s-app=busybox -o wide
```

```text
NAME        READY  STATUS   RESTARTS  AGE  IP            NODE
busybox-0   1/1    Running  0         10s  172.16.0.10   node-example-1
busybox-1   1/1    Running  0         10s  172.16.0.11   node-example-1
busybox-2   1/1    Running  0         10s  172.16.0.12   node-example-2
```

# expected: exit 0, contains "Running"

删除 Pod 验证 IP 保留：

```bash
kubectl delete pod busybox-0
kubectl get pods -l k8s-app=busybox -o wide
```

```text
NAME        READY  STATUS   RESTARTS  AGE  IP            NODE
busybox-0   1/1    Running  0         5s   172.16.0.10   node-example-1
```

# expected: exit 0, IP 与删除前一致

## 验证

### 控制面（tccli）

- `DescribeClusters` → `EnableStaticIp` 为 `true`，`VpcCniType` 为 `tke-route-eni` 或 `tke-direct-eni`。

### 数据面（kubectl）

- `kubectl get pods -o wide` 显示 Pod IP 为 VPC 子网内地址。
- 删除 Pod 后新 Pod 同名 IP 不变。

## 清理

```bash
kubectl delete sts busybox -n default
```

```text
statefulset.apps "busybox" deleted
```

# expected: exit 0, contains "deleted"

若测试使用了 `EnableVpcCniNetworkType` 开启 VPC-CNI，勿在生产集群上随意关闭。

## 排障

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `EnableVpcCniNetworkType` 返回权限错误 | 执行 `tccli tke DescribeClusters --region ap-guangzhou` 确认 CAM 权限；查看错误信息中的 `UnauthorizedOperation` 详情 | CAM 策略缺少 `tke:EnableVpcCniNetworkType` 权限 | 在 CAM 策略中添加 `tke:EnableVpcCniNetworkType` Action 授权 |
| Pod 处于 Pending 状态 | 执行 `kubectl describe pod <pod-name>` 查看 Events；执行 `kubectl get nodes -o wide` 确认节点子网信息 | 节点所在子网 IP 耗尽，或 `tke.cloud.tencent.com/eni-ip` 资源不可分配 | 添加该可用区内子网到容器网段；或检查 eniipamd 组件是否正常运行 |
| 删除固定 IP Pod 后新建 Pod IP 变化 | 执行 `kubectl get pod <pod-name> -o yaml | grep vpc-ip-claim-delete-policy` 检查注解；执行 `kubectl get sts <name> -o yaml` 确认工作负载类型 | 注解 `vpc-ip-claim-delete-policy` 未设为 `Never`，或工作负载类型为 Deployment（不支持固定 IP） | 在 Pod 模板中设置注解 `tke.cloud.tencent.com/vpc-ip-claim-delete-policy: Never`；确认使用 StatefulSet 而非 Deployment |
| 固定 IP Pod 调度失败 | 执行 `kubectl describe pod <pod-name>` 查看调度事件；执行 `kubectl get nodes -o wide` 确认节点所在可用区与子网 | 固定 IP Pod 不支持跨子网调度，节点与 Pod 不在同一子网 | 确保目标节点所在子网与 Pod IP 所属子网一致；或扩容目标子网所在可用区的节点 |

## 下一步

- [固定 IP 相关特性](https://cloud.tencent.com/document/product/457/50358)（生命周期、回收策略）· [非固定 IP 模式使用说明](https://cloud.tencent.com/document/product/457/64940)

## 控制台替代

[控制台 → 集群](https://console.cloud.tencent.com/tke2/cluster) → 基本信息 → 节点与网络信息 → VPC-CNI 开启 + 勾选「开启支持」固定 Pod IP。
