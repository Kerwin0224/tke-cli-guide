# 使用 CLB 为 Pod 分配公网地址映射（tccli）

> 对照官方：[使用 CLB 为 Pod 分配公网地址映射](https://cloud.tencent.com/document/product/457/111623) · page_id `111623`

## 概述

游戏服务器场景中，每个游戏服进程需要直接暴露在公网上，以便玩家直连。传统的 Service 负载均衡会将流量分发到后端多个 Pod，无法满足“每 Pod 一个独立公网地址/端口”的需求。腾讯云 TKE 支持通过 **CLB 直连 Pod 模式** 实现为 StatefulSet 中每个 Pod 分配独立的公网 IP 或端口映射，适用于游戏对战服、房间服等场景。

> **注意**：以下操作步骤中，控制面（tccli）命令可在任意有网络访问的环境中执行；数据面（kubectl）命令须在 VPN/IOA 连通集群内网后方可执行，否则会返回网络不可达。

### 适用场景

- MMO/FPS 等游戏的战斗服、房间服需要为每个游戏进程分配独立公网入口
- 使用 StatefulSet 管理游戏服务器，每个 Pod 绑定唯一的外部端口或 IP
- 需要 CLB 健康检查感知游戏服务器的存活状态，自动剔除异常节点
- 已在同一 VPC 下购买 CLB 实例，需要复用

## 功能要点

| 功能项 | 说明 |
|--------|------|
| CLB 直连 Pod 模式 | 跳过 NodePort，CLB 直接将流量转发至 Pod IP（要求 VPC-CNI 网络模式） |
| 每 Pod 独立端口映射（port-mapping） | 通过 annotation 为每个 Pod 分配独立外侧端口 |
| 每 Pod 独立 CLB（多 CLB 方案） | 每个 Pod 绑定独立 CLB 实例，获得独立公网 IP |
| CLB 健康检查 | 支持 TCP/UDP/HTTP 健康检查，适用于游戏协议 |
| 复用已有 CLB | 通过 `tke-existed-lbid` annotation 绑定已购 CLB 实例 |

## 前置条件

- 已创建 TKE 集群（CLUSTER_ID），且网络模式为 **VPC-CNI**（CLB 直连 Pod 的前置要求）
- 集群所在 VPC（VPC_ID）下有可用子网
- 已在目标地域（REGION）购买 CLB 实例（INSTANCE_ID），或将在创建 Service 时自动创建
- 本地已安装并配置 tccli 及 kubectl
- 已安装 jq（用于 JSON 解析）
- 数据面操作需通过 VPN/IOA 连通集群内网

## 控制台与 CLI 参数映射

| 控制台参数 | CLI 参数 | 必选 | 说明 | 幂等性 |
|-----------|----------|------|------|--------|
| 集群 ID | `--ClusterIds` / `--ClusterId` | 是 | TKE 集群 ID | 是（指定已存在资源） |
| 地域 | `--region` | 是 | 腾讯云地域，如 `ap-guangzhou` | 是（仅查询） |
| VPC ID | `vpc-id` filter | 是 | 私有网络 ID | 是（仅查询） |
| CLB 实例 ID | `service.kubernetes.io/tke-existed-lbid` annotation | 否 | 复用已有 CLB，不指定则自动创建 | 是（绑定已存在 CLB） |
| 后端 Pod 标签 | `service.kubernetes.io/qcloud-loadbalancer-backends-label` annotation | 否 | 指定 CLB 后端绑定的 Pod 标签 | 否（重复应用覆盖） |
| 端口映射 | `service.kubernetes.io/qcloud-loadbalancer-clb-port-mapping` annotation | 否 | 每 Pod 独立外侧端口映射 | 否（重复应用覆盖） |
| 健康检查 | `service.kubernetes.io/qcloud-loadbalancer-health-check-*` annotations | 否 | CLB 健康检查配置参数 | 否（重复应用覆盖） |
| 命名空间 | `-n` / `--namespace` | 是 | Kubernetes 命名空间 | 是（指定已存在资源） |

## 操作步骤

### 第一步：确认集群支持 CLB 直连 Pod

#### 选择依据

CLB 直连 Pod 模式（Direct Pod）要求集群网络为 VPC-CNI 模式（非 GlobalRouter），因为 CLB 需要直接将流量转发至 Pod 的 VPC 弹性网卡 IP。在使用该功能前，必须确认集群网络模式满足要求。

#### 最小配置

**1.1 查询集群基本信息，确认网络模式**

```bash
tccli tke DescribeClusters --region <Region> --ClusterIds '["CLUSTER_ID"]'
```

预期输出（关键字段）：

```json
{
  "Clusters": [
    {
      "ClusterId": "CLUSTER_ID",
      "ClusterName": "game-cluster",
      "ClusterNetworkSettings": {
        "ClusterCIDR": "10.0.0.0/16",
        "ServiceCIDR": "10.1.0.0/16",
        "VpcId": "VPC_ID",
        "IsVpcCni": true
      }
    }
  ],
  "TotalCount": 1
}
```

> **关键判断**：`IsVpcCni` 须为 `true`。若为 `false`，集群使用的是 GlobalRouter 网络模式，不支持 CLB 直连 Pod，需先通过控制台或 API 为集群开启 VPC-CNI 支持。

**1.2 查询集群运行状态**

```bash
tccli tke DescribeClusterStatus --region <Region> --ClusterIds '["CLUSTER_ID"]'
```

预期输出（关键字段）：

```json
{
  "ClusterStatusSet": [
    {
      "ClusterId": "CLUSTER_ID",
      "ClusterState": "Running",
      "ClusterBMonitor": true
    }
  ]
}
```

> 确保 `ClusterState` 为 `Running`。

**1.3 使用 --filters 过滤获取指定集群信息（精简查询）**

```bash
tccli tke DescribeClusters --region <Region> --ClusterIds '["CLUSTER_ID"]' | jq '.Clusters[] | {ClusterId, ClusterNetworkSettings: {VpcId, IsVpcCni}}'
```

预期输出：

```json
{
  "ClusterId": "CLUSTER_ID",
  "ClusterNetworkSettings": {
    "VpcId": "VPC_ID",
    "IsVpcCni": true
  }
}
```

### 第二步：获取 VPC 子网信息

#### 选择依据

CLB 实例须创建在集群所在 VPC 的某个子网中。查询子网列表以便后续创建 Service 时指定正确的子网（或确认自动创建 CLB 时使用的子网）。

#### 最小配置

**2.1 查询 VPC 下所有子网**

```bash
tccli vpc DescribeSubnets --region <Region> --Filters '[{"Name":"vpc-id","Values":["VPC_ID"]}]'
```

预期输出（关键字段）：

```json
{
  "SubnetSet": [
    {
      "SubnetId": "subnet-xxxxxxxx",
      "SubnetName": "game-subnet-1",
      "CidrBlock": "10.0.1.0/24",
      "Zone": "ap-guangzhou-3",
      "AvailableIpAddressCount": 200
    }
  ],
  "TotalCount": 1
}
```

**2.2 精简查询（仅输出子网 ID、CIDR 和可用 IP 数）**

```bash
tccli vpc DescribeSubnets --region <Region> --Filters '[{"Name":"vpc-id","Values":["VPC_ID"]}]' | jq '.SubnetSet[] | {SubnetId, CidrBlock, AvailableIpAddressCount, Zone}'
```

预期输出：

```json
{
  "SubnetId": "subnet-xxxxxxxx",
  "CidrBlock": "10.0.1.0/24",
  "AvailableIpAddressCount": 200,
  "Zone": "ap-guangzhou-3"
}
```

> 确保子网中有足够可用 IP。每个 Pod 在 VPC-CNI 模式下会消耗一个子网 IP。

### 第三步：最小配置 — 创建 StatefulSet + CLB Service

#### 选择依据

创建游戏服 StatefulSet 和 CLB 直连 Service 的最简组合。此处展示核心配置：StatefulSet 负责管理 Pod，Service 负责把 CLB 后端直接绑定到这些 Pod。

#### 最小配置

**3.1 创建 StatefulSet（游戏服）**

创建文件 `game-server-statefulset.yaml`：

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: game-server
  namespace: NAMESPACE
spec:
  serviceName: game-server-svc
  replicas: 3
  podManagementPolicy: OrderedReady
  selector:
    matchLabels:
      app: game-server
  template:
    metadata:
      labels:
        app: game-server
    spec:
      containers:
      - name: game-server
        image: ccr.ccs.tencentyun.com/NAMESPACE/game-server:latest
        ports:
        - containerPort: 27015
          protocol: TCP
          name: game
        resources:
          requests:
            cpu: "1"
            memory: "2Gi"
          limits:
            cpu: "2"
            memory: "4Gi"
```

应用 StatefulSet：

```bash
kubectl apply -f game-server-statefulset.yaml
```

预期输出：

```
statefulset.apps/game-server created
```

**3.2 验证 Pod 已就绪**

```bash
kubectl get pods -n NAMESPACE -l app=game-server -o wide
```

预期输出：

```
NAME             READY   STATUS    RESTARTS   AGE   IP           NODE
game-server-0    1/1     Running   0          30s   VPC_POD_IP   NODE_ID
game-server-1    1/1     Running   0          25s   VPC_POD_IP   NODE_ID
game-server-2    1/1     Running   0          20s   VPC_POD_IP   NODE_ID
```

> 在 VPC-CNI 模式下，Pod IP 是 VPC 子网内的 IP，可直接路由。

**3.3 创建 CLB 直连 Service（自动创建 CLB）**

创建文件 `game-server-clb-svc.yaml`：

```yaml
apiVersion: v1
kind: Service
metadata:
  name: game-server-svc
  namespace: NAMESPACE
  annotations:
    service.kubernetes.io/qcloud-loadbalancer-backends-label: "app=game-server"
    service.kubernetes.io/qcloud-loadbalancer-direct-access: "true"
  labels:
    app: game-server
spec:
  type: LoadBalancer
  selector:
    app: game-server
  ports:
  - port: 27015
    targetPort: 27015
    protocol: TCP
    name: game
```

应用 Service：

```bash
kubectl apply -f game-server-clb-svc.yaml
```

预期输出：

```
service/game-server-svc created
```

> **annotation 说明**：
> - `qcloud-loadbalancer-backends-label`：告诉 CLB 控制器直接绑定具有该标签的 Pod 作为后端（绕过 NodePort）
> - `qcloud-loadbalancer-direct-access`：启用直连模式

**3.4 查看 Service 状态，获取 CLB 公网 IP**

```bash
kubectl get svc -n NAMESPACE game-server-svc -o wide
```

预期输出：

```
NAME              TYPE           CLUSTER-IP     EXTERNAL-IP     PORT(S)             AGE
game-server-svc   LoadBalancer   10.1.x.x       EXTERNAL_IP     27015:30000/TCP     1m
```

### 第四步：增强配置 — 每 Pod 独立端口映射

#### 选择依据

当多个游戏服运行在不同 Pod 但都监听相同容器端口（如 27015）时，需要为每个 Pod 分配不同的公网端口，使得玩家可以通过 `EXTERNAL_IP:PORT` 分别直连不同的游戏服。这通过 `clb-port-mapping` annotation 实现。

#### 增强配置

**4.1 更新 Service 添加端口映射 annotation**

创建文件 `game-server-clb-svc-portmap.yaml`：

```yaml
apiVersion: v1
kind: Service
metadata:
  name: game-server-svc
  namespace: NAMESPACE
  annotations:
    service.kubernetes.io/qcloud-loadbalancer-backends-label: "app=game-server"
    service.kubernetes.io/qcloud-loadbalancer-direct-access: "true"
    service.kubernetes.io/qcloud-loadbalancer-clb-port-mapping: '[
      {"podLabel":"statefulset.kubernetes.io/pod-name=game-server-0","externalPort":27015,"internalPort":27015,"protocol":"TCP"},
      {"podLabel":"statefulset.kubernetes.io/pod-name=game-server-1","externalPort":27016,"internalPort":27015,"protocol":"TCP"},
      {"podLabel":"statefulset.kubernetes.io/pod-name=game-server-2","externalPort":27017,"internalPort":27015,"protocol":"TCP"}
    ]'
  labels:
    app: game-server
spec:
  type: LoadBalancer
  selector:
    app: game-server
  ports:
  - port: 27015
    targetPort: 27015
    protocol: TCP
    name: game
```

应用更新：

```bash
kubectl apply -f game-server-clb-svc-portmap.yaml
```

预期输出：

```
service/game-server-svc configured
```

> **关键点**：
> - `podLabel`：使用 StatefulSet Pod 的固定标识（`statefulset.kubernetes.io/pod-name=game-server-N`），而非 Pod 自身标签，确保每个 Pod 获取唯一且稳定的端口映射
> - `externalPort`：各 Pod 不同（27015、27016、27017），分别映射到同一 `internalPort`（27015）
> - StatefulSet Pod 名称是稳定的，重启后端口映射不会漂移

**4.2 验证端口映射生效**

```bash
kubectl get svc -n NAMESPACE game-server-svc -o yaml | grep -A 20 annotations
```

**4.3 对于大规模游戏服（如 10 个 Pod），使用 Parallel 模式加速部署**

更新 StatefulSet：

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: game-server
  namespace: NAMESPACE
spec:
  serviceName: game-server-svc
  replicas: 10
  podManagementPolicy: Parallel
  selector:
    matchLabels:
      app: game-server
  template:
    metadata:
      labels:
        app: game-server
    spec:
      containers:
      - name: game-server
        image: ccr.ccs.tencentyun.com/NAMESPACE/game-server:latest
        ports:
        - containerPort: 27015
          protocol: TCP
          name: game
        resources:
          requests:
            cpu: "1"
            memory: "2Gi"
          limits:
            cpu: "2"
            memory: "4Gi"
```

```bash
kubectl apply -f game-server-statefulset.yaml
```

```text
# command executed successfully
```

同时更新 Service 的 port-mapping annotation，将 `podLabel` 扩展至 10 个 Pod（game-server-0 到 game-server-9，externalPort 27015-27024）：

```bash
kubectl patch svc -n NAMESPACE game-server-svc --type merge -p '
{
  "metadata": {
    "annotations": {
      "service.kubernetes.io/qcloud-loadbalancer-clb-port-mapping": "[{\"podLabel\":\"statefulset.kubernetes.io/pod-name=game-server-0\",\"externalPort\":27015,\"internalPort\":27015,\"protocol\":\"TCP\"},{\"podLabel\":\"statefulset.kubernetes.io/pod-name=game-server-1\",\"externalPort\":27016,\"internalPort\":27015,\"protocol\":\"TCP\"},{\"podLabel\":\"statefulset.kubernetes.io/pod-name=game-server-2\",\"externalPort\":27017,\"internalPort\":27015,\"protocol\":\"TCP\"},{\"podLabel\":\"statefulset.kubernetes.io/pod-name=game-server-3\",\"externalPort\":27018,\"internalPort\":27015,\"protocol\":\"TCP\"},{\"podLabel\":\"statefulset.kubernetes.io/pod-name=game-server-4\",\"externalPort\":27019,\"internalPort\":27015,\"protocol\":\"TCP\"},{\"podLabel\":\"statefulset.kubernetes.io/pod-name=game-server-5\",\"externalPort\":27020,\"internalPort\":27015,\"protocol\":\"TCP\"},{\"podLabel\":\"statefulset.kubernetes.io/pod-name=game-server-6\",\"externalPort\":27021,\"internalPort\":27015,\"protocol\":\"TCP\"},{\"podLabel\":\"statefulset.kubernetes.io/pod-name=game-server-7\",\"externalPort\":27022,\"internalPort\":27015,\"protocol\":\"TCP\"},{\"podLabel\":\"statefulset.kubernetes.io/pod-name=game-server-8\",\"externalPort\":27023,\"internalPort\":27015,\"protocol\":\"TCP\"},{\"podLabel\":\"statefulset.kubernetes.io/pod-name=game-server-9\",\"externalPort\":27024,\"internalPort\":27015,\"protocol\":\"TCP\"}]"
    }
  }
}'
```

预期输出：

```
service/game-server-svc patched
```

### 第五步：增强配置 — 为每 Pod 绑定独立 CLB 公网 IP（多 CLB 方案）

#### 选择依据

在某些游戏场景（如 Steam Server Browser）中，每个游戏服需要一个独立的公网 IP，而非仅独立端口。多 CLB 方案通过为每个 StatefulSet Pod 创建独立 Service（各自绑定独立 CLB），使每个 Pod 获得独立公网 IP。

#### 增强配置

**5.1 创建独立的 Service Per Pod**

为每个 Pod 创建独立 Service，每个 Service 通过 `tke-existed-lbid` 绑定已有 CLB，或自动创建新 CLB，并通过 `backends-label` 精确定位到特定 Pod。

示例：为 `game-server-0` 创建独立 Service `game-server-0-svc`（自动创建 CLB）：

```yaml
apiVersion: v1
kind: Service
metadata:
  name: game-server-0-svc
  namespace: NAMESPACE
  annotations:
    service.kubernetes.io/qcloud-loadbalancer-backends-label: "statefulset.kubernetes.io/pod-name=game-server-0"
    service.kubernetes.io/qcloud-loadbalancer-direct-access: "true"
spec:
  type: LoadBalancer
  selector:
    app: game-server
  ports:
  - port: 27015
    targetPort: 27015
    protocol: TCP
    name: game
```

```bash
kubectl apply -f game-server-0-svc.yaml
```

预期输出：

```
service/game-server-0-svc created
```

> 此方案下，每个 Pod 有独立 Service，独立 CLB，独立公网 IP。但 CLB 按量计费，10 个 Pod 将产生 10 个 CLB 实例费用。

**5.2 复用已有 CLB 实例（指定 tke-existed-lbid）**

如果已预先购买 CLB 实例，可在 Service 中指定：

```yaml
apiVersion: v1
kind: Service
metadata:
  name: game-server-0-svc
  namespace: NAMESPACE
  annotations:
    service.kubernetes.io/qcloud-loadbalancer-backends-label: "statefulset.kubernetes.io/pod-name=game-server-0"
    service.kubernetes.io/tke-existed-lbid: "lb-xxxxxxxx"
spec:
  type: LoadBalancer
  selector:
    app: game-server
  ports:
  - port: 27015
    targetPort: 27015
    protocol: TCP
    name: game
```

**5.3 查看各 Service 获取的 CLB 公网 IP**

```bash
kubectl get svc -n NAMESPACE -l app=game-server
```

预期输出（多 Service 场景）：

```
NAME                 TYPE           CLUSTER-IP     EXTERNAL-IP      PORT(S)             AGE
game-server-0-svc    LoadBalancer   10.1.x.x       1.2.3.4          27015:30001/TCP     1m
game-server-1-svc    LoadBalancer   10.1.x.y       1.2.3.5          27015:30002/TCP     1m
game-server-2-svc    LoadBalancer   10.1.x.z       1.2.3.6          27015:30003/TCP     1m
```

### 第六步：游戏服务器健康检查配置

#### 选择依据

游戏服务器需要 CLB 层面的健康检查来探测游戏进程是否存活（TCP 端口通断 / 游戏协议探测），以便 CLB 自动将异常 Pod 从后端池移除，避免玩家连接到不可用的游戏服。

#### 增强配置

**6.1 TCP 健康检查（游戏服端口直连探测）**

在 Service 中添加健康检查 annotations：

```yaml
apiVersion: v1
kind: Service
metadata:
  name: game-server-svc
  namespace: NAMESPACE
  annotations:
    service.kubernetes.io/qcloud-loadbalancer-backends-label: "app=game-server"
    service.kubernetes.io/qcloud-loadbalancer-direct-access: "true"
    service.kubernetes.io/qcloud-loadbalancer-health-check-switch: "on"
    service.kubernetes.io/qcloud-loadbalancer-health-check-type: "TCP"
    service.kubernetes.io/qcloud-loadbalancer-health-check-interval: "5"
    service.kubernetes.io/qcloud-loadbalancer-health-check-timeout: "2"
    service.kubernetes.io/qcloud-loadbalancer-health-check-healthy-num: "3"
    service.kubernetes.io/qcloud-loadbalancer-health-check-unhealthy-num: "3"
  labels:
    app: game-server
spec:
  type: LoadBalancer
  selector:
    app: game-server
  ports:
  - port: 27015
    targetPort: 27015
    protocol: TCP
    name: game
```

应用 Service：

```bash
kubectl apply -f game-server-clb-svc.yaml
```

预期输出：

```
service/game-server-svc configured
```

**6.2 自定义游戏协议健康检查（HTTP/HTTPS）**

对于支持 HTTP 健康检查接口的游戏服务器（如 RESTful 健康端点），可配置 HTTP 健康检查：

```yaml
metadata:
  annotations:
    service.kubernetes.io/qcloud-loadbalancer-health-check-type: "HTTP"
    service.kubernetes.io/qcloud-loadbalancer-health-check-http-method: "GET"
    service.kubernetes.io/qcloud-loadbalancer-health-check-http-path: "/health"
    service.kubernetes.io/qcloud-loadbalancer-health-check-http-code: "200"
```

**6.3 健康检查参数说明**

| 参数 | 说明 | 默认值 | 推荐值（游戏场景） |
|------|------|--------|-------------------|
| `health-check-switch` | 健康检查开关 | on | on |
| `health-check-type` | 探测类型（TCP/HTTP/HTTPS） | TCP | TCP |
| `health-check-interval` | 探测间隔（秒） | 5 | 5 |
| `health-check-timeout` | 探测超时（秒） | 2 | 2 |
| `health-check-healthy-num` | 健康阈值（连续成功次数） | 3 | 2（游戏服尽快恢复） |
| `health-check-unhealthy-num` | 不健康阈值（连续失败次数） | 3 | 2（游戏服尽快剔除） |

## 验证

### 控制面

**验证 CLB 实例信息（当使用 tccli clb API 时）**

```bash
tccli clb DescribeLoadBalancers --region <Region> --LoadBalancerIds '["lb-xxxxxxxx"]'
```

预期输出（关键字段）：

```json
{
  "LoadBalancerSet": [
    {
      "LoadBalancerId": "lb-xxxxxxxx",
      "LoadBalancerName": "game-server-svc",
      "LoadBalancerType": "OPEN",
      "Status": 1,
      "VpcId": "VPC_ID",
      "LoadBalancerVips": ["EXTERNAL_IP"]
    }
  ],
  "TotalCount": 1
}
```

> `Status: 1` 表示 CLB 处于正常状态。

**对比 TKE 集群描述确认 CLB IP 归属**

```bash
tccli tke DescribeClusters --region <Region> --ClusterIds '["CLUSTER_ID"]' | jq '.Clusters[] | {ClusterId, VpcId: .ClusterNetworkSettings.VpcId}'
```

```json
{
  "TotalCount": 0,
  "Clusters": [],
  "ClusterId": "<ClusterId>",
  "ClusterName": "<ClusterName>",
  "ClusterDescription": "<ClusterDescription>",
  "ClusterVersion": "<ClusterVersion>",
  "ClusterOs": "<ClusterOs>",
  "ClusterType": "<ClusterType>"
}
```

### 数据面

**验证 Service 和 EXTERNAL-IP**

```bash
kubectl get svc -n NAMESPACE game-server-svc -o wide
```

预期输出：

```
NAME              TYPE           CLUSTER-IP     EXTERNAL-IP     PORT(S)             AGE
game-server-svc   LoadBalancer   10.1.x.x       EXTERNAL_IP     27015:30000/TCP     5m
```

> `EXTERNAL-IP` 列显示的是 CLB 公网 IP（非 pending），表示 CLB 已绑定成功。

**验证 Pod 状态及 IP**

```bash
kubectl get pods -n NAMESPACE -l app=game-server -o wide
```

预期输出：

```
NAME             READY   STATUS    RESTARTS   AGE   IP           NODE
game-server-0    1/1     Running   0          5m    VPC_POD_IP   NODE_ID
game-server-1    1/1     Running   0          5m    VPC_POD_IP   NODE_ID
game-server-2    1/1     Running   0          5m    VPC_POD_IP   NODE_ID
```

**测试 CLB 端口连通性（从外网或跳板机）**

```bash
telnet EXTERNAL_IP 27015
```

或使用 nc：

```bash
nc -zv EXTERNAL_IP 27015
```

预期输出（成功）：

```
Connection to EXTERNAL_IP port 27015 [tcp/*] succeeded!
```

**验证 CLB 端口映射每 Pod 独立端口（若配置了 port-mapping）**

```bash
nc -zv EXTERNAL_IP 27015 && echo "game-server-0 OK"
nc -zv EXTERNAL_IP 27016 && echo "game-server-1 OK"
nc -zv EXTERNAL_IP 27017 && echo "game-server-2 OK"
```

## 清理

> **计费提醒**：CLB 实例采用按量计费（后付费）或包年包月（预付费），未主动删除的 CLB 将持续产生费用：
> - **自动创建的 CLB**：删除关联 Service 时，TKE 控制器默认会自动删除由 Service 动态创建的 CLB 实例
> - **复用已有 CLB**（通过 `tke-existed-lbid` 绑定）：删除 Service 不会删除 CLB 实例本身，需通过控制台或 tccli 手动清理
> - CLB 计费包含 **实例费**（按量 0.02 元/小时起）和 **公网带宽费**（按流量或按带宽），请及时清理不再使用的资源

**清理 Kubernetes 资源**

```bash
kubectl delete svc -n NAMESPACE game-server-svc
```

预期输出：

```
service "game-server-svc" deleted
```

```bash
kubectl delete statefulset -n NAMESPACE game-server
```

预期输出：

```
statefulset.apps "game-server" deleted
```

**清理残留 Pod**

```bash
kubectl delete pods -n NAMESPACE -l app=game-server --force --grace-period=0
```

**验证 CLB 是否已自动删除（仅适用于自动创建的 CLB）**

```bash
tccli clb DescribeLoadBalancers --region <Region> --LoadBalancerIds '["lb-xxxxxxxx"]' 2>&1
```

> 自动创建的 CLB 在 Service 删除后约 30-60 秒会被清理。若为复用已有 CLB（通过 `tke-existed-lbid` 绑定），需手动删除。

**手动删除 CLB 实例（如果使用 tke-existed-lbid 绑定已有 CLB 或自动清理失败）**

```bash
tccli clb DeleteLoadBalancer --region <Region> --LoadBalancerIds '["lb-xxxxxxxx"]'
```

预期输出：

```json
{
  "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

## 排障

| 现象 | 可能原因 | 排查方法 | 解决方案 |
|------|---------|---------|---------|
| CLB 未创建，Service EXTERNAL-IP 持续 `pending` | 1. VPC-CNI 未启用<br>2. 子网 IP 不足<br>3. 账户 CLB 配额不足 | 1. `kubectl describe svc -n NAMESPACE game-server-svc` 查看 Events<br>2. 检查 VPC-CNI 状态<br>3. 检查 CLB 配额 | 1. 确保 `IsVpcCni: true`<br>2. 更换有足够 IP 的子网<br>3. 提交工单提升 CLB 配额 |
| CLB 后端 Pod 显示不健康 | 1. 容器内游戏进程未启动<br>2. 容器端口 27015 未监听<br>3. 安全组拦截健康检查流量 | 1. `kubectl logs -n NAMESPACE game-server-0` 查看游戏服日志<br>2. `kubectl exec -n NAMESPACE game-server-0 -- netstat -tlnp` 确认端口监听<br>3. 确认 CLB 安全组放通健康检查源 IP | 1. 检查游戏服配置和启动脚本<br>2. 确保容器端口与 targetPort 一致<br>3. 安全组规则放通 `100.64.0.0/10`（CLB 健康检查源 IP 段） |
| CLB 端口冲突，多个待创建 Service 指定了同一 externalPort | port-mapping annotation 中多个 Pod 使用了相同的 externalPort | 检查 annotation JSON 中的 externalPort 是否重复 | 为每个 Pod 分配唯一 externalPort，确保端口不重叠；可通过脚本批量生成端口映射 |
| StatefulSet Pods 未获取唯一端口映射 | 1. port-mapping 中 podLabel 与 Pod 实际标签不匹配<br>2. CLB 控制器版本过旧不支持 port-mapping | 1. `kubectl get pod -n NAMESPACE game-server-0 --show-labels` 查看实际标签<br>2. 检查 TKE 集群版本（建议 ≥ 1.24） | 1. 使用 `statefulset.kubernetes.io/pod-name` 标签匹配（StatefulSet Pod 专属）<br>2. 升级 TKE 集群以匹配 CLB 控制器版本 |
| CLB TCP 健康检查对游戏协议探测失败 | 游戏服端口在 TCP 层面可达但游戏协议握手未完成，CLB 误判健康 | CLB 健康检查仅验证 TCP 端口通断，不感知应用层协议 | 在游戏服务器上实现 HTTP 健康检查端点（如 `/health`），切换 `health-check-type` 为 `HTTP`，或部署旁路健康检查脚本 |
| VPC-CNI 模式未启用导致直连失败 | 集群为 GlobalRouter 网络模式 | `tccli tke DescribeClusters` 确认 `IsVpcCni` | 1. 新建 VPC-CNI 集群迁移<br>2. 或通过 TKE 控制台在已有集群上开启 VPC-CNI 支持（需满足条件） |
| 跨可用区 CLB 导致的延迟或不均衡 | CLB 实例所在可用区与 Pod 所在节点可用区不一致，跨 AZ 流量产生延迟 | `tccli clb DescribeLoadBalancers` 查看 CLB MasterZone，`kubectl get nodes -o wide` 查看节点 ZONE | 1. 创建 CLB 时指定与 Pod 调度节点相同的可用区<br>2. 使用 `service.kubernetes.io/qcloud-loadbalancer-master-zone-id` annotation 指定主可用区 |

## 参数参考

### Service Annotations（CLB 相关）

| Annotation | 作用 | 示例值 |
|-----------|------|--------|
| `service.kubernetes.io/qcloud-loadbalancer-backends-label` | 指定 CLB 后端绑定的 Pod 标签 | `app=game-server` |
| `service.kubernetes.io/qcloud-loadbalancer-direct-access` | 启用 CLB 直连 Pod 模式 | `true` |
| `service.kubernetes.io/tke-existed-lbid` | 绑定已有 CLB 实例 | `lb-xxxxxxxx` |
| `service.kubernetes.io/qcloud-loadbalancer-clb-port-mapping` | 每 Pod 独立端口映射（JSON 数组） | 见第四步示例 |
| `service.kubernetes.io/qcloud-loadbalancer-health-check-switch` | 健康检查开关 | `on` / `off` |
| `service.kubernetes.io/qcloud-loadbalancer-health-check-type` | 健康检查类型 | `TCP` / `HTTP` / `HTTPS` |
| `service.kubernetes.io/qcloud-loadbalancer-health-check-interval` | 健康检查间隔（秒） | `5` |
| `service.kubernetes.io/qcloud-loadbalancer-health-check-timeout` | 健康检查超时（秒） | `2` |
| `service.kubernetes.io/qcloud-loadbalancer-health-check-healthy-num` | 健康阈值（次） | `3` |
| `service.kubernetes.io/qcloud-loadbalancer-health-check-unhealthy-num` | 不健康阈值（次） | `3` |
| `service.kubernetes.io/qcloud-loadbalancer-master-zone-id` | 指定 CLB 主可用区 | `ap-guangzhou-3` |
| `service.kubernetes.io/qcloud-loadbalancer-internet-charge-type` | 公网计费类型 | `TRAFFIC_POSTPAID_BY_HOUR` / `BANDWIDTH_POSTPAID_BY_HOUR` |
| `service.kubernetes.io/qcloud-loadbalancer-internet-max-bandwidth-out` | 最大出带宽（Mbps） | `100` |

### StatefulSet 关键字段

| 字段 | 作用 | 推荐值（游戏场景） |
|------|------|-------------------|
| `spec.podManagementPolicy` | Pod 管理策略 | `Parallel`（快速并行部署） |
| `spec.serviceName` | 关联 Service 名称 | 须与 Service `metadata.name` 一致 |
| `spec.template.metadata.labels` | Pod 标签 | `app=game-server`，与 Service `backends-label` 一致 |

## 常用命令速查

```bash
# 查询集群 VPC-CNI 状态
tccli tke DescribeClusters --region <Region> --ClusterIds '["CLUSTER_ID"]' | jq '.Clusters[] | {ClusterId, IsVpcCni: .ClusterNetworkSettings.IsVpcCni}'

# 查询 VPC 子网
tccli vpc DescribeSubnets --region <Region> --Filters '[{"Name":"vpc-id","Values":["VPC_ID"]}]' | jq '.SubnetSet[] | {SubnetId, CidrBlock, AvailableIpAddressCount}'

# 创建 StatefulSet
kubectl apply -f game-server-statefulset.yaml

# 创建 CLB Service
kubectl apply -f game-server-clb-svc.yaml

# 查看 Service 状态
kubectl get svc -n NAMESPACE -l app=game-server -o wide

# 查看 Pod 详情
kubectl get pods -n NAMESPACE -l app=

```

```text
# command executed successfully
```json
{
  "TotalCount": 0,
  "Clusters": [],
  "ClusterId": "<ClusterId>",
  "ClusterName": "<ClusterName>",
  "ClusterDescription": "<ClusterDescription>",
  "ClusterVersion": "<ClusterVersion>",
  "ClusterOs": "<ClusterOs>",
  "ClusterType": "<ClusterType>"
}
```game-server -o wide

# 测试 CLB 连通性
nc -zv EXTERNAL_IP 27015

# 查看 CLB 实例
tccli clb DescribeLoadBalancers --region <Region> --LoadBalancerIds '["lb-xxxxxxxx"]'

# 清理资源
kubectl delete svc -n NAMESPACE game-server-svc
kubectl delete statefulset -n NAMESPACE game-server
```

## 控制台替代

[TKE 控制台](https://console.cloud.tencent.com/tke2)

## 下一步

- [环境准备](../../环境准备.md)
