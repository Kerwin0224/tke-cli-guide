# NginxIngress 日志配置（tccli）

> 对照官方：[NginxIngress 日志配置](https://cloud.tencent.com/document/product/457/50505) · page_id `50505`

## 概述

> **注意：** NginxIngress 扩展组件已停止更新，详情见 [NginxIngress 扩展组件停止更新公告](https://cloud.tencent.com/document/product/457/108517)。

TKE 通过集成日志服务 CLS，为 NginxIngress 提供全套日志采集和消费能力，支持 Nginx Controller 日志、AccessLog 和 ErrorLog 的分类采集与分析。

## 前置条件

- [环境准备](../../../../../环境准备.md)
- 已 [连接集群](../../../../../集群配置/集群管理/连接集群/tccli 操作.md)，kubectl 可正常访问 APIServer
- 已在 TKE 控制台 **功能管理** 中开启日志采集
- 已 [安装 NginxIngress 实例](../安装%20NginxIngress%20实例/tccli 操作.md)

## 控制台与 CLI 参数映射

| Console 操作 | CLI 等价操作 | 幂等 | 说明 |
|---|---|---|---|------|
| 开启日志采集 | 控制台 **运维 > 日志配置 > 重新设置** | 否 | 配置 CLS 日志集和主题 |
| 查看日志 | CLS 控制台检索 | 是 | 通过仪表盘分析 |
| 自定义日志规则 | `kubectl edit logconfig <name>` | 否 | 修改 LogConfig CRD |
| 自定义索引 | CLS 控制台 **日志主题 > 索引配置** | 是(apply) | 配置键值索引 |

## 操作步骤

### NginxIngress 日志类型

| 日志类型 | 重要性 | 说明 |
|---|---|---|
| Nginx Controller 日志 | 重要 | 控制面日志，记录控制面修改，用于排障 |
| AccessLog | 重要 | 数据面日志，记录七层请求信息，用于数据分析审计 |
| ErrorLog | 一般 | Nginx 内部错误日志 |

> 默认配置下 AccessLog 和 Controller 日志混合输出到标准输出流，TKE 通过区分日志路径分别采集。

### 采集日志步骤

通过控制台 **服务与路由 > NginxIngress > 实例详情 > 运维 > 日志配置 > 重新设置** 配置 CLS 日志集。

### 采集日志指标（LogConfig CRD）

```yaml
apiVersion: cls.cloud.tencent.com/v1
kind: LogConfig
metadata:
  name: nginx-ingress-test
spec:
  clsDetail:
    extractRule:
      beginningRegex: (\\S+)\\s-\\s(\\S+)\\s\\[(\\S+)\\]\\s(\\S+)\\s\"(\\w+)\\s(\\S+)\\s([^\\\"]+)\"\\s(\\S+)\\s(\\S+)\\s\"([^\"]*)\"\\s\"([^\"]*)\"\\s(\\S+)\\s(\\S+)\\s\\[([^\\]]*)\\]\\s\\[([^\\]]*)\\]\\s\\[([^\\]]*)\\]\\s\\[([^\\]]*)\\]\\s\\[([^\\]]*)\\]\\s\\[([^\\]]*)\\]\\s(\\S+)
      keys:
      - remote_addr
      - remote_user
      - time_local
      - timestamp
      - method
      - url
      - version
      - status
      - body_bytes_sent
      - http_referer
      - http_user_agent
      - request_length
      - request_time
      - proxy_upstream_name
      - proxy_alternative_upstream_name
      - upstream_addr
      - upstream_response_length
      - upstream_response_time
      - upstream_status
      - req_id
      logRegex: ...
    logType: fullregex_log
    topicId: <topic-id>
  inputDetail:
    containerFile:
      container: controller
      filePattern: nginx_access.log
      logPath: /var/log/nginx
      namespace: default
      workload:
        kind: deployment
        name: nginx-ingress-nginx-controller
    type: container_file
```

### NginxIngress 日志仪表盘

开启日志采集后，可在 CLS 控制台使用预置的 [Nginx 访问日志分析](https://cloud.tencent.com/document/product/614/61404) 仪表盘。

## 验证

### 数据面（kubectl）

```bash
kubectl get logconfig nginx-ingress-test -o yaml
```

```text
NAME  STATUS  AGE
...
```

在 CLS 控制台确认日志主题已收到日志。

## 清理

### 数据面（kubectl）

在控制台 **运维 > 日志配置** 中关闭日志采集。

## 排障

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `Unable to connect to the server` | `kubectl cluster-info` | 公网端点被 CAM 策略 strategyId:240463971 以 `tke:clusterExtranetEndpoint=true` 条件拒绝 | 通过 IOA/VPN 连接内网，或使用同 VPC CVM 执行 kubectl |
| 日志未采集 | `kubectl get logconfig` | TKE 功能管理中已开启日志采集 | 确认 TKE 功能管理中已开启日志采集 |
| 日志格式异常 | `access-log-path` | 修改 ConfigMap 中 `access-log-path`、`error-log-path`、`log-fo... | 请勿修改 ConfigMap 中 `access-log-path`、`error-log-path`、`log-format-upstream` |
| 自定义日志规则 | `kubectl get logconfig` | 参考 [NginxIngress 自定义日志](https://cloud.tencent.com/documen... | 参考 [NginxIngress 自定义日志](https://cloud.tencent.com/document/product/457/77840) |

## 下一步

- [安装 NginxIngress 实例](../安装%20NginxIngress%20实例/tccli 操作.md)
- [使用 NginxIngress 对象接入集群外部流量](../使用%20NginxIngress%20对象接入集群外部流量/tccli 操作.md)
- [通过 Terraform 安装 Nginx 插件和实例](../通过%20Terraform%20安装%20Nginx%20插件和实例/tccli 操作.md)

## 控制台替代

在控制台 **NginxIngress 实例详情 > 运维 > 日志配置** 中开启和管理日志采集。
