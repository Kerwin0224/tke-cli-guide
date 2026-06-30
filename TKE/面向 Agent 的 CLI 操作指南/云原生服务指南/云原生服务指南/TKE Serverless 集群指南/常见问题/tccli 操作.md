# 常见问题（tccli）

> 对照官方：常见问题(https://cloud.tencent.com/document/product/457/54778) · page_id `54778`

## 概述

本文汇总 TKE Serverless 集群的常见问题，从 CLI 操作视角给出可执行的排查命令和回答。涵盖集群运行、Pod 调度、负载均衡、镜像仓库、日志和监控六大方向。每条回答优先给出 CLI 命令，让你在终端即可诊断和解决，无需切换到控制台。

## 前置条件

- [环境准备](../../../../环境准备.md)
- 已安装 kubectl，并配好集群 kubeconfig
- 了解 TKE Serverless 集群基本概念（参见 [TKE Serverless 集群概述](../TKE%20Serverless%20集群概述/tccli%20操作.md)）

## 控制台与 CLI 参数映射

| 控制台操作 | CLI | 幂等 |
|-----------|-----|------|
| 查看集群信息 | `tccli tke DescribeEKSCluster --ClusterId <ClusterId>` | 是 |
| 查看 CLB 实例 | `tccli clb DescribeLoadBalancers --region <Region>` | 是 |
| 查看 TCR 实例 | `tccli tcr DescribeInstances --region <Region>` | 是 |
| 查看 Prometheus 实例 | `tccli monitor DescribePrometheusInstances` | 是 |
| 查看 CLS 日志主题 | `tccli cls DescribeTopics --region <Region>` | 是 |
| 查看子网信息（含可用 IP） | `tccli vpc DescribeSubnets --region <Region> --SubnetIds '["<SubnetId>"]'` | 是 |

## 操作步骤

### 1. TKE Serverless 集群相关问题

#### Q: Pod 规格如何确定？requests 和 limits 的关系？

Serverless 集群中，Pod 的资源计费和底层虚拟机规格由 **requests** 决定，limits 仅用于 cgroup 上限控制。建议 requests=limits 以避免资源争抢。

```bash
# 查看 Pod 的实际资源声明
kubectl get pod <PodName> -n <Namespace> -o json | jq '.spec.containers[].resources'
```

```text
NAME  STATUS  AGE
...
```

#### Q: 容器间网络如何互通？

Serverless 集群仅支持 VPC-CNI 网络模式，每个 Pod 从 VPC 子网获取独立弹性网卡 IP，Pod 间、Pod 与 CVM 间、Pod 与 VPC 内其他服务间均可直接互通。

```bash
# 查看集群网络配置
tccli tke DescribeEKSCluster --region ap-guangzhou --ClusterId cls-eks-a1b2c3d4 \
  --filter "ClusterNetworkSettings"

# 查看 Pod IP 和所在子网
kubectl get pod <PodName> -n <Namespace> -o json | jq '{podIP: .status.podIP, hostIP: .status.hostIP}'
```

```text
NAME  STATUS  AGE
...
```

#### Q: 子网 IP 不足怎么办？

Pod 直接占用 VPC 子网 IP，当子网 IP 枯竭时新 Pod 无法创建（报 `INSUFFICIENT_VPC_IP` 错误）。

```bash
# 检查子网可用 IP 数量
tccli vpc DescribeSubnets --region ap-guangzhou --SubnetIds '["subnet-k1l2m3n4"]' \
  --filter "TotalIpAddressCount,AvailableIpAddressCount"
```

```output
{
    "TotalCount": 1,
    "SubnetSet": [
        {
            "SubnetId": "subnet-k1l2m3n4",
            "SubnetName": "eks-pod-subnet",
            "VpcId": "vpc-9x8y7z6w",
            "CidrBlock": "172.16.0.0/20",
            "TotalIpAddressCount": 4091,
            "AvailableIpAddressCount": 127
        }
    ],
    "RequestId": "d4e5f6a7-b8c9-0123-defa-345678901234"
}
```

解决方案：
1. 扩展现有子网 CIDR（如果子网与 VPC CIDR 有扩展空间）
2. 新建子网并更新集群关联的子网列表
3. 清理已终止但 IP 未回收的 Pod

#### Q: 安全组如何配置？

Pod 的弹性网卡自动绑定集群安全组。安全组规则通过 VPC API 管理。

```bash
# 查看集群安全配置（获取安全组 ID）
tccli tke DescribeClusterSecurity --region ap-guangzhou --ClusterId cls-eks-a1b2c3d4

# 查看安全组规则
tccli vpc DescribeSecurityGroupPolicies --region ap-guangzhou --SecurityGroupId sg-example
```

```json
{
  "UserName": "<UserName>",
  "Password": "<Password>",
  "CertificationAuthority": "<CertificationAuthority>",
  "ClusterExternalEndpoint": "<ClusterExternalEndpoint>",
  "Domain": "<Domain>",
  "PgwEndpoint": "<PgwEndpoint>",
  "SecurityPolicy": [],
  "Kubeconfig": "<Kubeconfig>"
}
```

注意事项：
- Serverless 集群不支持 Pod 级别独立安全组，Pod 共享集群安全组
- 出入站规则需放通应用所需端口（如 Ingress 80/443、Service NodePort 范围 30000-32767）

#### Q: terminationMessagePath 和 terminationMessagePolicy 支持吗？

支持。可以在 Pod spec 中配置：

```bash
kubectl get pod <PodName> -n <Namespace> -o json | jq '.spec.containers[].terminationMessagePath, .spec.containers[].terminationMessagePolicy'
```

```text
NAME  STATUS  AGE
...
```

#### Q: 能设置 Host 参数吗（如 hostNetwork、hostPID）？

不支持。Serverless 集群没有传统 Node，Pod 之间通过虚拟机隔离，以下 host 级别参数不可用：
- `hostNetwork: true`
- `hostPID: true`
- `hostIPC: true`
- `hostAliases`

#### Q: 如何挂载 CFS/NFS？

Pod spec 中声明 NFS 类型的 Volume：

```bash
kubectl get pod <PodName> -n <Namespace> -o json | jq '.spec.volumes[] | select(.nfs)'

# 查看 CFS 文件系统列表
tccli cfs DescribeCfsFileSystems --region ap-guangzhou
```

```text
NAME  STATUS  AGE
...
```

#### Q: 镜像如何复用？

同一集群、同一地域内，相同镜像在多个 Pod 间可复用镜像缓存层，首次拉取后的 Pod 启动更快。镜像缓存在底层虚拟机镜像中有一定保留时间。

```bash
# 查看 Pod 使用的镜像
kubectl get pod <PodName> -n <Namespace> -o json | jq '.spec.containers[].image'
```

```text
NAME  STATUS  AGE
...
```

#### Q: NFS 挂载报错怎么办？

常见错误：
- `mount.nfs: access denied by server` — 确认 CFS 权限组允许 Pod 所在 VPC 访问
- `mount.nfs: connection timed out` — 确认 CFS 与 Pod 子网在同一 VPC，且安全组放通 NFS 端口 2049、111

```bash
# 查看 CFS 挂载点信息
tccli cfs DescribeMountTargets --region ap-guangzhou --FileSystemId cfs-example
```

#### Q: Pod 磁盘满了怎么办？

Serverless 集群为每个 Pod 分配临时存储（默认 20 GB），超出后 Pod 被 Evict 或 unable to write。临时存储不支持扩容。

```bash
# 查看 Pod 临时存储使用情况
kubectl exec <PodName> -n <Namespace> -- df -h /

# 清理日志和临时文件
kubectl exec <PodName> -n <Namespace> -- rm -rf /tmp/large-files
```

持久化数据应使用 CBS 或 CFS，不要依赖 Pod 临时存储。

#### Q: 端口 9100 是什么？

9100 是 EKS 底层数据面监控 Agent 使用的端口。如果你的应用监听 9100 端口可能与 Agent 冲突，建议避开。

```bash
# 检查 Pod 端口声明
kubectl get pod <PodName> -n <Namespace> -o json | jq '.spec.containers[].ports'
```

```text
NAME  STATUS  AGE
...
```

### 2. 负载均衡相关问题

#### Q: Ingress 如何触发 CLB 创建？

创建 Kubernetes Ingress 资源时，TKE 自动创建 CLB 实例（如未指定已有 CLB）。

```bash
# 查看 Ingress 及其对应的 CLB
kubectl get ingress -A

# 查看 CLB 实例列表
tccli clb DescribeLoadBalancers --region ap-guangzhou
```

```text
NAME  STATUS  AGE
...
```

#### Q: 怎样查看 Service/Ingress 关联的 CLB 实例？

```bash
# 从 Ingress 注解获取 CLB ID
kubectl get ingress <IngressName> -n <Namespace> -o json | \
  jq '.metadata.annotations["kubernetes.io/ingress.qcloud-loadbalance-id"]'

# 从 Service 注解获取 CLB ID
kubectl get svc <SvcName> -n <Namespace> -o json | \
  jq '.metadata.annotations["service.kubernetes.io/qcloud-loadbalancer-id"]'
```

```text
NAME  STATUS  AGE
...
```

获取 CLB ID 后查看详情：

```bash
tccli clb DescribeLoadBalancers --region ap-guangzhou --LoadBalancerIds '["lb-example"]'
```

#### Q: Service type=LoadBalancer 如何触发 CLB？

创建 `type: LoadBalancer` 的 Service 时，TKE Cloud Controller Manager 自动创建 CLB（公网/内网取决于注解配置）。

```bash
# 查看 LoadBalancer Service
kubectl get svc -A --field-selector spec.type=LoadBalancer

# 查看 Service 详情的 EXTERNAL-IP（即 CLB VIP）
kubectl get svc <SvcName> -n <Namespace>
```

```text
NAME  STATUS  AGE
...
```

#### Q: 为什么建议直接使用 CLB 直连 Pod？

VPC-CNI 模式下，CLB 可以直连 Pod IP（不再经过 NodePort），降低延迟并消除转发层。

配置方式：Service 添加注解 `service.kubernetes.io/qcloud-loadbalancer-backends-label`，指定后端为 Pod。

#### Q: ClusterIP Service 在 Serverless 集群中能正常工作吗？

可以。ClusterIP 通过 kube-proxy 的 iptables/IPVS 规则实现 Service 到 Pod 的负载均衡，与集群网络模式无关。

#### Q: 支持哪些 CLB 类型？

- 公网 CLB：`service.kubernetes.io/qcloud-loadbalancer-internet-charge-type`
- 内网 CLB：`service.kubernetes.io/qcloud-loadbalancer-internal-subnetid` 指定子网

```bash
# 查看内网 CLB
tccli clb DescribeLoadBalancers --region ap-guangzhou --LoadBalancerType INTERNAL
```

#### Q: 能否使用已有 CLB？

可以。在 Service 或 Ingress 注解中指定已有 CLB ID：

- Ingress: `kubernetes.io/ingress.existLbId: lb-xxxx`
- Service: `service.kubernetes.io/tke-existed-lbid: lb-xxxx`

#### Q: CLB 访问日志在哪里看？

CLB 访问日志存储在 CLS（日志服务）中，需提前开启日志投递。

```bash
# 查看 CLB 日志配置
tccli clb DescribeLoadBalancers --region ap-guangzhou --LoadBalancerIds '["lb-example"]' \
  --filter "LoadBalancerSet[].LogSetId,LoadBalancerSet[].LogTopicId"

# 查看 CLS 日志主题
tccli cls DescribeTopics --region ap-guangzhou --Filters '[{"Key":"topicId","Values":["<LogTopicId>"]}]'
```

#### Q: CLB 创建失败/不创建？

常见原因：
1. 子网 IP 不足 — 检查 `AvailableIpAddressCount`
2. 账户欠费 — 检查账户余额
3. VPC/子网不存在 — 检查 Ingress/Service 注解中的子网 ID

```bash
# 查看 Ingress 事件
kubectl describe ingress <IngressName> -n <Namespace> | grep -A 10 Events
```

#### Q: 多个 Service 能否共享一个 CLB？

可以。通过不同端口区分：

```yaml
# 同一 CLB 上的两个 Service 用不同 port
spec.ports[0].port: 80   # 第一个 Service
spec.ports[0].port: 8080 # 第二个 Service
```

注解 `service.kubernetes.io/qcloud-share-existed-lb: "true"` 允许已有 CLB 被多个 Service 复用。

#### Q: 如何通过 CLB VIP 直接访问 Pod？

获取 Service EXTERNAL-IP：

```bash
kubectl get svc -A --field-selector spec.type=LoadBalancer -o custom-columns=NAMESPACE:.metadata.namespace,NAME:.metadata.name,EXTERNAL-IP:.status.loadBalancer.ingress[0].ip,PORTS:.spec.ports[*].port
```

```text
NAME  STATUS  AGE
...
```

### 3. 超级节点相关问题

#### Q: 如何阻止 Pod 调度到超级节点？

在 Pod spec 中使用 `nodeSelector` 或 `nodeAffinity` 排除超级节点标签。超级节点通常带有 `node.kubernetes.io/instance-type: eklet` 标签。

```bash
# 查看超级节点标签
kubectl get nodes -l node.kubernetes.io/instance-type=eklet -o json | jq '.items[].metadata.labels'
```

```text
NAME  STATUS  AGE
...
```

#### Q: 如何禁用超级节点的自动调度？

TKE 默认将无 nodeSelector 的 Pod 调度到超级节点。如需关闭，在超级节点上添加 taint：

```bash
kubectl taint nodes -l node.kubernetes.io/instance-type=eklet dedicated=serverless:NoSchedule
```

#### Q: 如何手动将 Pod 调度到指定超级节点？

使用 `nodeName` 直接指定（在超级节点模式下不需要，Pod 由调度器自动处理）：

```bash
kubectl get pod <PodName> -n <Namespace> -o json | jq '.spec.nodeName'
```

#### Q: 如何强制 Pod 调度到超级节点？

添加 toleration 以忽略 taint：

```yaml
tolerations:
  - key: "serverless"
    operator: "Equal"
    value: "true"
    effect: "NoSchedule"
```

#### Q: 如何为超级节点上的 Pod 配置自定义 DNS？

在 Pod spec 中指定 `dnsPolicy` 和 `dnsConfig`：

```bash
kubectl get pod <PodName> -n <Namespace> -o json | jq '.spec.dnsPolicy, .spec.dnsConfig'
```

#

```json
{
  "TotalCount": 0,
  "Registries": [],
  "RegistryId": "<RegistryId>",
  "RegistryName": "<RegistryName>",
  "RegistryType": "<RegistryType>",
  "Status": "<Status>",
  "PublicDomain": "<PublicDomain>",
  "CreatedAt": "<CreatedAt>"
}
```## 4. 镜像仓库相关问题

#### Q: Serverless 集群如何集成 TCR？

TKE Serverless 集群与 TCR 企业版/个人版默认打通。Pod 可通过内网 VPC 拉取同地域 TCR 镜像，无需额外配置。

```bash
# 查看 TCR 实例
tccli tcr DescribeInstances --region ap-guangzhou

# 查看 TCR 实例关联的 VPC 访问入口
tccli tcr DescribeInternalEndpoints --region ap-guangzhou --RegistryId tcr-example
```

```json
{
  "RequestId": "..."
}
```

```output
{
    "AccessVpcSet": [
        {
            "VpcId": "vpc-9x8y7z6w",
            "SubnetId": "subnet-k1l2m3n4",
            "Status": "Connected",
            "AccessIp": "10.0.0.10",
            "EniLBIp": "172.16.1.100"
        }
    ],
    "TotalCount": 1,
    "RequestId": "e5f6a7b8-c9d0-1234-efab-456789012345"
}
```

#### Q: 能否使用自签名证书或 HTTP 镜像仓库？

可以，但需要配置。在 Pod 所在命名空间中添加 `imagePullSecret`，或者通过 TKE 控制台为集群配置自定义 Registry 的 CA 证书。

```bash
# 查看已有的 imagePullSecret
kubectl get secrets -A --field-selector type=kubernetes.io/dockerconfigjson

# 创建 imagePullSecret
kubectl create secret docker-registry my-registry \
  --docker-server=<RegistryUrl> \
  --docker-username=<Username> \
  --docker-password=<Password>
```

```text
NAME  STATUS  AGE
...
```

### 5. 日志采集相关问题

#### Q: Pod 的标准输出日志在哪里看？

可通过 kubectl 直接查看，或者配置 CLS 日志采集后从 CLS 控制台/API 检索。

```bash
# 实时查看 Pod 日志
kubectl logs -f <PodName> -n <Namespace>

# 查看已终止 Pod 日志
kubectl logs <PodName> -n <Namespace> --previous
```

```text
...log output...
```

#### Q: CLS 日志在哪里看？

配置日志采集规则后，日志投递到 CLS：

```bash
# 查看 CLS 日志主题
tccli cls DescribeTopics --region ap-guangzhou

# 搜索日志
tccli cls SearchLog --region ap-guangzhou \
  --TopicId <TopicId> \
  --From $(date -d '1 hour ago' +%s)000 \
  --To $(date +%s)000 \
  --Query "namespace:<Namespace> AND pod:<PodName>"
```

#### Q: 日志采集规则在哪里配置？

通过 TKE 控制台或 LogConfig CRD 配置：

```bash
# 查看 LogConfig 资源（如果集群安装了日志采集组件）
kubectl get logconfigs -A
```

```text
NAME  STATUS  AGE
...
```

### 6. Prometheus 监控相关问题

#### Q: 如何集成腾讯云 Prometheus 监控服务？

在 TKE 控制台为 Serverless 集群关联 Prometheus 实例，或通过 API 操作。

```bash
# 查看 Prometheus 实例
tccli monitor DescribePrometheusInstances

# 查看与集群关联的抓取配置
tccli monitor DescribePrometheusScrapeJobs --InstanceId <InstanceId>
```

```output
{
    "InstanceSet": [
        {
            "InstanceId": "prom-example",
            "InstanceName": "eks-monitoring",
            "InstanceState": 2,
            "Region": "ap-guangzhou",
            "VpcId": "vpc-9x8y7z6w",
            "SubnetId": "subnet-k1l2m3n4",
            "DataRetentionTime": 15,
            "GrafanaURL": "http://grafana.example.com"
        }
    ],
    "TotalCount": 1,
    "RequestId": "f6a7b8c9-d0e1-2345-fabc-567890123456"
}
```

#### Q: 可以使用自建 Prometheus 吗？

可以。通过 ServiceMonitor/PodMonitor CRD 抓取 Pod 指标，但需要 Prometheus 与 Serverless 集群网络互通（同 VPC 内或通过云联网/对等连接）。

```bash
# 查看 ServiceMonitor
kubectl get servicemonitors -A

# 验证 Prometheus 能否访问 Pod metrics 端点
kubectl run curl-test --image=curlimages/curl --rm -it -

```json
{
  "TotalCount": 0,
  "Clusters": [],
  "ClusterId": "<ClusterId>",
  "ClusterName": "<ClusterName>",
  "VpcId": "<VpcId>",
  "SubnetIds": [],
  "K8SVersion": "<K8SVersion>",
  "Status": "<Status>"
}
```-restart=Never -- \
  curl -s http://<PodIP>:<MetricsPort>/metrics
```

## 验证

```bash
# 验证集群可访问
tccli tke DescribeEKSClusters --region ap-guangzhou --filter "ClusterStatus=Running"

# 验证 CLB 可用
tccli clb DescribeLoadBalancers --region ap-guangzhou --LoadBalancerType OPEN \
  --filter "LoadBalancerSet[0].LoadBalancerId,LoadBalancerSet[0].LoadBalancerVips"

# 验证 TCR 内网连接
tccli tcr DescribeInternalEndpoints --region ap-guangzhou --RegistryId tcr-example

# 验证 Prometheus 运行中
tccli monitor DescribePrometheusInstances --filter "InstanceState=2"
```

## 清理

无需清理。本页为 FAQ 参考文档，不涉及资源创建。

## 排障

| 现象 | 处理命令 |
|------|---------|
| kubectl 无法连接到集群 | `tccli tke DescribeClusterKubeconfig --ClusterId <ClusterId>` 重新获取证书，检查 API Server 域名可达性 |
| Pod 一直 Pending | `kubectl describe pod <PodName> -n <Namespace>` 查看 Events，常见原因：子网 IP 不足、资源规格超限 |
| CLB 未创建 | `kubectl describe ingress <IngressName> -n <Namespace>` 查看 Ingress Events |
| 镜像拉取失败 | `kubectl describe pod <PodName> -n <Namespace>` 查看 imagePull 相关 Events，确认 imagePullSecret 配置正确 |
| 日志采集不生效 | 检查是否安装了 CLS 日志采集组件，确认 LogConfig/Pod 标签匹配 |
| 监控数据缺失 | `tccli monitor DescribePrometheusScrapeJobs --InstanceId <InstanceId>` 检查抓取任务状态 |

## 下一步

- [TKE Serverless 集群概述](../TKE%20Serverless%20集群概述/tccli%20操作.md)
- [TKE Serverless 集群管理](../TKE Serverless 集群管理/注意事项/tccli 操作.md)
- [Kubernetes 对象管理（Serverless）](../Kubernetes 对象管理/工作负载管理/tccli 操作.md)
- [购买 TKE Serverless 集群](../购买 TKE Serverless 集群/地域和可用区/tccli 操作.md)

## 控制台替代

控制台：容器服务 → 集群 → 选择 Serverless 集群 → 各功能标签页。常见问题的排查入口分布在：工作负载 Events、CLB 详情、日志检索、监控 Dashboard。
