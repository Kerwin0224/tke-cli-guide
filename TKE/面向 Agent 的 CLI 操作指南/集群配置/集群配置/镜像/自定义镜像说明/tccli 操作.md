# 自定义镜像说明

> 对照官方：[自定义镜像说明](https://cloud.tencent.com/document/product/457/39563) · page_id `39563`

## 概述

用户可通过 TKE 提供的基础镜像制作自定义镜像，将预装软件、配置等固化到镜像中，用于节点初始化和运行。**自定义镜像属非标准操作环境，TKE 原则上不提供 SLA 服务保障。**

## 前置条件

- 已获取基础镜像（TKE 支持的公共镜像）
- 已创建 CVM 实例（使用基础镜像）
- 需[提交工单](https://console.cloud.tencent.com/workorder/category)申请自定义镜像功能权限
- CAM 权限：`tke:ModifyClusterImage`

## 控制台与 CLI 参数映射

| 控制台操作 | CLI | 幂等 |
|-----------|-----|:--:|
| 查看支持的公共镜像 | `tccli tke DescribeImages` | 是 |
| 创建自定义镜像 | CVM 控制台 / `tccli cvm CreateImage` | 否 |
| 修改集群操作系统 | `tccli tke ModifyClusterImage --ClusterId <ClusterId> --ImageId <ImageId>` | 是 |
| 修改节点池操作系统 | `tccli tke ModifyClusterNodePool --ClusterId <ClusterId> --NodePoolId <NodePoolId>` | 是 |

## 操作步骤

### 创建 CVM 实例（基础镜像）

```bash
tccli cvm RunInstances --region <Region> \
    --ImageId img-eb30mz89 \
    --InstanceType <InstanceType> \
    --Placement '{"Zone":"<ZONE>"}' \
    --VirtualPrivateCloud '{"VpcId":"vpc-example","SubnetId":"subnet-example"}'
# expected: exit 0，返回 InstanceIdSet
```

### 登录实例，执行清理脚本

```bash
# 制作自定义镜像前必须执行清理脚本（不要在集群已有节点内执行）
curl --proto '=https' --tlsv1.2 -sSf https://mirrors.tencent.com/install/tke/clean-node.sh | bash
```

**制作自定义镜像的关键注意事项：**

| 禁止 | 原因 |
|------|------|
| 修改 `/etc/fstab` | 可能导致节点启动失败 |
| 预装运行时组件 | 节点初始化会直接报错 |
| 保护 `/etc/resolv.conf`（`chattr +i`） | 导致 cloud-init 失败 |
| 保留 `/var/lib/cloud` 目录 | `config_scripts_user` 残留会影响 cloud-init |
| 保留 `/var/lib/cloud` 目录的脏数据 | 历史数据影响节点正常初始化 |
| 使用集群中正在运行的节点制作镜像 | 需先从集群移出 |
| 把 yum 源放到 `/etc/yum.repos.d/` | 会引起 agent 安装 yum 源失败 |

### 创建自定义镜像

```bash
# 登录 CVM 实例后，制作自定义内容（如创建测试文件）
echo "custom image test" > test.txt

# 创建自定义镜像（通过 CVM 控制台或 CLI）
tccli cvm CreateImage --region <Region> \
    --InstanceId <InstanceId> \
    --ImageName <MyCustomImage> \
    --ImageDescription "TKE custom image based on TencentOS Server 3.1"
# expected: exit 0，返回 ImageId
```

### 修改集群操作系统（控制面）

```bash
tccli tke ModifyClusterImage --region <Region> \
    --ClusterId <ClusterId> \
    --ImageId <CustomImageId>
# expected: exit 0
# 注意：仅影响后续新增和重装的节点，不影响存量节点
```

## 验证

```bash
# 确认自定义镜像已创建
tccli cvm DescribeImages --region <Region> \
    --ImageIds '["<CustomImageId>"]'
# expected: ImageState 为 NORMAL

# 确认集群镜像已更新
tccli tke DescribeClusters --region <Region> \
    --ClusterIds '["<ClusterId>"]'
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

## 清理

```bash
# 删除自定义镜像（不再需要时）
tccli cvm DeleteImages --region <Region> --ImageIds '["<CustomImageId>"]'
# expected: exit 0
```

## 排障

| 现象 | 根因 | 修复 |
|------|------|------|
| 控制台看不到自定义镜像 | 基础镜像不在 TKE 公共镜像列表内 | 确保用 TKE 支持的基础镜像制作 |
| 控制台看不到自定义镜像 | 自定义镜像与集群不在同一地域 | 创建（共享）自定义镜像到集群地域 |
| 节点加入集群失败 | `cloud-init` 执行失败 | 检查 `/etc/resolv.conf` 保护、`/var/lib/cloud` 清理 |
| 节点初始化报错 | 镜像中预装了运行时组件 | 重新制作镜像，不预装运行时 |
| 新建节点失败 | TKE 镜像逻辑调整 | 留意站内信通知，重新制作镜像 |

## 下一步

- [镜像概述](../镜像概述/tccli 操作.md)
- [TencentOS Rainbow 镜像介绍](../TencentOS Rainbow/镜像介绍/tccli 操作.md)

## 控制台替代

[TKE 控制台 → 创建集群/新建节点池](https://console.cloud.tencent.com/tke2/cluster)：在"镜像提供方"中选择"自定义镜像"，"操作系统"中选择已创建的自定义镜像。
