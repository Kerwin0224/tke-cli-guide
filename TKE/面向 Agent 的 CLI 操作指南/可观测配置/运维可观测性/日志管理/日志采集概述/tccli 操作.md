# 日志采集概述（tccli）

> 对照官方：[日志采集概述](https://cloud.tencent.com/document/product/457/83871) · page_id `83871`

## 概述

日志采集功能是 TKE 提供的集群内日志采集工具，可将集群内服务或容器/节点文件的日志发送至 CLS（日志服务）或 CKafka（消息队列）。日志采集 Agent（LogListener）以 DaemonSet 方式在集群内运行。开启后需手动配置采集规则（控制台或 CRD 方式）。

## 前置条件

- `tccli` 已配置凭证
- kubectl 不可达：演示集群 cls-xxxxxxxx 为托管集群，公网端点可能受 CAM 策略限制（strategyId:240463971）；内网无 VPN 时不可达。以下 kubectl 命令作为文档参考。
- 集群节点有足够资源（CPU 0.11-1.1核，内存 24-560MB）
- 集群 Kubernetes 版本 >= 1.10

```bash
tccli tke DescribeClusters --ClusterIds '["cls-xxxxxxxx"]' --region ap-guangzhou
```

```json
{
  "RequestId": "1e8b2c3d-4a5f-6789-abcd-ef0123456789",
  "TotalCount": 1,
  "Clusters": [
    {
      "ClusterId": "cls-xxxxxxxx",
      "ClusterName": "my-test-cluster",
      "ClusterType": "MANAGED_CLUSTER",
      "ClusterStatus": "Running",
      "ClusterVersion": "1.30.0",
      "ClusterNodeNum": 2
    }
  ]
}
```

## 控制台与 CLI 参数映射

| 控制台操作 | CLI | 幂等 |
|-----------|-----|------|
| 开启日志采集 | `tccli tke InstallLogAgent` | 是 |
| 关闭日志采集 | `tccli tke UninstallLogAgent` | 否 |
| 创建采集规则 | `tccli tke CreateCLSLogConfig` 或 `kubectl apply -f logconfig.yaml` | 是 |
| 查看采集状态 | `kubectl get pods -n kube-system -l app=loglistener` | 是 |

## 操作步骤

### 开启日志采集

```bash
tccli tke InstallLogAgent --ClusterId cls-xxxxxxxx --region ap-guangzhou
```

```json
{
  "RequestId": "a5b6c7d8-e9f0-1234-5678-901234567890"
}
```

### 查看 LogListener DaemonSet 状态

```bash
kubectl get ds -n kube-system loglistener-tke
```

```text
NAME               DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE
loglistener-tke    3         3         3       3            3           <none>          5m
```

> 注意：kubectl 不可达时，此命令仅作为文档参考。实际状态可通过控制台 [运维功能管理](https://console.cloud.tencent.com/tke2/ops/list) 查看。

### 查看日志采集 Agent Pod

```bash
kubectl get pods -n kube-system -l app=loglistener-tke
```

```text
NAME                     READY   STATUS    RESTARTS   AGE
loglistener-tke-abcde    1/1     Running   0          5m
loglistener-tke-xyzab    1/1     Running   0          5m
loglistener-tke-defgh    1/1     Running   0          5m
```

### 关闭日志采集

```bash
tccli tke UninstallLogAgent --ClusterId cls-xxxxxxxx --region ap-guangzhou
```

```json
{
  "RequestId": "b6c7d8e9-f0a1-2345-6789-012345678901"
}
```

## 验证

### Control plane (tccli)

```bash
tccli tke DescribeLogSwitches --ClusterIds '["cls-xxxxxxxx"]' --region ap-guangzhou
```

```json
{
  "RequestId": "c7d8e9f0-a1b2-3456-7890-123456789012",
  "SwitchSet": [
    {
      "ClusterId": "cls-xxxxxxxx",
      "LogAgent": {
        "Enable": true
      }
    }
  ]
}
```

### Data plane (kubectl) — 参考

```bash
kubectl get ds -n kube-system loglistener-tke && kubectl get pods -n kube-system -l app=loglistener-tke
```

```text
NAME               DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE
loglistener-tke    3         3         3       3            3           <none>          5m

NAME                     READY   STATUS    RESTARTS   AGE
loglistener-tke-abcde    1/1     Running   0          5m
```

## 清理

### Control plane (tccli)

```bash
tccli tke UninstallLogAgent --ClusterId cls-xxxxxxxx --region ap-guangzhou
```

## 排障

| 现象 | 处理 |
|------|------|
| Agent 资源不足 | CPU 占用 0.11-1.1核，内存 24-560MB；日志量大时可调大资源 limits |
| 日志长度超过限制 | 单条日志最大 512KB，超出截断 |
| 节点无法访问消费端 | 确认集群节点能访问 CLS 或 Kafka 网络 |
| 采集规则不生效 | 日志采集规则变化最多 10s 内生效；检查 LogConfig status 是否为 Synced |
| 采集路径使用 CFS | 配置全量采集会造成重采，多个 Pod 可能重复上报，建议日志源不使用 CFS |
| kubectl 不可达 | 托管集群公网端点可能受 CAM 策略限制（strategyId:240463971）；内网无 VPN 时不可达；日志采集开关通过 `tccli tke InstallLogAgent`/`UninstallLogAgent` 操作，采集规则通过控制台或 `tccli tke CreateCLSLogConfig` 创建 |

## 下一步

- [采集容器日志到 CLS](../采集容器日志到 CLS/tccli 操作.md) — 控制台创建日志采集规则
- [使用 CRD 进行日志采集配置指导](../使用 CRD 配置日志采集/使用 CRD 进行日志采集配置指导/tccli 操作.md) — CRD 方式配置日志采集

## 控制台替代

登录 [容器服务控制台](https://console.cloud.tencent.com/tke2) > 选择集群 > 日志 > 业务日志 > 开启/关闭。
