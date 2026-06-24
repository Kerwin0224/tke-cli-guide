# 日志组件版本升级（tccli）

> 对照官方：[日志组件版本升级](https://cloud.tencent.com/document/product/457/48425) · page_id `48425`

## 概述

容器服务运维中心提供日志组件（tke-log-agent）版本升级功能。开启日志采集后，可在控制台查看当前组件版本并进行手动升级。升级属于不可逆操作，仅支持向上升级，默认升级至当前最新版本。升级时自动升级配套的 LogListener 和 Provisioner 版本，并更新集群内 CRD 资源。

## 前置条件

- `tccli` 已配置凭证
- kubectl 不可达：演示集群 cls-xxxxxxxx 为托管集群，公网端点可能受 CAM 策略限制（strategyId:240463971）；内网无 VPN 时不可达。以下 kubectl 命令作为文档参考。
- 已开启日志采集功能（`tccli tke InstallLogAgent`）

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
| 查看组件版本 | `tccli tke DescribeAddon --AddonName tke-log-agent` | 是 |
| 升级组件版本 | `tccli tke UpdateAddon --AddonName tke-log-agent` | 否（不可逆） |

## 操作步骤

### 查看当前日志组件版本

```bash
tccli tke DescribeAddon --ClusterId cls-xxxxxxxx --AddonName tke-log-agent --region ap-guangzhou
```

```json
{
  "RequestId": "b2c3d4e5-f6a7-8901-2345-678901234567",
  "Addons": [
    {
      "AddonName": "tke-log-agent",
      "AddonVersion": "1.0.5",
      "Status": "Running"
    }
  ]
}
```

### 升级日志组件

升级默认至当前最新版本。如需指定版本：

```bash
tccli tke UpdateAddon \
  --ClusterId cls-xxxxxxxx \
  --AddonName tke-log-agent \
  --AddonVersion 1.1.0 \
  --region ap-guangzhou
```

```json
{
  "RequestId": "c3d4e5f6-a7b8-9012-3456-789012345678"
}
```

### 确认升级结果

```bash
tccli tke DescribeAddon --ClusterId cls-xxxxxxxx --AddonName tke-log-agent --region ap-guangzhou
```

```json
{
  "RequestId": "d4e5f6a7-b8c9-0123-4567-890123456789",
  "Addons": [
    {
      "AddonName": "tke-log-agent",
      "AddonVersion": "1.1.0",
      "Status": "Running"
    }
  ]
}
```

### 查看版本迭代记录

版本详情参见 [日志组件版本迭代记录](https://cloud.tencent.com/document/product/457/67279)。

## 验证

### Control plane (tccli)

```bash
tccli tke DescribeAddon --ClusterId cls-xxxxxxxx --AddonName tke-log-agent --region ap-guangzhou
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

确认 `AddonVersion` 已更新且 `Status` 为 `Running`。

## 清理

升级不可逆，无法回滚。如需回退，需联系技术支持。

## 排障

| 现象 | 处理 |
|------|------|
| 升级后采集异常 | 升级自动更新 LogListener 和 Provisioner 以及 CRD 资源；如异常请检查 Pod 日志 |
| 无"组件可升级"提示 | 当前版本已是最新版本 |
| 升级失败 | 升级不可逆，失败后请查看 DescribeAddon 输出的 Status 判断状态 |
| 升级影响业务 | 升级过程中 DaemonSet 会滚动更新，短暂影响日志采集 |
| kubectl 不可达 | 托管集群公网端点可能受 CAM 策略限制（strategyId:240463971）；内网无 VPN 时不可达；组件版本管理通过 `tccli tke DescribeAddon`/`UpdateAddon` 操作，不依赖 kubectl |

## 下一步

- [日志采集概述](../日志采集概述/tccli 操作.md) — 日志采集基本概念与开启/关闭
- [采集容器日志到 CLS](../采集容器日志到 CLS/tccli 操作.md) — 创建日志采集规则

## 控制台替代

登录 [容器服务控制台](https://console.cloud.tencent.com/tke2) > 运维中心 > 运维功能管理 > 选择集群 > 组件可升级 > tke-log-agent > 升级。
