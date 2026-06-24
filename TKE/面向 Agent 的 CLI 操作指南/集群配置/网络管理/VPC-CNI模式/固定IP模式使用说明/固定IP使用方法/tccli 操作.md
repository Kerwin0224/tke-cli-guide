# 固定IP使用方法（tccli）

> 对照官方：[固定IP使用方法](https://cloud.tencent.com/document/product/457/34994) · page_id `34994`

## 概述

VPC-CNI 固定 IP 模式允许 Pod 在重启和重新调度后保留其 IP 地址不变。Pod IP 与 Pod 的 namespace + name 绑定，只要 StatefulSet 不被删除，Pod 重建后仍可获得相同的 VPC 子网 IP。该模式适用于有状态工作负载（如数据库、消息队列）、需要基于 IP 做白名单的场景、以及依赖固定 IP 进行服务注册发现的业务。

当前集群 `cls-xxxxxxxx`（kerwinwjyan-rewrite-s1）尚未开启 VPC-CNI（EnableIPAMD: false），需先开启 VPC-CNI 网络模式并启用固定 IP 特性。

## 前置条件

- 集群状态为 Running，且已开启 VPC-CNI 网络模式（EnableVpcCniNetworkType）
- VPC（vpc-of73262z）下有可用的子网，子网 IP 充足
- 已安装 kubectl 并配置好集群访问凭证
- 工作负载类型必须为 StatefulSet（固定 IP 仅支持 StatefulSet）
- 具备 tccli 和 kubectl 的执行权限

## 控制台与 CLI 参数映射

| 控制台操作 | CLI | 幂等 |
|-----------|-----|:--:|
| 查询 VPC-CNI 状态 | tccli tke DescribeIPAMD --region ap-guangzhou --ClusterId cls-xxxxxxxx | 是 |
| 开启固定 IP 模式 | tccli tke EnableVpcCniNetworkType --region ap-guangzhou --ClusterId cls-xxxxxxxx --VpcCniType "tke-route-eni" --EnableStaticIp true | 否 |
| 添加子网 | tccli tke AddVpcCniSubnets --region ap-guangzhou --ClusterId cls-xxxxxxxx --SubnetIds '["subnet-xxxxxxxx"]' | 是 |
| 创建 StatefulSet（固定 IP） | kubectl apply -f statefulset.yaml | 否 |
| 验证 Pod IP 保持不变 | kubectl delete pod <NAME> && kubectl get pod <NEW> -o wide | 是 |
| 查询 Pod 数量限制 | tccli tke DescribeVpcCniPodLimits --region ap-guangzhou --Zone ap-guangzhou-3 --InstanceFamily S5 --InstanceType S5.MEDIUM2 | 是 |

## 操作步骤

### 1. 查询 VPC-CNI 状态

```bash
tccli tke DescribeIPAMD --region ap-guangzhou --ClusterId cls-xxxxxxxx
```

```json
{
  "EnableIPAMD": true,
  "EnableCustomizedPodCidr": true,
  "DisableVpcCniMode": true,
  "Phase": "<Phase>",
  "Reason": "<Reason>",
  "SubnetIds": [],
  "ClaimExpiredDuration": "<ClaimExpiredDuration>",
  "EnableTrunkingENI": true
}
```

期望输出中 `EnableIPAMD` 为 `true`，`Phase` 为 `Running`。若 `EnableIPAMD` 为 `false`（当前状态），需先开启 VPC-CNI。

### 2. 开启 VPC-CNI 并启用固定 IP 模式

```bash
tccli tke EnableVpcCniNetworkType \
  --region ap-guangzhou \
  --ClusterId cls-xxxxxxxx \
  --VpcCniType "tke-route-eni" \
  --EnableStaticIp true
```

该操作会为集群开启 VPC-CNI 网络模式并同时启用固定 IP 特性。操作返回 `RequestId`，可通过 DescribeIPAMD 轮询确认 `Phase` 变为 `Running`。

注意：若 VPC-CNI 已开启但未启用固定 IP，再次调用此接口设置 `--EnableStaticIp true` 即可单独开启固定 IP 模式。

### 3. 添加子网

```bash
tccli tke AddVpcCniSubnets \
  --region ap-guangzhou \
  --ClusterId cls-xxxxxxxx \
  --SubnetIds '["subnet-xxxxxxxx"]'
```

VPC-CNI 模式下 Pod IP 来自 VPC 子网，需确保子网 IP 充足。`SubnetIds` 为 JSON 数组格式的字符串。该接口幂等，重复调用不会重复添加。

### 4. 创建使用固定 IP 的 StatefulSet

编写 StatefulSet YAML，在 `spec.template.metadata.annotations` 中添加固定 IP 注解：

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: static-ip-demo
  namespace: default
spec:
  serviceName: "static-ip-demo"
  replicas: 2
  selector:
    matchLabels:
      app: static-ip-demo
  template:
    metadata:
      labels:
        app: static-ip-demo
      annotations:
        tke.cloud.tencent.com/vpc-ip-claim-delete-policy: "Never"
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80
```

注解 `tke.cloud.tencent.com/vpc-ip-claim-delete-policy` 控制 IP 回收策略：
- `"Never"`：仅在 StatefulSet 删除时释放 IP，Pod 删除不释放
- `"Immediate"`：Pod 删除时立即释放 IP

应用配置：

```bash
kubectl apply -f statefulset.yaml
```

```text
# command executed successfully
```

### 5. 验证固定 IP 效果

查看 Pod IP：

```bash
kubectl get pod -o wide -l app=static-ip-demo
```

```text
NAME  STATUS  AGE
...
```

记录 Pod 的 IP 地址，然后删除一个 Pod：

```bash
kubectl delete pod static-ip-demo-0
```

StatefulSet 控制器会自动重建 Pod，再次查看 Pod IP：

```bash
kubectl get pod -o wide -l app=static-ip-demo
```

```text
NAME  STATUS  AGE
...
```

重建后的 Pod IP 应与删除前一致。

## 验证

- `DescribeIPAMD` 返回 `EnableIPAMD: true` 且 `Phase: Running`
- StatefulSet Pod 成功调度并分配 VPC 子网 IP
- 删除 Pod 后，新 Pod 获得相同的 IP 地址
- `kubectl describe pod static-ip-demo-0` 中可看到 `tke.cloud.tencent.com/vpc-ip` 注解记录了分配的 IP

## 清理

- 删除 StatefulSet：`kubectl delete statefulset static-ip-demo`（若注解为 `"Never"`，IP 在 StatefulSet 删除时释放）
- 关闭固定 IP 模式需通过控制台或重新调用 `EnableVpcCniNetworkType` 调整配置
- **警告**：`DisableVpcCniNetworkType` 会更改所有 Pod 的 IP，要求 Pod 重建，可能影响业务。请谨慎操作。

## 排障

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| Pod 重建后 IP 变化 | 检查 Pod annotations 是否包含 `tke.cloud.tencent.com/vpc-ip-claim-delete-policy` | StatefulSet 模板未配置固定 IP 注解 | 在 StatefulSet spec.template.metadata.annotations 中添加 `tke.cloud.tencent.com/vpc-ip-claim-delete-policy: "Never"` |
| Pod 处于 Pending 状态 | `kubectl describe pod` 查看 Events，关注 "failed to allocate IP" | VPC 子网 IP 池耗尽 | 通过 `AddVpcCniSubnets` 添加新子网 |
| 开启固定 IP 失败 | 查看 API 返回的错误信息 | 集群未先开启 VPC-CNI 或 VPC 无可用子网 | 先调用 `EnableVpcCniNetworkType --VpcCniType "tke-route-eni"` 开启 VPC-CNI，再启用固定 IP |
| DescribeIPAMD 报错 | 分析错误码 | GR 集群查询 VPC-CNI 状态可能返回 `FailedOperation.EnableVPCCNIFailed: cluster is not vpc-cni cluster` | 先调用 `EnableVpcCniNetworkType` 开启 VPC-CNI |
| Pod 数量超出限制 | `kubectl get events` 查看调度失败原因 | 节点 Pod 数量达到上限（S5.MEDIUM2 上限 27） | 扩容节点或使用更高规格实例 |

## 下一步

- [固定 IP 相关特性](https://cloud.tencent.com/document/product/457/50358) -- page_id `50358`
- [多 Pod 共享网卡](https://cloud.tencent.com/document/product/457/50356) -- page_id `50356`

## 控制台替代

登录 [容器服务控制台](https://console.cloud.tencent.com/tke2/cluster/sub/detail/resource/statefulset?rid=1&clusterId=cls-xxxxxxxx)，进入目标集群的「工作负载」→「StatefulSet」页面，创建 StatefulSet 时在「高级设置」中勾选「VPC-CNI 固定 IP」并选择回收策略。
