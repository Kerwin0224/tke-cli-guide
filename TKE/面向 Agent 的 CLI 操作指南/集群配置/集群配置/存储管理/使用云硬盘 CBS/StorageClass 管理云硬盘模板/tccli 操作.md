# StorageClass 管理云硬盘模板

> 对照官方：[StorageClass 管理云硬盘模板](https://cloud.tencent.com/document/product/457/44239) · page_id `44239`

## 概述

通过创建 CBS 类型 StorageClass，配合 PVC 动态创建云硬盘。TKE 默认提供 `cbs` StorageClass（高性能云硬盘、随机可用区、按量计费），用户可按需自定义。

## 前置条件

- [环境准备](../../../环境准备.md)
- CBS-CSI 组件（addon: `cbs`）为系统默认预装
- kubectl 可达集群以 apply YAML

> **kubectl 可达性说明**：当前共享集群 kubectl 不可达。以下 YAML 完整，但 `kubectl apply` 无法真跑验证。

## 控制台与 CLI 参数映射

| 控制台操作 | CLI | 幂等 |
|-----------|-----|:--:|
| 创建 StorageClass | `kubectl apply -f <yaml>` | 否（名称冲突） |
| 创建 PVC | `kubectl apply -f <yaml>` | 否（名称冲突） |
| 创建 StatefulSet 挂载 PVC | `kubectl apply -f <yaml>` | 是 |

## 操作步骤

### 创建 StorageClass

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: cloud-premium
provisioner: com.tencent.cloud.csi.cbs
parameters:
  diskType: CLOUD_PREMIUM
  diskZone: <REGION>-<ZONE>
  diskChargeType: POSTPAID_BY_HOUR
reclaimPolicy: Retain
volumeBindingMode: WaitForFirstConsumer
```

Parameters 参数说明：

| 参数 | 描述 |
|------|------|
| `diskType` | `CLOUD_PREMIUM`（高性能）、`CLOUD_SSD`（SSD）、`CLOUD_HSSD`（增强型SSD）、`CLOUD_BSSD`（通用型SSD） |
| `diskZone` | 指定可用区，不指定则随机选择 |
| `diskChargeType` | `POSTPAID_BY_HOUR`（按量计费）或 `PREPAID`（包年包月） |
| `diskChargePrepaidPeriod` | 包年包月时长（月），支持 1~12、24、36 |
| `diskChargePrepaidRenewFlag` | 自动续费标识：`NOTIFY_AND_AUTO_RENEW`、`NOTIFY_AND_MANUAL_RENEW`、`DISABLE_NOTIFY_AND_MANUAL_RENEW` |
| `volumeBindingMode` | `Immediate`（立即绑定）或 `WaitForFirstConsumer`（延迟调度，推荐） |
| `reclaimPolicy` | `Delete`（删除）或 `Retain`（保留） |
| `aspid` | 自动快照策略 ID |

```bash
kubectl apply -f cbs-storageclass.yaml
# expected: storageclass.storage.k8s.io/cloud-premium created
```

### 创建多实例 StatefulSet（自动创建 PVC）

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web
spec:
  selector:
    matchLabels:
      app: nginx
  serviceName: "nginx"
  replicas: 3
  template:
    metadata:
      labels:
        app: nginx
    spec:
      terminationGracePeriodSeconds: 10
      containers:
      - name: nginx
        image: nginx
        ports:
        - containerPort: 80
          name: web
        volumeMounts:
        - name: www
          mountPath: /usr/share/nginx/html
  volumeClaimTemplates:
  - metadata:
      name: www
    spec:
      accessModes: [ "ReadWriteOnce" ]
      storageClassName: cloud-premium
      resources:
        requests:
          storage: 20Gi
```

```bash
kubectl apply -f web-statefulset.yaml
# expected: statefulset.apps/web created
```

## 验证

```bash
kubectl get sc cloud-premium
kubectl get pvc -l app=nginx
# expected: StorageClass 存在，PVC 自动创建
```

```text
NAME  STATUS  AGE
...
```

## 清理

```bash
kubectl delete statefulset web
kubectl delete pvc -l app=nginx
kubectl delete sc cloud-premium
```

## 排障

| 现象 | 根因 | 修复 |
|------|------|------|
| PVC 一直 Pending | 可用区不支持所选云盘类型 | 检查 `diskZone` 和 `diskType` 是否匹配 |
| 包年包月 PVC 创建失败 | TKE_QCSRole 缺少 `QcloudCVMFinanceAccess` 策略 | 为角色添加支付权限 |
| CBS 扩容不生效 | 未启用在线扩容 | 确保 StorageClass 启用 `allowVolumeExpansion: true` |

## 下一步

- [PV 和 PVC 管理云硬盘](../PV 和 PVC 管理云硬盘/tccli 操作.md)
- [云硬盘使用说明](../云硬盘使用说明/tccli 操作.md)

## 控制台替代

[TKE 控制台 → 集群详情 → 存储 → StorageClass](https://console.cloud.tencent.com/tke2/cluster)：新建 → 选择"云硬盘 CBS(CSI)" Provisioner。
