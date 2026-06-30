# 使用对象存储 COS

> 对照官方：[使用对象存储 COS](https://cloud.tencent.com/document/product/457/44232) · page_id `44232` · tccli ≥3.1.107 · API 2018-05-25

## 概述

在 TKE 集群中通过安装 COS-CSI 组件（cosfs 驱动），以静态 PV/PVC 方式将腾讯云对象存储（COS）存储桶挂载到 Pod。COS 仅支持静态 PV 绑定，不支持通过 StorageClass 动态供给。

**重要**：`tccli cos` 产品不存在——经对 300+ 产品全量扫描确认 tccli 无 COS 模块。COS 存储桶的创建、配置、删除操作需通过 [COS 控制台](https://console.cloud.tencent.com/cos5) 完成，本文档仅覆盖 TKE 集群侧的组件安装与 PV/PVC 绑定。

## 前置条件

- [环境准备](../../../环境准备.md)

### 环境检查

```bash
# 1. 检查 tccli 版本
tccli --version
# expected: tccli version >= 1.0.0

# 2. 检查当前凭据和地域
tccli configure list
# expected: secretId, secretKey, region 均已配置，region 为目标地域

# 3. 检查 CAM 权限 — 需要以下 Action 名
#    tke:InstallAddon, tke:DescribeAddon, tke:DescribeClusters
# 验证：执行 DescribeClusters 确认 TKE 权限
tccli tke DescribeClusters --region <REGION>
# expected: exit 0，返回集群列表（可为空）
```

### 资源检查

```bash
# 4. 确认目标集群存在且状态正常
tccli tke DescribeClusters --region <REGION> --ClusterIds '["<CLUSTER_ID>"]'
# expected: TotalCount >= 1，ClusterStatus: "Running"
```

**预期输出**：

```json
{
    "TotalCount": 1,
    "Clusters": [
        {
            "ClusterId": "cls-example",
            "ClusterName": "example-cluster",
            "ClusterVersion": "1.32.2",
            "ClusterStatus": "Running",
            "ClusterType": "MANAGED_CLUSTER",
            "ClusterNetworkSettings": {
                "VpcId": "vpc-example",
                "SubnetId": "subnet-example"
            }
        }
    ],
    "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

> **kubectl 连通性说明**：当前环境外网端点被组织级 CAM 策略 `strategyId:240463971`（条件 `tke:clusterExtranetEndpoint=true`）硬拒绝，自建安全组无法绕过。内网端点仅 VPC 内可达，集群域名（如 `cls-xxx.ccs.tencent-cloud.com`）外网 DNS 不可解析。本文档中所有 kubectl 命令的 YAML 已通过 Python 语法校验，实际执行需通过 IOA/VPN 或同 VPC CVM 操作。

## 控制台与 CLI 参数映射

| 控制台操作 | tccli / kubectl 命令 | 幂等 |
|-----------|---------------------|:--:|
| 安装 COS-CSI 组件 | `tccli tke InstallAddon --AddonName cos` | 否 |
| 查询组件状态 | `tccli tke DescribeAddon --AddonName cos` | 是 |
| 创建 COS 存储桶 | COS 控制台（tccli 无 cos 产品） | — |
| 创建访问凭证 Secret | `kubectl create secret generic` | 否 |
| 创建 PV/PVC 绑定 COS | `kubectl apply -f cos-pv-pvc.yaml` | 否 |

## 关键字段说明

以下说明 `InstallAddon` 的关键参数。完整参数定义见 `tccli tke InstallAddon --generate-cli-skeleton`。

| 字段 | 类型 | 必填 | 取值与约束 | 错误后果 |
|------|------|:--:|------|------|
| `AddonName` | String | 是 | 固定值 `cos`（**必须小写**）。大写 `COS` 会被 API 拒绝 | 填 `COS`（大写）→ `UnknownParameter` |
| `AddonVersion` | String | 否 | 组件版本号，如 `1.0.12`。不指定时安装最新版本 | 指定不存在的版本 → "addon version not found" |
| `ClusterId` | String | 是 | 目标集群 ID，格式 `cls-xxxxxxxx` | 集群不存在 → `ResourceNotFound.ClusterNotFound` |
| `RawValues` | String | 否 | 组件自定义配置 JSON 字符串的 Base64 编码 | 格式错误 → `InvalidParameter.RawValues` |

## 操作步骤

### 步骤 1：安装 COS-CSI 组件

#### 选择依据

- **AddonName**：必须用小写 `cos`。经真实验证，大写 `COS` 返回 `UnknownParameter` 错误（而非 `InvalidParameter`），这是一个 API 摩擦——大小写不一致导致参数被识别为未知而非无效。
- **AddonVersion**：不指定即可安装最新版（经实测安装 `1.0.12`）。手动指定不存在的版本会返回 "addon version not found"。
- **ClusterId**：确保目标集群处于 `Running` 状态、类型为 `MANAGED_CLUSTER`。

#### 最小安装（不指定版本，安装最新）

```bash
tccli tke InstallAddon --region <REGION> \
    --ClusterId <CLUSTER_ID> \
    --AddonName cos
# expected: exit 0，返回 RequestId
```

**预期输出**：

```json
{
    "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

#### 增强配置（指定版本）

```bash
tccli tke InstallAddon --region <REGION> \
    --ClusterId <CLUSTER_ID> \
    --AddonName cos \
    --AddonVersion <ADDON_VERSION>
# expected: exit 0，返回 RequestId
```

| 占位符 | 说明 | 获取方式 |
|--------|------|---------|
| `<ADDON_VERSION>` | COS-CSI 组件版本 | 通过 `DescribeAddon` 查看已安装版本，或查阅 [COS-CSI 说明](https://cloud.tencent.com/document/product/457/49258) |

### 步骤 2：轮询组件安装状态

组件安装是异步操作。轮询直到 `Phase` 为 `Succeeded`。

> **注意**：`DescribeAddon` 返回的是 `Addons` 数组（复数），而非单个 `Addon` 对象。访问时需用 `jq '.Addons[0]'` 等方式取第一个元素。

```bash
tccli tke DescribeAddon --region <REGION> --ClusterId <CLUSTER_ID> --AddonName cos
# expected: Addons[0].Phase: "Succeeded"
```

**预期输出**：

```json
{
    "Addons": [
        {
            "AddonName": "cos",
            "AddonVersion": "1.0.12",
            "RawValues": "eyJnbG9iYWwiOnsiY2x1c3RlciI6eyJoaWdoQXZhaWxhYmlsaXR5Ijp0cnVlfX19",
            "Phase": "Succeeded",
            "Reason": "",
            "CreateTime": "2026-06-23T08:20:09Z"
        }
    ],
    "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

### 步骤 3：创建 COS 访问凭证 Secret

COS-CSI 通过 Kubernetes Secret 读取访问 COS 的 SecretId/SecretKey，Pod 挂载时自动使用。

**Secret 必须创建在 `kube-system` 命名空间**（依据官方文档要求），而非 `default`。后续 PV 的 `nodePublishSecretRef.namespace` 需与此一致。

```bash
kubectl create secret generic cos-secret \
    --from-literal=SecretId=<SECRET_ID> \
    --from-literal=SecretKey=<SECRET_KEY> \
    -n kube-system
# expected: secret/cos-secret created
```

**预期输出**：

```text
secret/cos-secret created
```

| 占位符 | 说明 | 获取方式 |
|--------|------|---------|
| `<SECRET_ID>` | 腾讯云 API 密钥 SecretId | [访问管理 → API 密钥管理](https://console.cloud.tencent.com/cam/capi) |
| `<SECRET_KEY>` | 腾讯云 API 密钥 SecretKey | 同上 |

### 步骤 4：创建 PV 和 PVC

#### 选择依据

- **driver**：`com.tencent.cloud.csi.cosfs`，COS-CSI 组件注册的 CSI 驱动名。
- **不设置 storageClassName**：CSI 驱动直接处理卷供给，静态绑定时 PV 和 PVC 均无需 `storageClassName` 字段。添加此字段可能导致绑定异常。
- **url**：使用 `http://` 协议（非 `https://`）。COS-CSI 驱动通过 HTTP 端点 `http://cos.<REGION>.myqcloud.com` 连接。
- **accessModes**：`ReadWriteMany`，COS 支持多 Pod 并发读写。
- **persistentVolumeReclaimPolicy**：`Retain`，删除 PV 时保留 COS 存储桶中的数据。
- **nodePublishSecretRef.namespace**：必须为 `kube-system`，与 Secret 所在命名空间一致。

#### 配置清单

`cos-pv-pvc.yaml`：

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: cos-pv
spec:
  accessModes:
  - ReadWriteMany
  capacity:
    storage: 10Gi
  csi:
    driver: com.tencent.cloud.csi.cosfs
    volumeHandle: cos-pv
    volumeAttributes:
      url: "http://cos.<REGION>.myqcloud.com"
      bucket: "<BUCKET_NAME>-<APPID>"
      path: "/"
    nodePublishSecretRef:
      name: cos-secret
      namespace: kube-system
  persistentVolumeReclaimPolicy: Retain
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: cos-pvc
  namespace: default
spec:
  accessModes:
  - ReadWriteMany
  resources:
    requests:
      storage: 10Gi
  volumeName: cos-pv
```

```bash
kubectl apply -f cos-pv-pvc.yaml
# expected: persistentvolume/cos-pv created, persistentvolumeclaim/cos-pvc created
```

**预期输出**：

```text
persistentvolume/cos-pv created
persistentvolumeclaim/cos-pvc created
```

| 占位符 | 说明 | 获取方式 |
|--------|------|---------|
| `<BUCKET_NAME>-<APPID>` | COS 存储桶完整名称（含 APPID 后缀） | [COS 控制台](https://console.cloud.tencent.com/cos5/bucket) → 存储桶列表，完整名称如 `example-1250000000` |

### 步骤 5：创建测试 Pod（可选）

验证 COS 存储卷可正常挂载和读写。

`cos-test-pod.yaml`：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: cos-test-pod
  namespace: default
spec:
  containers:
  - name: test
    image: busybox:latest
    command: ["sleep", "3600"]
    volumeMounts:
    - name: cos-volume
      mountPath: /mnt/cos
  volumes:
  - name: cos-volume
    persistentVolumeClaim:
      claimName: cos-pvc
```

```bash
kubectl apply -f cos-test-pod.yaml
# expected: pod/cos-test-pod created

kubectl exec cos-test-pod -- ls /mnt/cos
# expected: 列出 COS 存储桶根目录下的文件
```

## 验证

### 控制面（tccli）

| 维度 | 命令 | 预期 |
|------|------|------|
| 组件状态 | `tccli tke DescribeAddon --region <REGION> --ClusterId <CLUSTER_ID> --AddonName cos` | `Addons[0].Phase: "Succeeded"` |
| 集群状态 | `tccli tke DescribeClusters --region <REGION> --ClusterIds '["<CLUSTER_ID>"]'` | `ClusterStatus: "Running"` |

### 数据面（kubectl）

```bash
kubectl get pv cos-pv
# expected: STATUS: Available（PVC 未绑定时）或 Bound（已绑定）

kubectl get pvc cos-pvc -n default
# expected: STATUS: Bound
```

**预期输出**：

```text
NAME     CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM
cos-pv   10Gi       RWX            Retain           Available
```

```text
NAME      STATUS   VOLUME   CAPACITY   ACCESS MODES
cos-pvc   Bound    cos-pv   10Gi       RWX
```

> **kubectl 不可达提示**：如 `kubectl` 返回 `Unable to connect to the server`，参见 [排障](#排障) 中关于集群端点连通性的排查路径。此时 tccli 层面的验证仍可正常进行。

## 清理

> **⚠️ 副作用警告**：`kubectl delete pv cos-pv` 只删除 Kubernetes PV 资源，**不会删除 COS 存储桶及其中数据**。存储桶数据需在 [COS 控制台](https://console.cloud.tencent.com/cos5/bucket) 手动清空并删除，否则持续产生 COS 存储费用。
>
> **数据安全警告**：删除 PV 前，确认存储桶中的数据已备份或不再需要。`persistentVolumeReclaimPolicy: Retain` 策略保留后端数据，但误操作仍可能导致数据不可访问。

### 1. 清理前状态检查

```bash
kubectl get pv cos-pv
kubectl get pvc cos-pvc -n default
kubectl get pod cos-test-pod -n default 2>/dev/null
# 确认是待删除的目标资源
```

### 2. 删除测试 Pod（如已创建）

```bash
kubectl delete pod cos-test-pod -n default
# expected: pod "cos-test-pod" deleted
```

### 3. 删除 PVC，再删 PV

按依赖倒序：先删 PVC（解除绑定），再删 PV。

```bash
kubectl delete pvc cos-pvc -n default
# expected: persistentvolumeclaim "cos-pvc" deleted

kubectl delete pv cos-pv
# expected: persistentvolume "cos-pv" deleted
```

### 4. 删除 Secret（可选）

```bash
kubectl delete secret cos-secret -n kube-system
# expected: secret "cos-secret" deleted
```

### 5. 验证已删除

```bash
kubectl get pv cos-pv 2>&1
# expected: Error from server (NotFound): ... "cos-pv" not found

kubectl get pvc cos-pvc -n default 2>&1
# expected: Error from server (NotFound): ... "cos-pvc" not found
```

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `InstallAddon` 返回 `UnknownParameter` | 检查请求中 `AddonName` 的值 | 填写了大写 `COS`——API 只接受小写 `cos`。经真实验证：大写触发 `UnknownParameter`（而非 `InvalidParameter`），属于 API 大小写不一致的摩擦 | 改用 `--AddonName cos`（全小写）。保留 RequestId 以备排查 |
| `InstallAddon` 返回 "addon version not found" | `tccli tke DescribeAddon --region <REGION> --ClusterId <CLUSTER_ID> --AddonName cos` 查看当前已安装版本 | 指定的 `--AddonVersion` 在目标集群中不存在 | 去掉 `--AddonVersion` 参数以安装最新版，或填入 `DescribeAddon` 返回的实际版本号 |
| `DescribeAddon` 返回空 / 无数据 | 执行 `tccli tke DescribeAddon --region <REGION> --ClusterId <CLUSTER_ID> --AddonName cos` 并检查响应结构 | `DescribeAddon` 返回 `Addons`（数组），不是 `Addon`（对象）。用 `.Addon` 访问结果为 `null` | 用 `jq '.Addons[0]'` 获取数组第一个元素 |
| `tccli cos` 返回 "command not found" 或类似错误 | 执行 `tccli help` 列出所有可用产品 | `tccli` 不含 `cos` 产品。对 300+ 产品的全量扫描确认无 COS 模块 | 使用 [COS 控制台](https://console.cloud.tencent.com/cos5) 管理存储桶。此为产品能力缺口，非配置错误 |
| `kubectl` 返回 `Unable to connect to the server` | `tccli tke DescribeClusterEndpoints --region <REGION> --ClusterId <CLUSTER_ID>` 检查端点状态 | 公网端点被组织级 CAM 策略 `strategyId:240463971` 拒绝（条件 `tke:clusterExtranetEndpoint=true`）。自建安全组无法绕过此策略。内网端点仅 VPC 内可达，集群域名外网 DNS 不可解析 | 通过 IOA/VPN、同 VPC CVM 或专线访问。不使用公网端点（`--IsExtranet false`） |
| `kubectl apply` 返回 "driver com.tencent.cloud.csi.cosfs not found" | `kubectl get csidriver` 查看已注册的 CSI 驱动 | COS-CSI 组件未安装或未就绪 | 回到步骤 1 安装组件，等待 `Phase` 为 `Succeeded`。确认 `AddonName` 为小写 `cos` |

### 安装成功但 PVC 或挂载异常

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| PVC 一直 `Pending` | `kubectl describe pvc cos-pvc -n default` 查看 Events | PV 不存在，或 PVC 的 `volumeName` 与 PV 的 `name` 不匹配，或 PV/PVC 中误加了 `storageClassName` | 确认 PV 已创建（`kubectl get pv cos-pv`）。确认 PV 和 PVC 均无 `storageClassName` 字段。确认 PVC 的 `volumeName` 与 PV 的 `metadata.name` 一致 |
| Pod 挂载 COS 失败，Events 含 CSI 错误 | `kubectl describe pod <POD_NAME>` 查看 Events 详情 | Secret 中的 SecretId/SecretKey 错误、存储桶 URL 错误或 Secret 所在命名空间错误 | 检查 Secret 在 `kube-system` 命名空间（非 `default`）。检查 PV 中 `url` 使用 `http://`（非 `https://`）。确认存储桶名含 `-<APPID>` 后缀 |
| Pod 挂载成功但无法写入 | `kubectl exec <POD_NAME> -- touch /mnt/cos/test-write` 测试写入 | API 密钥无 `cos:PutObject` 权限，或存储桶 ACL 限制写入 | 在 [访问管理](https://console.cloud.tencent.com/cam/capi) 确认密钥拥有 COS 写入权限。在 [COS 控制台](https://console.cloud.tencent.com/cos5/bucket) 检查存储桶 ACL |
| 挂载后文件列表与预期不符 | `kubectl exec <POD_NAME> -- ls -la /mnt/cos` | PV 中 `volumeAttributes.path` 配置为子目录而非根目录 | 如需挂载存储桶根目录，设 `path: "/"`。如需挂载子目录，确认路径存在且拼写正确 |
| 集群网络连通性阻断 kubectl | `tccli tke DescribeClusterEndpoints --region <REGION> --ClusterId <CLUSTER_ID>` 查看 `ClusterExternalEndpoint` | 外网端点被 CAM 策略硬拒绝：`ClusterExternalEndpoint` 为空字符串。此为组织级安全策略，非个人权限问题 | 使用 `ClusterIntranetEndpoint`（如 `172.24.0.12`）。通过 IOA/VPN、专线或同 VPC CVM 执行 kubectl 命令 |

## 下一步

- [PV 和 PVC 的绑定规则](../PV%20和%20PVC%20的绑定规则/tccli%20操作.md)
- [使用云硬盘 CBS](../使用云硬盘%20CBS/PV%20和%20PVC%20管理云硬盘/tccli%20操作.md)
- [使用文件存储 CFS](../使用文件存储%20CFS/PV%20和%20PVC%20管理文件存储/tccli%20操作.md)
- [COS 控制台 — 管理存储桶](https://console.cloud.tencent.com/cos5/bucket)

## 控制台替代

[控制台 → 集群 → 组件管理](https://console.cloud.tencent.com/tke2/cluster) 安装 COS-CSI 组件；[COS 控制台](https://console.cloud.tencent.com/cos5) 管理存储桶。
