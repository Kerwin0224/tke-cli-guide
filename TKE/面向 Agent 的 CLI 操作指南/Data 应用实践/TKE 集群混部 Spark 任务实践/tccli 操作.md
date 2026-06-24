# TKE 集群混部运行 Spark 任务（tccli）

> 对照官方：[TKE 集群混部 Spark 任务实践](https://cloud.tencent.com/document/product/457/125431) · page_id `125431`

## 概述

在已部署混部能力的 TKE 原生节点集群上，以离线任务方式运行 Spark 计算任务。Spark Driver 和 Executor Pod 使用 `gocrane.io/cpu` 和 `gocrane.io/memory` 扩展资源，利用在线业务的闲置资源执行数据计算。

本页涵盖两种提交方式：`spark-submit`（原生方式）和 Spark Operator（K8s 声明式 CRD），以及两种调度器：Kubernetes 默认调度器和 Volcano 批量调度器。

> **kubectl 数据面不可达**：CAM 拒绝公网端点（strategyId:240463971），内网需 VPN/IOA。以下 kubectl、helm 和 spark-submit 命令为参考格式，需在内网/VPN 环境下执行。

## 前置条件

- [环境准备](../../环境准备.md)
- 已完成 [TKE 集群混部原理与部署介绍](../TKE%20集群混部原理与部署介绍/tccli%20操作.md) -- page_id `125430`（集群有原生节点、Craned + QoSAgent 已安装、离线避让规则已部署）

### 环境检查

```bash
# 1. 检查 tccli 版本
tccli --version
# expected: tccli version >= 1.0.0

# 2. 检查当前凭据和地域
tccli configure list
# expected: secretId、secretKey、region 均已配置

# 3. 检查 CAM 权限 — 需要以下 Action 名
#    tke:DescribeClusters, tke:DescribeClusterKubeconfig
#    tke:DescribeClusterInstances
# 验证
tccli tke DescribeClusters --region <Region>
# expected: exit 0

# 4. 检查 kubectl（须 VPN/IOA）
kubectl version --client
# expected: Client Version v1.32+

# 5. 检查 helm（须 VPN/IOA，Spark Operator 和 Volcano 需要）
helm version
# expected: Version v3.x.x

# 6. 检查 Java（spark-submit 需要）
java -version
# expected: openjdk version "11.x.x" 或更高
```

```json
{
  "TotalCount": "<TotalCount>",
  "Clusters": "<Clusters>",
  "ClusterId": "<ClusterId>",
  "ClusterName": "<ClusterName>",
  "ClusterDescription": "<ClusterDescription>",
  "ClusterVersion": "<ClusterVersion>"
}
```

### 资源检查

```bash
# 7. 确认混部组件已安装
tccli tke DescribeAddon --region <Region> \
    --ClusterId CLUSTER_ID --AddonName craned
# expected: Status "Running"

tccli tke DescribeAddon --region <Region> \
    --ClusterId CLUSTER_ID --AddonName qosagent
# expected: Status "Running"

# 8. 确认节点有离线扩展资源
kubectl get nodes -o json | jq '.items[] | {
    name: .metadata.name,
    offline_cpu: .status.capacity["gocrane.io/cpu"]
}' | head -20
# expected: 原生节点显示 offline_cpu 有非零值
```

```json
{
  "Addons": "<Addons>",
  "AddonName": "<AddonName>",
  "AddonVersion": "<AddonVersion>",
  "RawValues": "<RawValues>",
  "Phase": "<Phase>",
  "Reason": "<Reason>"
}
```

### 版本与规格选择

| 组件 | 推荐版本 | 说明 |
|------|---------|------|
| Spark | 3.3.2 | 稳定版本，K8s 支持成熟 |
| Spark Operator | 2.3.0 | Helm Chart，支持 Volcano 集成 |
| Volcano | 5.12.2（客户端 JAR） | 批量调度器，支持 Gang Scheduling |
| Spark 镜像 | `ccr.ccs.tencentyun.com/use-test/spark:v3.3.2` | TCR 镜像，避免 Docker Hub 拉取超时 |
| Spark Operator 镜像 | `ccr.ccs.tencentyun.com/use-test/spark-operator:controller-2.3.0` | 替代 `ghcr.io/kubeflow/spark-operator` |

## 控制台与 CLI 参数映射

| 控制台操作 | tccli 命令 | 幂等 |
|-----------|-----------|:--:|
| 查看集群列表 | `DescribeClusters` | 是 |
| 查看集群节点 | `DescribeClusterInstances` | 是 |
| 查看混部组件状态 | `DescribeAddon` | 是 |
| 获取 kubeconfig | `DescribeClusterKubeconfig` | 是 |

> Spark 环境的安装、Spark Job 的提交和管理通过 kubectl + helm + spark-submit 完成（数据面操作）。

## 操作步骤

### 步骤 1：确认混部环境就绪

```bash
# 获取 kubeconfig
tccli tke DescribeClusterKubeconfig --region <Region> --ClusterId CLUSTER_ID \
    | jq -r '.Kubeconfig' > ~/.kube/config
# expected: kubeconfig 写入成功

# 确认集群状态
tccli tke DescribeClusters --region <Region> --ClusterIds '["CLUSTER_ID"]'
# expected: ClusterStatus "Running"

# 确认混部组件运行
tccli tke DescribeAddon --region <Region> \
    --ClusterId CLUSTER_ID --AddonName craned
# expected: Status "Running"
```

```json
{
  "Kubeconfig": "<Kubeconfig>",
  "RequestId": "<RequestId>"
}
```

| 占位符 | 说明 | 约束 | 获取方式 |
|--------|------|------|---------|
| `REGION` | 地域 | 如 `ap-guangzhou` | `tccli configure list` |
| `CLUSTER_ID` | 集群 ID | 格式 `cls-xxxxxxxx` | `tccli tke DescribeClusters` |

### 方案 A：spark-submit 提交（须 VPN/IOA）

#### 选择依据

- **spark-submit vs Spark Operator**：spark-submit 适合熟悉 Spark 命令行的用户，无需额外 K8s 资源；Spark Operator 适合需要声明式管理和批量调度的场景。
- **podTemplate**：必须在 Pod 模板中声明 `gocrane.io/cpu` 和 `gocrane.io/memory`，因为 spark-submit 不支持通过 `--conf` 指定非标准资源。

#### A.1 安装 Spark 环境

```bash
mkdir -p /usr/local/service/
cd /usr/local/service/
wget https://archive.apache.org/dist/spark/spark-3.3.2/spark-3.3.2-bin-hadoop3.tgz
tar -xzvf spark-3.3.2-bin-hadoop3.tgz
mv spark-3.3.2-bin-hadoop3 spark
# expected: /usr/local/service/spark/ 目录创建成功
```

#### A.2 创建 ServiceAccount 和 RBAC

```bash
kubectl create namespace workspark
# expected: namespace/workspark created
```

`spark-rbac.yaml`：

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: spark
  namespace: workspark
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: spark-cluster-admin-binding-workspark
subjects:
- kind: ServiceAccount
  name: spark
  namespace: workspark
roleRef:
  kind: ClusterRole
  name: cluster-admin
  apiGroup: rbac.authorization.k8s.io
```

```bash
kubectl apply -f spark-rbac.yaml
# expected: serviceaccount/spark created, clusterrolebinding created
```

#### A.3 准备 Pod 模板（含混部扩展资源）

**Driver Pod 模板** (`driver-pod-template.yaml`)：

```yaml
apiVersion: v1
kind: Pod
spec:
  containers:
    - name: spark-kubernetes-driver
      resources:
        limits:
          gocrane.io/cpu: "2"
          gocrane.io/memory: 2Gi
        requests:
          gocrane.io/cpu: "2"
          gocrane.io/memory: 2Gi
```

**Executor Pod 模板** (`executor-pod-template.yaml`)：

```yaml
apiVersion: v1
kind: Pod
spec:
  containers:
    - name: spark-kubernetes-executor
      resources:
        limits:
          gocrane.io/cpu: "2"
          gocrane.io/memory: 2Gi
        requests:
          gocrane.io/cpu: "2"
          gocrane.io/memory: 2Gi
```

#### A.4 创建 PodQOS 规则匹配 Spark Pod

```yaml
apiVersion: ensurance.crane.io/v1alpha1
kind: PodQOS
metadata:
  name: podqos-spark
spec:
  labelSelector:
    matchExpressions:
      - key: spark-role
        operator: In
        values:
          - driver
          - executor
  resourceQOS:
    cpuQOS:
      cpuPriority: 7
```

```bash
kubectl apply -f spark-podqos.yaml
# expected: podqos.ensurance.crane.io/podqos-spark created
```

#### A.5 获取 ApiServer 地址并提交 Job

```bash
# 获取 ApiServer URL
kubectl cluster-info | grep 'Kubernetes control plane'
# expected: Kubernetes control plane is running at https://APISERVER_URL
```

```bash
# 提交 Spark Pi 计算任务（最小可运行示例）
/usr/local/service/spark/bin/spark-submit \
  --master k8s://APISERVER_URL \
  --deploy-mode cluster \
  --name spark-pi \
  --class org.apache.spark.examples.SparkPi \
  --conf spark.executor.instances=2 \
  --conf spark.kubernetes.namespace=workspark \
  --conf spark.kubernetes.driver.container.image=ccr.ccs.tencentyun.com/use-test/spark:v3.3.2 \
  --conf spark.kubernetes.executor.container.image=ccr.ccs.tencentyun.com/use-test/spark:v3.3.2 \
  --conf spark.kubernetes.authenticate.driver.serviceAccountName=spark \
  --conf spark.kubernetes.driver.podTemplateFile=./driver-pod-template.yaml \
  --conf spark.kubernetes.executor.podTemplateFile=./executor-pod-template.yaml \
  --conf spark.kubernetes.driver.limit.cores=2 \
  --conf spark.kubernetes.driver.request.cores=0 \
  --conf spark.kubernetes.executor.limit.cores=2 \
  --conf spark.kubernetes.executor.request.cores=0 \
  --conf spark.driver.memory=2g \
  --conf spark.executor.memory=2g \
  local:///opt/spark/examples/jars/spark-examples_2.12-3.3.2.jar 20000
# expected: 提交成功，输出日志含 submission ID
```

**关键参数说明**：

| 参数 | 说明 | 约束 |
|------|------|------|
| `spark.kubernetes.namespace` | 命名空间 | 必须与 ServiceAccount namespace 一致 |
| `spark.kubernetes.driver.podTemplateFile` | Driver Pod 模板路径 | 必须声明 `gocrane.io/cpu` 扩展资源 |
| `spark.kubernetes.driver.request.cores=0` | 设 0 | 避免与扩展资源冲突（CPU 资源由 podTemplate 声明） |
| `spark.kubernetes.driver.limit.cores=2` | 设计划上限值 | 配合扩展资源使用 |
| `spark.executor.instances=2` | Executor 数量 | 调整以匹配可用离线资源量 |

### 方案 B：Spark Operator 提交（须 VPN/IOA）

#### 选择依据

- **Spark Operator vs spark-submit**：Operator 以 CRD（SparkApplication）声明任务，支持自动生命周期管理和批量调度集成。
- **batchScheduler**：如需 Gang Scheduling（等待所有 Executor 就绪后统一启动），选择 Volcano 调度器。

#### B.1 部署 Spark Operator

```bash
helm repo add spark-operator https://kubeflow.github.io/spark-operator
helm repo update
helm install spark-operator spark-operator/spark-operator \
    --version 2.3.0 \
    --namespace spark-operator \
    --create-namespace \
    --set controller.batchScheduler.enable=true
# expected: STATUS: deployed
```

> 如 `ghcr.io` 镜像拉取失败，替换为：`ccr.ccs.tencentyun.com/use-test/spark-operator:controller-2.3.0`
> 在 Helm values 中覆盖：`--set image.repository=ccr.ccs.tencentyun.com/use-test/spark-operator --set image.tag=controller-2.3.0`

确认 Operator 就绪：

```bash
kubectl wait --timeout=2m \
    -n spark-operator deployment/spark-operator-controller \
    --for=condition=Available
# expected: condition met
```

#### B.2 创建 ServiceAccount（如未创建）

同方案 A.2，确保 `spark` ServiceAccount 存在于目标 namespace。

#### B.3 编写并提交 SparkApplication（含混部资源）

`spark-pi-colocation.yaml`：

```yaml
apiVersion: "sparkoperator.k8s.io/v1beta2"
kind: SparkApplication
metadata:
  name: spark-pi
  namespace: workspark
spec:
  arguments:
  - "10000"
  type: Scala
  mode: cluster
  image: "ccr.ccs.tencentyun.com/use-test/spark:v3.3.2"
  imagePullPolicy: Always
  mainClass: org.apache.spark.examples.SparkPi
  mainApplicationFile: "local:///opt/spark/examples/jars/spark-examples_2.12-3.3.2.jar"
  sparkVersion: "3.3.2"
  restartPolicy:
    type: Never
  driver:
    memory: "2g"
    template:
      spec:
        containers:
        - name: spark-kubernetes-driver
          resources:
            limits:
              gocrane.io/cpu: "2"
              gocrane.io/memory: "2Gi"
            requests:
              gocrane.io/cpu: "2"
              gocrane.io/memory: "2Gi"
    labels:
      version: 3.3.2
    serviceAccount: spark
  executor:
    instances: 2
    memory: "2g"
    template:
      spec:
        containers:
        - name: spark-kubernetes-executor
          resources:
            limits:
              gocrane.io/cpu: "2"
              gocrane.io/memory: "2Gi"
            requests:
              gocrane.io/cpu: "2"
              gocrane.io/memory: "2Gi"
    labels:
      version: 3.3.2
```

```bash
kubectl apply -f spark-pi-colocation.yaml
# expected: sparkapplication.sparkoperator.k8s.io/spark-pi created
```

### 方案 C：集成 Volcano 批量调度器（须 VPN/IOA）

#### 选择依据

- **Volcano**：提供 Gang Scheduling（一组 Pod 同时启动/失败），适合需要 N 个 Executor 同时运行的场景。支持队列管理和资源预留。
- **何时使用**：Spark 任务需要固定的 N 个 Executor 同时就绪才能开始计算时。

#### C.1 部署 Volcano

```bash
helm repo add volcano-sh https://volcano-sh.github.io/helm-charts
helm repo update
helm install volcano volcano-sh/volcano \
    -n volcano-system --create-namespace
# expected: STATUS: deployed

kubectl get pods -n volcano-system
# expected: volcano-controllers 和 volcano-scheduler Pod Running
```

#### C.2 创建 Volcano Queue（含混部资源）

`volcano-queue.yaml`：

```yaml
apiVersion: scheduling.volcano.sh/v1beta1
kind: Queue
metadata:
  name: sparkqueue
spec:
  weight: 1
  reclaimable: false
  capability:
    gocrane.io/cpu: 20
    gocrane.io/memory: 100Gi
```

```bash
kubectl apply -f volcano-queue.yaml
# expected: queue.scheduling.volcano.sh/sparkqueue created
```

#### C.3 SparkApplication 配置 Volcano 调度

在 SparkApplication YAML 中增加：

```yaml
spec:
  batchScheduler: volcano
  batchSchedulerOptions:
    queue: "sparkqueue"
```

完整示例参见方案 B.3，增加以上两行。

## 验证

### 控制面（tccli）

```bash
# 1. 确认集群状态
tccli tke DescribeClusters --region <Region> --ClusterIds '["CLUSTER_ID"]'
# expected: ClusterStatus "Running"

# 2. 确认混部组件运行
tccli tke DescribeAddon --region <Region> \
    --ClusterId CLUSTER_ID --AddonName craned
# expected: Status "Running"
```

```json
{
  "TotalCount": "<TotalCount>",
  "Clusters": "<Clusters>",
  "ClusterId": "<ClusterId>",
  "ClusterName": "<ClusterName>",
  "ClusterDescription": "<ClusterDescription>",
  "ClusterVersion": "<ClusterVersion>"
}
```

### 数据面（须 VPN/IOA）

```bash
# 3. 确认 Spark Driver Pod 创建
kubectl get pods -n workspark | grep spark-pi
# expected: driver Pod STATUS Running，executor Pod STATUS Running 或 Completed

# 4. 确认 Pod 使用混部扩展资源
kubectl get pod DRIVER_POD_NAME -n workspark -o json | \
    jq '.spec.containers[0].resources'
# expected: limits 和 requests 包含 gocrane.io/cpu 和 gocrane.io/memory

# 5. 确认 Pod 有混部 QoS 注解
kubectl describe pod DRIVER_POD_NAME -n workspark | grep gocrane
# expected: 包含 gocrane.io/cpu-qos 和 gocrane.io/qos 注解

# 6. 查看 Spark 任务日志
kubectl logs DRIVER_POD_NAME -n workspark | grep "Pi is roughly"
# expected: Pi is roughly 3.14xxxxx

# 7. （Volcano）确认 PodGroup 和队列状态
kubectl get podgroups -A
# expected: PodGroup 存在且状态正常

# 8. （Volcano）确认调度器为 volcano
kubectl get pod DRIVER_POD_NAME -n workspark -o jsonpath='{.spec.schedulerName}'
# expected: volcano
```

```text
NAME  STATUS  AGE
...
```

## 清理

> **警告**：Spark Driver Pod 在任务完成后保留为 Completed 状态（便于查看日志），不自动删除。卸载 Spark Operator 不会删除已提交的 SparkApplication。清理前确认所有任务已完成。

### 数据面清理（须 VPN/IOA，在控制面之前）

```bash
# 1. 清理前状态检查
kubectl get sparkapplications -A
kubectl get pods -n workspark
# 确认是待清理的资源

# 2. 删除 SparkApplication（Spark Operator 方式）
kubectl delete sparkapplication spark-pi -n workspark
# expected: sparkapplication.sparkoperator.k8s.io "spark-pi" deleted

# 3. 删除残留 Spark Pod（spark-submit 方式）
kubectl delete pods -n workspark -l spark-role=driver
kubectl delete pods -n workspark -l spark-role=executor
# expected: Pod 已删除

# 4. 删除 PodQOS 规则
kubectl delete podqos podqos-spark
# expected: podqos.ensurance.crane.io "podqos-spark" deleted

# 5. 卸载 Spark Operator（如不再使用）
helm uninstall spark-operator -n spark-operator
# expected: release "spark-operator" uninstalled

# 6. 卸载 Volcano（如不再使用）
helm uninstall volcano -n volcano-system
# expected: release "volcano" uninstalled

# 7. 删除 workspace namespace
kubectl delete namespace workspark
# expected: namespace "workspark" deleted

# 8. 验证已删除
kubectl get pods -n workspark
# expected: No resources found
```

```text
NAME  STATUS  AGE
...
```

### 控制面

混部组件（Craned、QoSAgent）属于集群基础设施，一般保留。如需完整清理，参考 [TKE 集群混部原理与部署介绍](../TKE%20集群混部原理与部署介绍/tccli%20操作.md) 的清理节。

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `spark-submit` 报 `ClassNotFoundException` | 检查 jar 路径 | `mainApplicationFile` 路径不存在于镜像中 | 确认使用 `local:///opt/spark/examples/jars/...` 格式 |
| Spark Operator Pod 一直 `ImagePullBackOff` | `kubectl describe pod -n spark-operator` 查看 Events | `ghcr.io` 镜像拉取超时 | 使用 TCR 镜像：`ccr.ccs.tencentyun.com/use-test/spark-operator:controller-2.3.0` |
| `kubectl apply` SparkApplication 报 `no matches for kind` | `kubectl api-resources \| grep sparkoperator` | Spark Operator CRD 未安装 | 确认 `helm list -n spark-operator` 有 spark-operator Release |
| `helm install volcano` 超时 | `helm list -A` 查看 Release 状态 | 镜像拉取超时或网络延迟 | 重试安装；如持续失败使用代理 |

### 提交成功但任务异常

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| Driver Pod 一直 Pending | `kubectl describe pod DRIVER_POD_NAME -n workspark` | 节点无可用 `gocrane.io/cpu` 离线资源 | 减少资源请求或等待在线业务释放资源；检查 `kubectl describe node` 确认 `gocrane.io/cpu` 可用量 |
| Executor Pod 创建后 Pending | 同上 | 离线资源不足或调度到普通节点 | 确认所有 Executor 的 Pod 模板正确声明了 `gocrane.io/cpu` 资源 |
| Pod 使用普通 CPU 而非扩展资源 | `kubectl get pod POD_NAME -o yaml \| grep -A 5 resources` | Pod 模板中未声明 `gocrane.io/cpu`，回退到标准 cpu | 在 Pod 模板的 `limits` 和 `requests` 中显式声明 `gocrane.io/cpu` |
| Pod 无 `gocrane.io/qos` 注解 | `kubectl describe pod POD_NAME` 检查注解 | QoSAgent 未处理好该 Pod 或 PodQOS 规则未匹配 | 确认 PodQOS 的 labelSelector 与 Pod 标签一致（spark-role: driver/executor） |
| Volcano 模式下 Pod 不调度 | `kubectl describe podgroup PODGROUP_NAME` | Queue 资源配额不足或 minResources 不满足 | 增加 Queue 的 `gocrane.io/cpu` 容量或降低 minMember 要求 |
| Spark Pi 结果异常或超时 | `kubectl logs DRIVER_POD_NAME -n workspark` | 离线 Pod 被驱逐或 CPU 被压制导致计算中断 | 检查在线业务负载是否过高导致频繁驱逐；调整 NodeQOS 阈值 |

## 下一步

- [TKE 集群混部原理与部署介绍](../TKE%20集群混部原理与部署介绍/tccli%20操作.md) -- page_id `125430`
- [Crane 调度器介绍](../../调度配置/Crane%20调度器介绍/tccli%20操作.md) -- page_id `75472`
- [可抢占式 Job](../../调度配置/可抢占式%20Job/tccli%20操作.md)
- [Apache Spark on Kubernetes 官方文档](https://spark.apache.org/docs/latest/running-on-kubernetes.html)
- [Volcano 官方文档](https://volcano.sh/)

## 控制台替代

[容器服务控制台 - 工作负载](https://console.cloud.tencent.com/tke2/cluster)
