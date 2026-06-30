# 边缘集群迁移至 TKE 标准集群注册节点公网版（tccli）

> 对照官方：[边缘集群迁移至 TKE 标准集群注册节点公网版](https://cloud.tencent.com/document/product/457/110447) · page_id `110447`

## 概述

将边缘集群中的节点和工作负载迁移至 TKE 标准集群，采用注册节点（公网版）方式将边缘节点纳入 TKE 标准集群管理。此方案适用于边缘节点与腾讯云控制面之间通过公网可达的场景，无需搭建专线或 VPN 即可完成节点注册。

> **注意**：本指南中所有涉及 `kubectl` 的操作均假设 kubectl 已配置并可正常连接目标集群。如 kubectl 不可达（例如边缘节点侧网络受限），请参见[排障](#排障)中的网络连通性排查指引。

**迁移整体流程：**

1. 在腾讯云创建或确认一个 TKE 标准集群作为目标集群
2. 开启目标集群的外部节点支持
3. 获取节点注册脚本
4. 在边缘节点上执行注册脚本，将边缘节点注册到标准集群
5. 验证节点注册状态
6. 将工作负载从边缘集群迁移至标准集群的注册节点上

---

## 控制台与 CLI 参数映射

| 控制台操作 | tccli 命令 | 关键参数 | 参数说明 | 幂等性 |
|---|---|---|---|---|
| 创建 TKE 标准集群 | `tccli tke CreateCluster` | `--ClusterType "MANAGED_CLUSTER"` | 创建托管集群作为目标 | 是（同名已存在则复用） |
| 开启外部节点支持 | `tccli tke EnableExternalNodeSupport` | `--ClusterId`, `--ClusterExternalConfig` | 为目标集群开启注册节点功能 | 是（重复调用无副作用） |
| 查看外部节点注册配置 | `tccli tke DescribeExternalNodeSupportConfig` | `--ClusterId` | 获取注册命令/脚本 | 是（只读） |
| 查看集群详情 | `tccli tke DescribeClusters` | `--ClusterIds` | 确认集群状态 | 是（只读） |
| 查看集群节点列表 | `tccli tke DescribeClusterInstances` | `--ClusterId` | 确认节点注册状态 | 是（只读） |
| 删除注册节点 | `tccli tke DeleteClusterInstances` | `--ClusterId`, `--InstanceIds` | 从标准集群移除注册节点 | 否（删除后不可恢复） |

---

## 前置条件

- 已完成[环境准备](../../环境准备.md)
- tccli 已配置凭据和地域

## 操作步骤

### 步骤 1：创建或确认 TKE 标准集群（控制面）

#### 选择依据

- 如果已有可用的 TKE 标准集群（版本 >= 1.20），可直接复用，跳至步骤 2
- 如果需新建集群，选择与边缘节点地域相同的 REGION 以降低延迟
- 集群网络模式选择 VPC-CNI 或 GlobalRouter 均可，注册节点不受此限制

#### 最小配置

**查询已有集群：**

```bash
tccli tke DescribeClusters --region <Region> --ClusterIds '["CLUSTER_ID"]'
```

输出示例：

```json
{
    "TotalCount": 1,
    "Clusters": [
        {
            "ClusterId": "CLUSTER_ID",
            "ClusterName": "my-target-cluster",
            "ClusterStatus": "Running",
            "ClusterVersion": "1.28.3",
            "ClusterType": "MANAGED_CLUSTER"
        }
    ]
}
```

如果 `TotalCount` 为 0 或集群不存在，执行以下命令创建新集群：

```bash
tccli tke CreateCluster \
    --region <Region> \
    --ClusterType "MANAGED_CLUSTER" \
    --ClusterCIDRSettings '{"ClusterCIDR":"10.0.0.0/16"}' \
    --ClusterBasicSettings '{"ClusterName":"edge-migration-target","ClusterVersion":"1.28.3","VpcId":"VPC_ID"}'
```

输出示例：

```json
{
    "ClusterId": "CLUSTER_ID",
    "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

**确认集群处于 Running 状态：**

```bash
tccli tke DescribeClusters --region <Region> --ClusterIds '["CLUSTER_ID"]'
```

输出示例：

```json
{
    "TotalCount": 1,
    "Clusters": [
        {
            "ClusterId": "CLUSTER_ID",
            "ClusterStatus": "Running"
        }
    ]
}
```

> 等待 `ClusterStatus` 变为 `Running` 后再继续后续步骤。

#### 增强配置

- **集群版本**：建议选择 >= 1.28 版本以获得更好的外部节点特性支持
- **VPC 规划**：确保 VPC 的 CIDR 与边缘节点所在网络无冲突
- **集群标签**：通过 `--ClusterBasicSettings '{"ClusterName":"...","Labels":[{"Name":"env","Value":"edge-migration"}]}'` 添加业务标签以便后续管理

---

### 步骤 2：开启集群外部节点支持（控制面）

#### 选择依据

- 此操作为一次性配置，开启后集群即具备纳管外部节点的能力
- 操作幂等，重复调用不会产生副作用

#### 最小配置

```bash
tccli tke EnableExternalNodeSupport \
    --region <Region> \
    --ClusterId CLUSTER_ID \
    --ClusterExternalConfig '{"Enabled":true}'
```

输出示例：

```json
{
    "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

> 无错误返回即表示外部节点支持已开启。

#### 增强配置

- 如需要限制可注册的外部节点网络范围，可在 `ClusterExternalConfig` 中配置 `NetworkPolicy` 参数
- 建议在开启后等待约 30 秒，确保配置在控制面完全生效

---

### 步骤 3：获取注册脚本（控制面）

#### 选择依据

- 注册脚本是将边缘节点纳入标准集群管理的核心凭证，包含 CA 证书、API Server 地址及节点注册命令
- 该接口返回的数据面注册信息需安全保存，避免泄露

#### 最小配置

```bash
tccli tke DescribeExternalNodeSupportConfig \
    --region <Region> \
    --ClusterId CLUSTER_ID
```

输出示例：

```json
{
    "ClusterCIDR": "10.0.0.0/16",
    "NetworkType": "GR",
    "SubnetId": "",
    "ClusterExternalEndpoint": "https://CLUSTER_ID.ccs.tencent-cloud.com",
    "KubeConfig": "YXBpVmVyc2lvbjogdjEKY2x1c3RlcnM6Ci0gY2x1c3RlcjoK...",
    "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

> `KubeConfig` 为 Base64 编码的 kubeconfig 内容，解码后可用于在边缘节点上生成注册命令。

**提取注册命令模式：**

若直接 API 未返回完整的注册命令字符串，可先使用 `--generate-cli-skeleton` 查看接口完整参数结构：

```bash
tccli tke DescribeExternalNodeSupportConfig --generate-cli-skeleton
```

输出示例：

```json
{
    "ClusterId": "CLUSTER_ID",
    "NetworkType": "GR"
}
```

根据返回的 `ClusterExternalEndpoint` 和 `KubeConfig`，注册脚本通常包含以下核心内容：

1. 将 `KubeConfig` 解码并写入边缘节点的 `/etc/kubernetes/` 目录
2. 使用 `ClusterExternalEndpoint` 作为 API Server 地址
3. 执行节点注册二进制（如 `tnctl`）完成注册

#### 增强配置

- 如需指定子网和网络类型，可通过 `--NetworkType` 参数传入 `VPC-CNI` 或 `GR`
- 脚本有效期通常为 24 小时，过期后需重新获取

---

### 步骤 4：在边缘节点上执行注册脚本（数据面，边缘节点侧）

#### 选择依据

- 此步骤在边缘节点本地执行，需确保边缘节点可通过公网访问步骤 3 中获取的 `ClusterExternalEndpoint`
- 建议为边缘节点指定合适的标签（labels）和污点（taints），以便后续调度

#### 最小配置

使用 `tnctl`（TKE 外部节点注册工具）执行注册。注册前需先初始化镜像凭据：

```bash
# 1. 初始化外部节点运行所需的镜像拉取凭据
./tnctl init --cluster-id CLUSTER_ID --region <Region>
```

输出示例：

```
[INFO] Initializing external node environment...
[INFO] Pulling required images...
[INFO] Initialization complete.
```

```bash
# 2. 将节点注册到 TKE 标准集群
./tnctl register \
    --cluster-id CLUSTER_ID \
    --node-name EDGE_NODE_NAME \
    --labels "node-role.kubernetes.io/edge=" \
    --region <Region>
```

输出示例：

```
[INFO] Registering node EDGE_NODE_NAME to cluster CLUSTER_ID...
[INFO] Fetching bootstrap token...
[INFO] Node EDGE_NODE_NAME registered successfully.
[INFO] Node status: Pending (will become Ready after kubelet syncs)
```

> **说明**：`tnctl` 是 TKE 提供的外部节点注册工具，通常与 `kubelet`、`kube-proxy` 等组件一同部署在外部节点上。首次使用时需从腾讯云官方渠道下载对应版本的二进制文件。

#### 增强配置

- **指定节点标签**：为节点添加业务语义标签，方便后续工作负载调度

```bash
./tnctl register \
    --cluster-id CLUSTER_ID \
    --node-name EDGE_NODE_NAME \
    --labels "node-role.kubernetes.io/edge=,env=production,location=beijing-edge" \
    --taints "node-role.kubernetes.io/edge=:NoSchedule" \
    --region <Region>
```

- **添加污点**：如不希望非边缘专用负载调度到此节点，可添加 taint
- **自定义 kubelet 参数**：通过环境变量或配置文件指定 kubelet 的 `--max-pods`、`--reserved-cpus` 等参数

> 注册完成后，kubelet 会自动启动并开始与控制面同步节点状态。等待约 30-60 秒后在控制面验证节点状态。

---

### 步骤 5：验证节点注册成功（控制面）

#### 选择依据

- 确认节点已成功注册并状态正常
- 检查节点的标签和污点是否符合预期

#### 最小配置

```bash
tccli tke DescribeClusterInstances \
    --region <Region> \
    --ClusterId CLUSTER_ID
```

输出示例：

```json
{
    "TotalCount": 1,
    "InstanceSet": [
        {
            "InstanceId": "NODE_ID",
            "InstanceRole": "WORKER",
            "InstanceState": "running",
            "FailedReason": "",
            "LanIP": "EDGE_NODE_IP",
            "NodePoolId": "",
            "InstanceAdvancedSettings": {
                "Labels": [
                    {
                        "Name": "node-role.kubernetes.io/edge",
                        "Value": ""
                    }
                ]
            },
            "CreatedTime": "2025-01-01T00:00:00Z"
        }
    ],
    "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

> 关键检查点：
> - `InstanceState` 为 `running`
> - `FailedReason` 为空
> - `Labels` 中包含预期的节点标签

**确认集群总体状态：**

```bash
tccli tke DescribeClusters --region <Region> --ClusterIds '["CLUSTER_ID"]'
```

输出示例：

```json
{
    "TotalCount": 1,
    "Clusters": [
        {
            "ClusterId": "CLUSTER_ID",
            "ClusterStatus": "Running",
            "ClusterNodeNum": 1
        }
    ]
}
```

> `ClusterNodeNum` 应包含新注册的节点数。

#### 增强配置

- **对比边缘集群节点与注册节点**：确认节点数量一致，无遗漏
- **检查节点标签完整性**：对比边缘集群中原有标签，确保注册节点携带了正确的调度标签

---

### 步骤 6：迁移工作负载（数据面，需 VPN/IOA）

#### 选择依据

- 工作负载迁移需要从边缘集群导出 YAML 描述，按需修改后进行应用
- 需通过 nodeSelector 或 nodeAffinity 将 Pod 调度到已注册的边缘节点
- 操作涉及多集群管理，建议使用独立的 kubeconfig 文件区分源集群和目标集群
- 若边缘集群与 TKE 控制面之间网络不通（例如边缘集群在本地区域无法访问公网），需通过 VPN 或 IOA 建立临时连接

#### 最小配置

**1. 从边缘集群导出 Deployment：**

```bash
kubectl --kubeconfig /path/to/edge-cluster-kubeconfig \
    get deployment DEPLOYMENT_NAME -n NAMESPACE -o yaml > /tmp/DEPLOYMENT_NAME.yaml
```

输出示例：

```
deployment.apps/DEPLOYMENT_NAME exported
```

**2. 修改 YAML，添加 nodeSelector 约束：**

编辑 `/tmp/DEPLOYMENT_NAME.yaml`，在 `spec.template.spec` 中添加：

```yaml
spec:
  template:
    spec:
      nodeSelector:
        node-role.kubernetes.io/edge: ""
```

**3. 应用到 TKE 标准集群：**

```bash
kubectl --kubeconfig /path/to/tke-cluster-kubeconfig \
    apply -f /tmp/DEPLOYMENT_NAME.yaml
```

输出示例：

```
deployment.apps/DEPLOYMENT_NAME created
```

**4. 验证 Pod 调度到注册节点：**

```bash
kubectl --kubeconfig /path/to/tke-cluster-kubeconfig \
    get pods -n NAMESPACE -o wide
```

输出示例：

```
NAME                                  READY   STATUS    RESTARTS   AGE   NODE
DEPLOYMENT_NAME-xxxxxxxxxx-xxxxx      1/1     Running   0          30s   EDGE_NODE_NAME
```

> 确认 Pod 的 `NODE` 列为注册节点的名称。

#### 增强配置

- **使用 nodeAffinity 实现更精确的调度**：

```yaml
spec:
  template:
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: node-role.kubernetes.io/edge
                operator: Exists
```

- **批量导出与迁移**：使用以下脚本批量导出所有命名空间的 Deployment：

```bash
for ns in $(kubectl --kubeconfig /path/to/edge-cluster-kubeconfig get ns -o jsonpath='{.items[*].metadata.name}'); do
  kubectl --kubeconfig /path/to/edge-cluster-kubeconfig \
    get deployment -n "$ns" -o yaml > /tmp/deployments-"$ns".yaml
done
```

- **StatefulSet 和 DaemonSet 迁移**：方法与 Deployment 类似，需分别导出后添加 nodeSelector 约束再应用
- **ConfigMap 和 Secret 迁移**：使用 `kubectl get configmap/secret -n NAMESPACE -o yaml` 导出，直接在目标集群 apply
- **灰度迁移**：先以 `replicas: 1` 部署灰度 Pod 验证功能正常，再逐步扩展至完整副本数

---

## 验证

### 控制面验证

**1. 确认注册节点在集群节点列表中：**

```bash
tccli tke DescribeClusterInstances \
    --region <Region> \
    --ClusterId CLUSTER_ID
```

输出示例：

```json
{
    "TotalCount": 1,
    "InstanceSet": [
        {
            "InstanceId": "NODE_ID",
            "InstanceRole": "WORKER",
            "InstanceState": "running",
            "FailedReason": "",
            "LanIP": "EDGE_NODE_IP"
        }
    ],
    "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

**2. 确认集群状态正常，节点数量正确：**

```bash
tccli tke DescribeClusters --region <Region> --ClusterIds '["CLUSTER_ID"]'
```

输出示例：

```json
{
    "TotalCount": 1,
    "Clusters": [
        {
            "ClusterId": "CLUSTER_ID",
            "ClusterStatus": "Running",
            "ClusterNodeNum": 1
        }
    ]
}
```

### 数据面验证

**1. 确认注册节点 Ready：**

```bash
kubectl --kubeconfig /path/to/tke-cluster-kubeconfig get nodes
```

输出示例：

```
NAME              STATUS   ROLES            AGE   VERSION
EDGE_NODE_NAME    Ready    edge             10m   v1.28.3-tke.1
```

> 关键检查点：`STATUS` 为 `Ready`，`ROLES` 包含 `edge` 标签。

**2. 确认工作负载已调度到注册节点并正常运行：**

```bash
kubectl --kubeconfig /path/to/tke-cluster-kubeconfig \
    get pods -n NAMESPACE -o wide
```

输出示例：

```
NAME                                  READY   STATUS    RESTARTS   AGE   NODE              NOMINATED NODE
DEPLOYMENT_NAME-xxxxxxxxxx-xxxxx      1/1     Running   0          2m    EDGE_NODE_NAME    <none>
```

> 确认 `STATUS` 为 `Running`，`NODE` 为注册节点名称。

**3. 查看节点详细信息确认标签和资源：**

```bash
kubectl --kubeconfig /path/to/tke-cluster-kubeconfig \
    describe node EDGE_NODE_NAME
```

输出示例：

```
Name:               EDGE_NODE_NAME
Roles:              edge
Labels:             node-role.kubernetes.io/edge=
                    kubernetes.io/hostname=EDGE_NODE_NAME
Annotations:        ...
Capacity:
  cpu:              4
  memory:           8192Mi
  pods:             110
Conditions:
  Type             Status  LastHeartbeatTime                 LastTransitionTime                Reason                       Message
  ----             ------  -----------------                 ------------------                ------                       -------
  Ready            True    Fri, 01 Jan 2025 00:01:00 +0800   Fri, 01 Jan 2025 00:00:00 +0800   KubeletReady                 kubelet is posting ready status
```

---

## 清理

### 清理步骤

> **警告**：清理操作将从 TKE 标准集群中移除注册节点。请确保以下前提条件均已满足：
> 1. 所有工作负载已从该节点调度到其他节点或已完成备份
> 2. 确认节点上已无运行中的 Pod
> 3. 边缘节点本身不会被销毁，仅移除其与 TKE 标准集群的注册关系

**1. 从 TKE 标准集群中删除注册节点：**

```bash
tccli tke DeleteClusterInstances \
    --region <Region> \
    --ClusterId CLUSTER_ID \
    --InstanceIds '["NODE_ID"]'
```

输出示例：

```json
{
    "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
    "SuccInstanceIds": ["NODE_ID"]
}
```

**2. 停止边缘节点上的 kubelet 和相关组件：**

```bash
systemctl stop kubelet
systemctl disable kubelet
```

**3. 清理节点上的 Kubernetes 相关文件和配置：**

```bash
kubeadm reset -f
rm -rf /etc/kubernetes/
rm -rf /var/lib/kubelet/
rm -rf /etc/cni/
```

### 清理验证

**1. 确认节点已从集群中移除：**

```bash
tccli tke DescribeClusterInstances \
    --region <Region> \
    --ClusterId CLUSTER_ID
```

输出示例：

```json
{
    "TotalCount": 0,
    "InstanceSet": [],
    "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

> `TotalCount` 为 0 表示节点已从集群中完全移除。

**2. 确认 kubelet 在边缘节点上已停止：**

```bash
systemctl status kubelet
```

输出示例：

```
● kubelet.service - kubelet: The Kubernetes Node Agent
   Loaded: loaded (/etc/systemd/system/kubelet.service; disabled)
   Active: inactive (dead)
```

---

## 排障

| 问题现象 | 可能原因 | 排查方法 | 解决方案 |
|---|---|---|---|
| 节点注册失败，提示 "connection refused" 或 "timeout" | 边缘节点无法访问 TKE 控制面的 `ClusterExternalEndpoint` | 在边缘节点执行 `curl -v https://CLUSTER_ID.ccs.tencent-cloud.com`；检查防火墙或安全组是否放行 443 端口 | 确保边缘节点可公网访问控制面地址；如需要，配置 HTTPS 代理或打通 VPN |
| 注册成功后节点长期处于 `NotReady` 状态 | kubelet 与 API Server 通信异常，或边缘节点资源不足 | 在边缘节点执行 `systemctl status kubelet` 查看日志；检查 `kubectl describe node EDGE_NODE_NAME` 的 Conditions | 排查 kubelet 日志定位错误原因（常见：CNI 插件未安装、容器运行时异常）；确保节点资源和磁盘空间充足 |
| 工作负载调度到注册节点后 Pod 一直 `Pending` | 节点资源不足或 nodeSelector 配置错误 | 执行 `kubectl describe pod POD_NAME -n NAMESPACE` 查看 Events；检查节点 `Allocatable` 资源是否满足 Pod 请求 | 调整 Pod 的资源 requests 或增加节点资源；确认 nodeSelector 标签与节点标签匹配 |
| Pod 正常运行但无法访问服务 | 边缘节点与集群内其他网络的 IP 连通性问题 | 在边缘节点执行 `ping POD_IP` 和 `curl SERVICE_CLUSTER_IP:PORT`；检查网络插件（如 Flannel、Calico）的 overlay 通信 | 配置适当的网络策略；如使用 VPC-CNI，确认边缘节点到 VPC 的路由可达 |
| `DescribeExternalNodeSupportConfig` 返回的 KubeConfig 为空 | 外部节点支持未成功开启 | 执行 `tccli tke DescribeClusters --ClusterIds '["CLUSTER_ID"]'` 确认集群状态；检查是否已执行 `EnableExternalNodeSupport` | 重新执行步骤 2 开启外部节点支持；如持续失败，提工单联系腾讯云技术支持 |
| 迁移后 Pod 的 PVC 无法挂载 | 边缘节点缺少对应的存储驱动或存储后端不可达 | 执行 `kubectl describe pvc PVC_NAME -n NAMESPACE` 查看 PVC 状态；检查边缘节点存储驱动（CSI）是否安装 | 重新部署对应存储的 CSI 插件；或将有状态应用的数据迁移至新的存储后端再绑定 PVC |
| `tnctl register` 提示 "bootstrap token expired" | 注册脚本或 token 已过期（默认有效期 24 小时） | 检查边缘节点系统时间是否与标准时间同步；确认获取注册脚本的时间 | 重新执行步骤 3 获取新的注册信息，并重新运行 `tnctl register` |

## 控制台替代

[TKE 控制台](https://console.cloud.tencent.com/tke2)

## 下一步

- [环境准备](../../环境准备.md)
