# 告警管理（tccli）

> 对照官方：[告警管理](https://cloud.tencent.com/document/product/457/121328) · page_id `121328`

## 概述

容器服务 TKE 支持通过腾讯云可观测平台（TCOP）为集群配置告警策略。告警管理覆盖资源维度（CPU、内存、网络、磁盘等）和事件维度（Pod 异常重启、节点异常等），支持默认告警策略和自定义告警策略，可通过控制台或 monitor API 进行管理。

## 前置条件

- `tccli` 已配置凭证
- kubectl 不可达：演示集群 cls-xxxxxxxx 为托管集群，公网端点可能受 CAM 策略限制（strategyId:240463971）；内网无 VPN 时不可达。以下 kubectl 命令作为文档参考。
- 已开通腾讯云可观测平台（TCOP）服务
- 集群节点已安装监控组件（托管集群默认已安装）

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
| 查看告警策略列表 | `monitor:DescribeAlarmPolicies` | 是 |
| 创建告警策略 | `monitor:CreateAlarmPolicy` | 否 |
| 修改告警策略 | `monitor:ModifyAlarmPolicy` | 是 |
| 删除告警策略 | `monitor:DeleteAlarmPolicy` | 否 |
| 绑定告警策略到集群 | `monitor:BindAlarmPolicyObject` | 否 |
| 查看告警历史 | `monitor:DescribeAlarmHistories` | 是 |
| 查看通知模板 | `monitor:DescribeAlarmNotices` | 是 |
| 创建通知模板 | `monitor:CreateAlarmNotice` | 否 |
| 查看默认告警策略 | 控制台 | - |

## 操作步骤

### 查看现有告警策略

```bash
tccli monitor DescribeAlarmPolicies \
  --Module monitor \
  --MonitorTypes '["MT_QCE"]' \
  --Namespaces '["qce/tke2"]' \
  --region ap-guangzhou
```

```json
{
  "RequestId": "a3b4c5d6-e7f8-9012-3456-789012345678",
  "TotalCount": 2,
  "Policies": [
    {
      "PolicyId": "policy-xxxxxxxx",
      "PolicyName": "TKE 默认告警策略",
      "MonitorType": "MT_QCE",
      "Namespace": "qce/tke2",
      "Enable": 1,
      "NoticeIds": ["notice-xxxxxxxx"],
      "Conditions": [
        {
          "MetricName": "K8sNodeCpuUsage",
          "Operator": "gt",
          "Threshold": "80"
        }
      ]
    }
  ]
}
```

### 创建自定义告警策略

创建一个 Pod CPU 使用率超过 90% 的告警策略：

```bash
tccli monitor CreateAlarmPolicy \
  --Module monitor \
  --PolicyName "cls-xxxxxxxx-pod-cpu-alert" \
  --MonitorType MT_QCE \
  --Namespace qce/tke2 \
  --Enable 1 \
  --Condition '{"IsUnionRule":0,"Rules":[{"MetricName":"K8sPodCpuUsage","Period":60,"Operator":"gt","Value":"90","ContinuePeriod":3,"NoticeFrequency":86400}]}' \
  --Remark "Pod CPU 使用率超过 90% 告警" \
  --region ap-guangzhou
```

```json
{
  "RequestId": "b4c5d6e7-f8a9-0123-4567-890123456789",
  "PolicyId": "policy-yyyyyyyy"
}
```

常用监控指标说明：

| 指标 | MetricName | 说明 |
|------|-----------|------|
| 节点 CPU 使用率 | `K8sNodeCpuUsage` | 节点 CPU 使用率百分比 |
| 节点内存使用率 | `K8sNodeMemUsage` | 节点内存使用率百分比 |
| Pod CPU 使用率 | `K8sPodCpuUsage` | Pod CPU 使用率百分比 |
| Pod 内存使用率 | `K8sPodMemUsage` | Pod 内存使用率百分比 |
| Pod 重启次数 | `K8sPodRestartTotal` | Pod 重启累计次数 |
| 节点磁盘使用率 | `K8sNodeDiskUsage` | 节点磁盘使用率百分比 |

### 绑定告警策略到集群

```bash
tccli monitor BindAlarmPolicyObject \
  --Module monitor \
  --PolicyId policy-yyyyyyyy \
  --Dimensions '[{"Key":"tke_cluster_instance_id","Value":"cls-xxxxxxxx"}]' \
  --region ap-guangzhou
```

```json
{
  "RequestId": "c5d6e7f8-a9b0-1234-5678-901234567890"
}
```

### 修改告警策略

```bash
tccli monitor ModifyAlarmPolicy \
  --Module monitor \
  --PolicyId policy-yyyyyyyy \
  --Enable 0 \
  --region ap-guangzhou
```

```json
{
  "RequestId": "d6e7f8a9-b0c1-2345-6789-012345678901"
}
```

### 查看告警历史

```bash
tccli monitor DescribeAlarmHistories \
  --Module monitor \
  --MonitorTypes '["MT_QCE"]' \
  --region ap-guangzhou
```

```json
{
  "RequestId": "e7f8a9b0-c1d2-3456-7890-123456789012",
  "TotalCount": 1,
  "Histories": [
    {
      "AlarmId": "alarm-xxxxxxxx",
      "PolicyId": "policy-xxxxxxxx",
      "Content": "Pod CPU 使用率 95.2% > 90%",
      "AlarmStatus": "ALARM",
      "FirstOccurTime": "2026-06-18 10:30:00"
    }
  ]
}
```

### 查看通知模板

```bash
tccli monitor DescribeAlarmNotices --Module monitor --region ap-guangzhou
```

```json
{
  "RequestId": "f8a9b0c1-d2e3-4567-8901-234567890123",
  "TotalCount": 1,
  "Notices": [
    {
      "Id": "notice-xxxxxxxx",
      "Name": "默认通知模板",
      "NoticeType": "ALARM",
      "NoticeLanguage": "zh-CN"
    }
  ]
}
```

### 删除告警策略

```bash
tccli monitor DeleteAlarmPolicy \
  --Module monitor \
  --PolicyIds '["policy-yyyyyyyy"]' \
  --region ap-guangzhou
```

```json
{
  "RequestId": "a9b0c1d2-e3f4-5678-9012-345678901234"
}
```

## 验证

### Control plane (tccli)

```bash
tccli monitor DescribeAlarmPolicies \
  --Module monitor \
  --MonitorTypes '["MT_QCE"]' \
  --Namespaces '["qce/tke2"]' \
  --region ap-guangzhou
```

确认告警策略列表包含已创建的策略。

## 清理

### Control plane (tccli)

```bash
# 解绑告警策略对象
tccli monitor UnbindAlarmPolicyObject \
  --Module monitor \
  --PolicyId policy-yyyyyyyy \
  --Dimensions '[{"Key":"tke_cluster_instance_id","Value":"cls-xxxxxxxx"}]' \
  --region ap-guangzhou

# 删除告警策略
tccli monitor DeleteAlarmPolicy \
  --Module monitor \
  --PolicyIds '["policy-yyyyyyyy"]' \
  --region ap-guangzhou
```

## 排障

| 现象 | 处理 |
|------|------|
| 无告警数据 | 确认集群已安装监控组件；数据上报有 1-2 分钟延迟 |
| 告警未触发 | 检查告警策略阈值和持续周期设置（ContinuePeriod）；确认策略已启用（Enable=1） |
| 告警未通知 | 检查通知模板是否正确关联（NoticeIds）；确认接收人已配置 |
| 默认告警策略不可删除 | TKE 默认告警策略不可删除，仅可停用或修改 |
| 权限不足 | 使用 monitor API 需要 QCloudMonitorFullAccess 策略或对应 CAM 权限 |
| kubectl 不可达 | 托管集群公网端点可能受 CAM 策略限制（strategyId:240463971）；内网无 VPN 时不可达；告警策略管理通过 monitor API 操作，不依赖 kubectl |

## 下一步

- [容器服务可观测体系概述](../容器服务可观测体系概述/tccli 操作.md) — 可观测体系整体架构
- [风险推送](../风险推送/tccli 操作.md) — kubejarvis 健康检查风险推送
- [监控告警概述](../监控管理/基础监控与告警/监控告警概述/tccli 操作.md) — 监控告警概念与配置

## 控制台替代

登录 [容器服务控制台](https://console.cloud.tencent.com/tke2) > 选择集群 > 监控 > 告警配置，或登录 [腾讯云可观测平台控制台](https://console.cloud.tencent.com/monitor/alarm2/policy) 管理告警策略。
