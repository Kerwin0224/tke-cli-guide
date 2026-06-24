# 应用性能管理（tccli）

> 对照官方：[应用性能管理](https://cloud.tencent.com/document/product/457/118520) · page_id `118520`

## 概述

腾讯云应用性能管理（APM）支持与 TKE 集成，实现对 Kubernetes 中运行业务应用的链路追踪和性能监控。通过部署 APM Java Agent / Go Agent 等方式，可以无侵入或低侵入地采集应用性能数据，并在 APM 控制台查看调用链路、错误分析和性能瓶颈。

核心操作流程：
1. 在 APM 控制台创建 APM 实例（或通过 `apm:CreateApmInstance`）
2. 在 TKE 工作负载中配置 APM Agent（Sidecar / Init Container / 环境变量注入）
3. 通过 kubectl patch 或 Deployment 修改注入 Agent 配置

## 前置条件

- `tccli` 已配置凭证
- kubectl 不可达：演示集群 cls-xxxxxxxx 为托管集群，公网端点可能受 CAM 策略限制（strategyId:240463971）；内网无 VPN 时不可达。以下 kubectl 命令作为文档参考。
- 已开通腾讯云 APM 服务
- 集群 cls-xxxxxxxx 中已有运行业务应用的工作负载

```bash
tccli tke DescribeClusters --ClusterIds '["cls-xxxxxxxx"]' --region ap-guangzhou
```

```json
{
  "RequestId": "e.g.-a1b2c3d4-5678-90ab-cdef-0123456789ab",
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
|-----------|-----|:--:|
| 查看 APM 实例列表 | `tccli apm DescribeApmInstances --region ap-guangzhou` | 是 |
| 创建 APM 实例 | `tccli apm CreateApmInstance --region ap-guangzhou` | 否 |
| 查看 APM 实例详情 | `tccli apm DescribeApmInstanceDetail --InstanceId <InstanceId> --region ap-guangzhou` | 是 |
| 注入 APM Agent 到工作负载 | `kubectl patch deployment <DeployName> -n <NS>` 或修改 Deployment YAML | 否 |
| 查看工作负载运行状态 | `kubectl get pods -n <NS> -l app=<Label>` | 是 |
| 查看链路追踪数据 | 通过 APM 控制台（无直接 CLI 查询方法） | — |

## 操作步骤

### 1. 查看已有 APM 实例

```bash
tccli apm DescribeApmInstances --region ap-guangzhou
```

```json
{
  "RequestId": "e.g.-c5d6e7f8-9012-3f45-6789-012abcdef345",
  "TotalCount": 1,
  "Instances": [
    {
      "InstanceId": "apm-abc123xxx",
      "Name": "tke-apm-instance",
      "Region": "ap-guangzhou",
      "Description": "TKE cluster tracing",
      "Status": "运行中",
      "Tags": null
    }
  ]
}
```

### 2. 创建 APM 实例（如无已有实例）

```bash
tccli apm CreateApmInstance \
  --Name "tke-apm-demo" \
  --Description "TKE APM instance for cls-xxxxxxxx" \
  --region ap-guangzhou
```

```json
{
  "RequestId": "e.g.-d6e7f8a9-0123-4f56-7890-12bcdef34567",
  "InstanceId": "apm-def789xxx"
}
```

### 3. 查看 APM 实例详情（获取接入 Token）

```bash
tccli apm DescribeApmInstanceDetail \
  --InstanceId "apm-abc123xxx" \
  --region ap-guangzhou
```

```json
{
  "RequestId": "e.g.-e7f8a9b0-1234-5678-9012-bcdef3456789",
  "InstanceId": "apm-abc123xxx",
  "Name": "tke-apm-instance",
  "Status": "运行中",
  "Region": "ap-guangzhou",
  "Token": "abc123def456"
}
```

记录 `Token` 值，后续注入 APM Agent 时使用。

### 4. 将 APM Agent 注入工作负载（kubectl — 文档参考）

APM Agent 通过环境变量方式注入业务容器，无需修改镜像。

**方式一：kubectl patch（推荐用于已有 Deployment）**

以 Java 应用为例，设置 APM Agent 环境变量：

```bash
kubectl patch deployment <DeployName> -n <Namespace> --type='json' \
  -p='[
    {
      "op": "add",
      "path": "/spec/template/spec/containers/0/env/-",
      "value": {
        "name": "JAVA_TOOL_OPTIONS",
        "value": "-javaagent:/apm-agent/opentelemetry-javaagent.jar"
      }
    },
    {
      "op": "add",
      "path": "/spec/template/spec/containers/0/env/-",
      "value": {
        "name": "OTEL_EXPORTER_OTLP_ENDPOINT",
        "value": "https://ap-guangzhou.apm.tencentcs.com:4317"
      }
    },
    {
      "op": "add",
      "path": "/spec/template/spec/containers/0/env/-",
      "value": {
        "name": "OTEL_RESOURCE_ATTRIBUTES",
        "value": "service.name=<ServiceName>,token=abc123def456"
      }
    }
  ]'
```

```text
deployment.apps/<DeployName> patched
```

**方式二：直接在 Deployment YAML 中添加环境变量**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
  namespace: default
spec:
  template:
    spec:
      containers:
        - name: my-app
          image: my-app:latest
          env:
            - name: JAVA_TOOL_OPTIONS
              value: "-javaagent:/apm-agent/opentelemetry-javaagent.jar"
            - name: OTEL_EXPORTER_OTLP_ENDPOINT
              value: "https://ap-guangzhou.apm.tencentcs.com:4317"
            - name: OTEL_RESOURCE_ATTRIBUTES
              value: "service.name=my-app,token=abc123def456"
```

```bash
kubectl apply -f deployment-with-apm.yaml
```

```text
# command executed successfully
```

### 5. 验证工作负载更新（kubectl — 文档参考）

```bash
kubectl get pods -n <Namespace> -l app=<Label> -w
# 查看 Pod 环境变量
kubectl describe pod <PodName> -n <Namespace> | grep -A 20 "Environment:"
```

```text
NAME  STATUS  AGE
...
```

### 6. 在 APM 控制台查看链路数据

注入 Agent 后，应用产生的链路数据自动上报至 APM。在 APM 控制台（[ap-guangzhou 地域](https://console.cloud.tencent.com/apm)）查看：

- **调用链路**：追踪请求在各服务间的流转路径
- **错误分析**：定位异常请求和错误率
- **性能瓶颈**：查看慢请求分布和耗时热点

## 验证

### 控制面验证

```bash
tccli apm DescribeApmInstances --region ap-guangzhou && \
tccli apm DescribeApmInstanceDetail \
  --InstanceId "apm-abc123xxx" \
  --region ap-guangzhou
```

预期：APM 实例状态为 `运行中`，Token 非空。

### 数据面验证（kubectl — 文档参考）

```bash
kubectl get pods -n <Namespace> -l app=<Label>
kubectl logs <PodName> -n <Namespace> | grep -i "opentelemetry\|apm\|otlp"
```

```text
NAME  STATUS  AGE
...
```

预期：Pod 处于 Running 状态；日志中无 APM Agent 连接错误。

### APM 控制台验证

登录 [APM 控制台](https://console.cloud.tencent.com/apm)，在「调用查询」中确认已有链路数据上报。

## 清理

### 移除工作负载中的 APM 环境变量（kubectl — 文档参考）

```bash
kubectl patch deployment <DeployName> -n <Namespace> --type='json' \
  -p='[{"op":"remove","path":"/spec/template/spec/containers/0/env"}]'
```

重新 Apply 原始的不含 APM 配置的 Deployment YAML。

### 删除 APM 实例（谨慎操作）

```bash
# APM 实例删除需通过控制台操作；CLI 确认无直接 Delete 命令。
# 在控制台中释放实例：https://console.cloud.tencent.com/apm
```

## 排障

| 现象 | 处理 |
|------|------|
| `DescribeApmInstances` 返回空 | 当前地域（ap-guangzhou）无 APM 实例；先执行 `CreateApmInstance` 创建 |
| APM 控制台无链路数据 | 确认 Agent 环境变量正确注入；检查 Pod 是否已重建（kubectl rollout restart）；确认 Token 正确 |
| Agent 连接被拒绝（Connection Refused） | 检查集群是否可访问 APM 数据上报地址（ap-guangzhou.apm.tencentcs.com）；确认安全组出站规则允许 4317 端口 |
| `kubectl patch` 报错 `the server rejected our request` | 检查 RBAC 权限，确认 ServiceAccount 有 patch Deployment 的权限 |
| APM Agent 影响应用启动 | 检查 JAVA_TOOL_OPTIONS 参数是否正确；使用 Init Container 预下载 Agent jar 到共享卷可减少启动时间 |
| 不同语言的应用接入 | Go: `otel` SDK + `OTEL_EXPORTER_OTLP_ENDPOINT`；Python: `opentelemetry-distro`；Node.js: `@opentelemetry/sdk-node` |

## 下一步

- [查看 TKE 集群控制面组件监控](../控制面组件监控/查看 TKE 集群控制面组件监控/tccli 操作.md) — 控制面组件监控
- [监控告警概述](../基础监控与告警/监控告警概述/tccli 操作.md) — TKE 基础监控概念
- [APM 官方文档](https://cloud.tencent.com/document/product/1463) — APM 产品完整文档

## 控制台替代

登录 [应用性能管理控制台](https://console.cloud.tencent.com/apm) 查看链路追踪数据、错误分析和性能统计。
