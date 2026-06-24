# 使用对象存储 COS

> 对照官方：[使用对象存储 COS](https://cloud.tencent.com/document/product/457/44232) · page_id `44232`

## 概述

TKE 支持通过 CSI 驱动将腾讯云对象存储 COS 挂载为 Pod 数据卷。COS 仅支持**静态配置**（先创建 COS 存储桶，再通过 PV/PVC 静态绑定）。挂载方式支持 COSFS 和 GooseFS-Lite 两种工具。

## 前置条件

- [环境准备](../../../../环境准备.md)
- 已安装 COS-CSI 组件（addon: `cos`）
- 已获取 SecretId 和 SecretKey（建议使用子账号密钥）
- 已在对象存储控制台创建存储桶，并获取子目录路径
- kubectl 可达集群以 apply YAML

> **kubectl 可达性说明**：当前共享集群 kubectl 不可达（公网端点 CAM 拒绝 strategyId:240463971）。以下 PV/PVC/Pod 创建步骤 YAML 完整，但 `kubectl apply` 无法真跑验证。

## 控制台与 CLI 参数映射

| 控制台操作 | CLI | 幂等 |
|-----------|-----|:--:|
| 安装 COS-CSI 组件 | `tccli tke InstallAddon --AddonName cos` | 否 |
| 查看已安装组件 | `tccli tke DescribeAddon` | 是 |
| 创建 Secret | `kubectl apply -f <yaml>` | 是 |
| 创建 PV（静态） | `kubectl apply -f <yaml>` | 否（名称冲突） |
| 创建 PVC | `kubectl apply -f <yaml>` | 否（名称冲突） |
| 创建工作负载挂载 PVC | `kubectl apply -f <yaml>` | 是 |

## 操作步骤

### 安装 COS-CSI 组件

```bash
tccli tke InstallAddon --region <Region> \
    --cli-input-json '{"ClusterId":"<ClusterId>","AddonName":"cos","AddonVersion":"<AddonVersion>"}'
# expected: exit 0
```

### 创建访问密钥 Secret

COS-CSI 通过 Secret 存储 SecretId 和 SecretKey（需 base64 编码）。

```yaml
apiVersion: v1
kind: Secret
type: Opaque
metadata:
  name: cos-secret
  namespace: kube-system
data:
  SecretId: <base64-encoded-secret-id>
  SecretKey: <base64-encoded-secret-key>
```

```bash
kubectl apply -f cos-secret.yaml
# expected: secret/cos-secret created
```

### 创建 PV（静态配置）

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
    nodePublishSecretRef:
      name: cos-secret
      namespace: kube-system
    volumeAttributes:
      url: http://cos.<REGION>.myqcloud.com
      bucket: <bucket-name>-<appid>
      path: /<sub-path>
    volumeHandle: cos-pv
  persistentVolumeReclaimPolicy: Retain
  volumeMode: Filesystem
```

```bash
kubectl apply -f cos-pv.yaml
# expected: persistentvolume/cos-pv created
```

### 创建 PVC

COS 仅支持静态配置，`storageClassName` 必须为空字符串。

```yaml
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
      storage: 1Gi
  volumeMode: Filesystem
  volumeName: cos-pv
  storageClassName: ""
```

```bash
kubectl apply -f cos-pvc.yaml
# expected: persistentvolumeclaim/cos-pvc created
```

### 创建 Pod 使用 COS PVC

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-cos
spec:
  containers:
  - name: pod-cos
    command: ["tail", "-f", "/etc/hosts"]
    image: centos:latest
    volumeMounts:
    - mountPath: /data
      name: cos
    resources:
      requests:
        memory: "128Mi"
        cpu: "0.1"
  volumes:
  - name: cos
    persistentVolumeClaim:
      claimName: cos-pvc
```

```bash
kubectl apply -f pod-cos.yaml
# expected: pod/pod-cos created
```

## 验证

### 控制面（tccli）

```bash
tccli tke DescribeAddon --region <Region> \
    --ClusterId <ClusterId> \
    | jq '.Addons[] | select(.AddonName == "cos") | {AddonName, AddonVersion, Status}'
# expected: Status 为 "Enabled"
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

### 数据面（kubectl）

```bash
kubectl get pv cos-pv
kubectl get pvc cos-pvc -n default
kubectl get pod pod-cos
# expected: PV/PVC Status 为 Bound，Pod Running
```

```text
NAME  STATUS  AGE
...
```

## 清理

### 数据面（kubectl）

```bash
kubectl delete pod pod-cos
kubectl delete pvc cos-pvc -n default
kubectl delete pv cos-pv
kubectl delete secret cos-secret -n kube-system
```

### 控制面（tccli）

```bash
tccli tke DeleteAddon --region <Region> \
    --ClusterId <ClusterId> \
    --AddonName cos
# expected: exit 0
```

## 排障

| 现象 | 根因 | 修复 |
|------|------|------|
| Pod 挂载 COS 失败 | Secret 中的 SecretId/SecretKey 无效或 base64 编码错误 | 检查 Secret，用 `echo -n '<value>' \| base64` 重新编码 |
| PVC 一直 Pending | COS 仅支持静态配置，但 `storageClassName` 误填 | 确保 `storageClassName: ""`，`volumeName` 指定已创建的 PV |
| GooseFS-Lite 挂载子目录报错 | GooseFS-Lite 要求子目录已存在 | 在 COS 控制台预先创建子目录 |

## 下一步

- [使用数据加速器 GooseFS](../使用数据加速器 GooseFS/tccli 操作.md)
- [使用文件存储 CFS](../使用文件存储 CFS/文件存储使用说明/tccli 操作.md)

## 控制台替代

[TKE 控制台 → 集群详情 → 存储 → PersistentVolume](https://console.cloud.tencent.com/tke2/cluster)：新建 PV 选择"对象存储 COS" Provisioner。
