---
doc_type: FAQ
---
> 官方文档：[节点概述](https://cloud.tencent.com/document/product/457/32201) · [注册节点价格和配额说明](https://cloud.tencent.com/document/product/457/79747)
>
> 配额：节点规格与集群配额须匹配。[配额说明](https://cloud.tencent.com/document/product/457/9087)
>
> ⚠️ **高危操作**：容器运行时不兼容致注册失败；脚本版本过期致节点无法注册；安全组策略过严致节点无法通信；节点资源不足致 Pod 调度失败。[常见高危操作](https://cloud.tencent.com/document/product/457/39539)

# 注册节点常见问题

> 注册节点的高频疑问与处理指引。更多背景见[注册节点概述](overview.md)。

## 集群类

### 注册节点特性是否支持独立部署集群？

支持。你可以在托管形态与独立部署形态的集群中使用注册节点特性。

### 使用注册节点特性，为什么要求集群内必须要存在云上节点？

注册节点所在网络与 VPC 网络存在差异，集群内部分系统组件必须运行在云上节点，因此当前要求集群内必须存在云上节点。

## 节点类

### 注册节点和云上节点在能力上有哪些差异？

注册节点与云上节点在计费、运维边界与部分能力上不同（如公网版不支持 CLB 与 Prometheus）。能力对比见[注册节点与云上节点能力对比](https://cloud.tencent.com/document/product/457/57916)。

### 注册节点支持哪些操作系统？

当前注册节点的操作系统仅支持 TencentOS Server 3.1 与 TencentOS Server 2.4（TK4）。其他操作系统会导致注册失败。

### 节点上由于 docker、containerd 相关软件导致添加节点失败，如何处理？

使用下载的注册脚本执行清理指令后再添加：

```bash
./add2tkectl-cls-<CLUSTER_ID>-np-<NODEPOOL_ID> clear
```

### 注册节点脚本安装过程中报错中断如何处理？

常见报错与处理：

- **"nvidia nv_driver not installed"**：NVIDIA 驱动未安装。执行 `clear` 清理环境，安装 NVIDIA Tesla 驱动并 `nvidia-smi` 验证后，重新执行 `install`。
- **"Install gpu toolkit failed!"**：nvidia toolkit 安装失败。执行 `clear` 清理环境后重新执行 `install`。
- **"can not get nodes node-xxx gpu capacity after 60s"**：节点 GPU 容器能力初始化失败。执行 `clear` 清理环境后重新执行 `install`。

```bash
./add2tkectl-cls-<CLUSTER_ID>-np-<NODEPOOL_ID> clear
./add2tkectl-cls-<CLUSTER_ID>-np-<NODEPOOL_ID> install
```

## 网络、流量接入类

### 注册节点的容器如何对外暴露服务？

基于腾讯云负载均衡 CLB 提供四层与七层流量接入方案，详见[流量接入](traffic-access.md)。注意：当前仅注册节点（专线版）支持接入 CLB，公网版暂不支持。

## 运维类

### 注册节点的日志如何接入日志服务 CLS？

TKE 集群接入 CLS 后默认支持集群中的注册节点，无需特殊配置。注册节点默认使用内网方式投递日志（占用专线带宽）。

如需公网投递日志：修改 `kube-system` 命名空间下 `externalnode-config` ConfigMap 的 `clsPushMethod` 值——`intranet`（默认内网）/ `public`（公网，需注册节点具备公网访问能力），再重建 `kube-system` 下 `tke-log-agent` DaemonSet 管理的 Pod 使配置生效。

### 注册节点如何接入 Prometheus 监控服务？

TKE 集群 Prometheus 监控服务默认支持集群中的注册节点，无需特殊配置。注意：当前仅注册节点（专线版）支持接入 Prometheus，公网版暂不支持。

### Cilium-Overlay 模式下如何创建 admission webhook？

集群使用 Cilium-Overlay 时，若注册节点上部署 admission webhook 组件，apiserver（托管在 TKE meta 集群，不在用户 VPC overlay 网络）无法通过 webhook 的 Service 访问其 Pod IP，导致访问失败。处理方式：将 webhook 的网络模式设置为 `HostNetwork`。
