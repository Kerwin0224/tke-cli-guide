# 为数据盘设置文件系统卷标（tccli）

> 对照官方：[为数据盘设置文件系统卷标](https://cloud.tencent.com/document/product/457/82843) · page_id `82843`

## 概述

TKE 节点数据盘默认无卷标，通过 `e2label` 或 `tune2fs` 设置卷标，便于通过 `/dev/disk/by-label/<label>` 引用。适用于需要通过固定路径挂载数据盘到 Pod 的场景（如 hostPath 或 local PV）。通过 tccli 检查集群和节点，SSH 到节点操作磁盘。

> **kubectl 数据面不可达**：CAM拒绝公网端点(strategyId:240463971)，内网需VPN/IOA。以下 kubectl 命令为参考格式，需在内网/VPN 环境下执行。

## 前置条件

- [环境准备](../../环境准备.md)
- 节点 SSH 可登录
- 数据盘未挂载或可卸载

### 环境检查

```bash
tccli --version
# 预期输出: tccli version X.X.X
```

### 资源检查

```bash
tccli tke DescribeClusters --region ap-guangzhou --ClusterIds '["cls-xxxxxxxx"]'
```

**预期输出**：

```json
{
  "TotalCount": 1,
  "Clusters": [
    {
      "ClusterId": "cls-xxxxxxxx",
      "ClusterName": "my-test-cluster",
      "ClusterStatus": "Running",
      "ClusterVersion": "1.30.0",
      "ClusterType": "MANAGED_CLUSTER"
    }
  ]
}
```

```bash
tccli tke DescribeClusterInstances --region ap-guangzhou --ClusterId cls-xxxxxxxx
```

**预期输出**：

```json
{
  "TotalCount": 3,
  "InstanceSet": [
    {
      "InstanceId": "ins-xxxxxxxx",
      "InstanceRole": "WORKER",
      "InstanceState": "running",
      "LanIP": "10.0.1.10"
    }
  ]
}
```

## 控制台与 CLI 参数映射

| 控制台操作 | tccli / CLI | 幂等 |
|---|---|---|
| 查看集群状态 | `tccli tke DescribeClusters --region ap-guangzhou --ClusterIds '["cls-xxxxxxxx"]'` | 是 |
| 查看节点实例 | `tccli tke DescribeClusterInstances --region ap-guangzhou --ClusterId cls-xxxxxxxx` | 是 |
| 查看磁盘信息 | SSH 到节点 `lsblk` | 是 |
| 设置卷标 | SSH 到节点 `e2label /dev/vdb <label>` | 否 |
| 查看卷标 | SSH 到节点 `blkid /dev/vdb` | 是 |

## 操作步骤

### 步骤1：控制面检查集群和节点

```bash
tccli tke DescribeClusters --region ap-guangzhou --ClusterIds '["cls-xxxxxxxx"]'
```

**预期输出**：

```json
{
  "Clusters": [
    {
      "ClusterId": "cls-xxxxxxxx",
      "ClusterStatus": "Running",
      "ClusterVersion": "1.30.0"
    }
  ]
}
```

### 步骤2：SSH 到节点查看磁盘

```bash
ssh root@<node-ip>
lsblk
```

**预期输出**：

```
NAME   MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
vda    252:0    0   50G  0 disk
|-vda1 252:1    0   50G  0 part /
vdb    252:16   0  100G  0 disk /data
```

确认数据盘设备名（如 `/dev/vdb`）和当前挂载点。

### 步骤3：卸载数据盘（如需）

如果数据盘已挂载，需要先卸载：

```bash
ssh root@<node-ip>
umount /data
# 或
umount /dev/vdb
```

数据盘上有 Pod 正在使用时需先驱逐该节点上的 Pod。

### 步骤4：设置文件系统卷标

**对于 ext2/ext3/ext4 文件系统：**

```bash
e2label /dev/vdb DATA_DISK
# 预期: 无输出，设置成功
```

或使用 `tune2fs`：

```bash
tune2fs -L DATA_DISK /dev/vdb
# 预期: tune2fs 1.46.5 ... 设置成功
```

**对于 xfs 文件系统：**

```bash
xfs_admin -L DATA_DISK /dev/vdb
```

### 步骤5：重新挂载并验证

```bash
mount /dev/vdb /data

# 验证卷标
blkid /dev/vdb
```

**预期输出**：

```
/dev/vdb: LABEL="DATA_DISK" UUID="a1b2c3d4-..." TYPE="ext4"
```

```bash
# 通过卷标路径验证
ls -la /dev/disk/by-label/DATA_DISK
```

**预期输出**：

```
lrwxrwxrwx 1 root root 10 ... /dev/disk/by-label/DATA_DISK -> ../../vdb
```

### 步骤6：在 Pod 中通过卷标挂载

设置好卷标后，可在 Pod 中通过固定路径引用：

```yaml
volumes:
- name: data
  hostPath:
    path: /dev/disk/by-label/DATA_DISK
    type: Directory

# 或直接挂载为挂载点
volumes:
- name: data
  hostPath:
    path: /data
    type: Directory
```

使用卷标的好处：设备名（如 `/dev/vdb`）可能在重启后变化，但卷标不变，保证挂载可靠性。

## 验证

### 控制面（tccli）

```bash
tccli tke DescribeClusters --region ap-guangzhou --ClusterIds '["cls-xxxxxxxx"]'
```

**预期输出**：

```json
{
  "Clusters": [
    {
      "ClusterId": "cls-xxxxxxxx",
      "ClusterStatus": "Running"
    }
  ]
}
```

### 数据面（需 VPN/IOA 或 SSH）

```bash
ssh root@<node-ip> "blkid /dev/vdb | grep LABEL"
# 预期: LABEL="DATA_DISK"

ssh root@<node-ip> "ls -la /dev/disk/by-label/DATA_DISK"
# 预期: 符号链接指向正确的块设备
```

## 清理

无需特殊清理。卷标设置持久化在文件系统中，节点重启不会丢失。如需移除卷标：

```bash
e2label /dev/vdb ""
# 或
tune2fs -L "" /dev/vdb
```

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|---|---|---|---|
| `e2label` 报 `Bad magic number` | `blkid /dev/vdb` 查看文件系统类型 | 文件系统不是 ext2/ext3/ext4 | 使用对应文件系统工具：xfs 用 `xfs_admin`，btrfs 用 `btrfs filesystem label` |
| `umount` 报 `target is busy` | `lsof /data` 查看占用进程 | 有进程在使用该挂载点 | 停止占用进程；或驱逐节点上的 Pod 后再卸载 |
| `/dev/disk/by-label/` 无对应设备 | `ls /dev/disk/by-label/` 确认 | 卷标设置后未触发 udev | 运行 `udevadm trigger` 或重启节点 |

### 操作成功但异常

| 现象 | 诊断 | 根因 | 修复 |
|---|---|---|---|
| 节点重启后磁盘挂载失败 | 检查 `/etc/fstab` 配置 | 仅设置了卷标但未在 fstab 中配置自动挂载 | 在 `/etc/fstab` 添加：`LABEL=DATA_DISK /data ext4 defaults 0 2` |
| 多节点卷标冲突 | 不同节点上同名卷标指向不同大小的磁盘 | 节点规格不一致但批量执行了相同脚本 | 使用差异化卷标命名（如包含节点 IP 后缀） |

## 下一步

- [节点磁盘爆满排障处理](../节点磁盘爆满排障处理/tccli%20操作.md) -- page_id `43126`
- [使用云硬盘 CBS](../../集群配置/存储管理/使用云硬盘%20CBS/云硬盘使用说明/tccli%20操作.md)

## 控制台替代

无控制台界面，需 SSH 登录节点操作。
