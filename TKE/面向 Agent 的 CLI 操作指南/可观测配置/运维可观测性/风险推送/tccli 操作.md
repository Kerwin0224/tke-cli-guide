# 风险推送（tccli）

> 对照官方：[风险推送](https://cloud.tencent.com/document/product/457/95136) · page_id `95136`

## 概述

容器服务 TKE 通过与腾讯云事件总线 EB（EventBridge）对接，支持自定义推送规则配置。TKE 检测到的风险（如集群资源数量超过推荐值/最大配额）可以通过短信、站内信等方式推送给用户。前置条件：kubejarvis 组件版本 >= 1.0.8。

## 前置条件

- `tccli` 已配置凭证
- kubectl 不可达：演示集群 cls-xxxxxxxx 为托管集群，公网端点可能受 CAM 策略限制（strategyId:240463971）；内网无 VPN 时不可达。以下 kubectl 命令作为文档参考。
- 已开通事件总线 EB 服务
- kubejarvis 组件版本 >= 1.0.8

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
| 查看 kubejarvis 版本 | `tccli tke DescribeAddon --AddonName kubejarvis` | 是 |
| 升级 kubejarvis | `tccli tke UpdateAddon --AddonName kubejarvis` | 否 |
| 配置 kubejarvis 导出器 | `kubectl edit kjc default` | 是 |
| 开通事件总线 | EB API（`eb:CreateEventBus`） | 是 |
| 创建事件规则 | EB API（`eb:CreateRule`） | 是 |
| 配置告警通知 | 控制台操作（EB + 消息模板） | - |

## 操作步骤

### 检查 kubejarvis 组件版本

```bash
tccli tke DescribeAddon --ClusterId cls-xxxxxxxx --AddonName kubejarvis --region ap-guangzhou
```

```json
{
  "RequestId": "b0c1d2e3-f4a5-6789-0123-456789012345",
  "Addons": [
    {
      "AddonName": "kubejarvis",
      "AddonVersion": "1.0.8",
      "Status": "Running"
    }
  ]
}
```

如果版本低于 1.0.8，需要升级组件：

```bash
tccli tke UpdateAddon --ClusterId cls-xxxxxxxx --AddonName kubejarvis --region ap-guangzhou
```

```json
{
  "RequestId": "c1d2e3f4-a5b6-7890-1234-567890123456"
}
```

### 配置 kubejarvis 导出器

在 `exporters` 下添加 EB 导出器配置：

```bash
kubectl edit kjc default
```

```yaml
  exporters:
  - config: null
    name: my-eb-test
    requirements:
    - diagnostic: hosting-apiserver-etcd-object-counts-check
      levels:
      - warn
      - risk
      - serious
    type: eb
```

> 注意：kubectl 不可达时，kubejarvis 导出器可先在 YAML 中编辑，连通后 apply。

### 开通事件总线

如果是首次使用事件总线功能，需先开通：

1. 访问 [事件总线控制台](https://console.cloud.tencent.com/eb)
2. 按照指引开通服务

### 创建事件推送规则

通过 EB API 创建规则（以集群资源数量超限为例）：

```bash
tccli eb CreateRule \
  --EventBusId default \
  --RuleName cls-xxxxxxxx-resource-exceed \
  --EventPattern '{"source":["tke.cloud.tencent"],"type":["tke:ClusterResourceExceed"],"region":"ap-guangzhou"}' \
  --region ap-guangzhou
```

```json
{
  "RequestId": "d2e3f4a5-b6c7-8901-2345-678901234567",
  "RuleId": "rule-xxxxxxxx"
}
```

### 配置通知目标

在 EB 控制台为目标规则添加消息推送目标（短信、站内信等），配置通用通知模板和接收对象。

## 验证

### Control plane (tccli)

```bash
tccli tke DescribeAddon --ClusterId cls-xxxxxxxx --AddonName kubejarvis --region ap-guangzhou
```

```json
{
  "Addons": [],
  "AddonName": "<AddonName>",
  "AddonVersion": "<AddonVersion>",
  "RawValues": "<RawValues>",
  "Phase": "<Phase>",
  "Reason": "<Reason>",
  "CreateTime": "<CreateTime>",
  "RequestId": "<RequestId>"
}
```

确认 kubejarvis 版本 >= 1.0.8 且 Status 为 `Running`。

### Data plane (kubectl) — 参考

```bash
kubectl get kjc default -o yaml
```

```text
NAME  STATUS  AGE
...
```

确认 exporters 配置中包含 EB 导出器。

## 清理

### Control plane (EB)

在 [事件总线控制台](https://console.cloud.tencent.com/eb) 删除对应事件规则。

### Data plane (kubectl) — 参考

```bash
kubectl edit kjc default
# 删除添加的 EB exporter 配置
```

## 排障

| 现象 | 处理 |
|------|------|
| kubejarvis 版本低于 1.0.8 | 使用 `tccli tke UpdateAddon` 升级组件 |
| 未收到推送 | 确认已开通事件总线 EB 服务；检查事件规则的事件模式和推送目标配置 |
| 消息长度超限 | 短信限制 500 字，电话限制 350 字；建议同时配置多个渠道 |
| 跨境回调失败 | 跨境接口回调可能存在网络不稳定导致失败的情况，谨慎选择 |
| EB 事件规则不匹配 | 确认事件类型正确，云服务类型为"容器服务" |
| kubectl 不可达 | 托管集群公网端点可能受 CAM 策略限制（strategyId:240463971）；内网无 VPN 时不可达。kubejarvis 组件通过 `tccli tke DescribeAddon`/`UpdateAddon` 管理；EB 事件规则通过 `tccli eb` 命令管理；仅 kjc CRD 编辑依赖 kubectl |

## 下一步

- [告警管理](../告警管理/tccli 操作.md) — TCOP 告警策略配置
- [容器服务可观测体系概述](../容器服务可观测体系概述/tccli 操作.md) — 可观测体系整体架构

## 控制台替代

登录 [容器服务控制台](https://console.cloud.tencent.com/tke2) > 集群详情 > 组件管理（检查 kubejarvis）；[事件总线控制台](https://console.cloud.tencent.com/eb) 配置事件规则和推送目标。
