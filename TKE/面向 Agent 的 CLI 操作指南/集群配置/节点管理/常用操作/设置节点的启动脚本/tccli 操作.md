# 设置节点的启动脚本（tccli）

> 对照官方：[设置节点的启动脚本](https://cloud.tencent.com/document/product/457/32206) · page_id `32206`

## 概述

在节点初始化阶段执行**自定义启动脚本**（UserScript），用于安装 agent、挂载磁盘、调优内核参数、拉取预置镜像等。脚本在节点首次启动时执行，执行日志位于 **`/usr/local/qcloud/tke/userscript`** 目录。

启动脚本可在三个时机设置：

| 时机 | API | 字段路径 | 对存量节点影响 |
|------|-----|---------|:--:|
| 新建节点池 | `CreateClusterNodePool` | `InstanceAdvancedSettings.UserScript` | 无（新池） |
| 新建游离节点 | `CreateClusterInstances` | `InstanceAdvancedSettings.UserScript` | 无（新节点） |
| 添加已有 CVM | `AddExistedInstances` | `InstanceAdvancedSettings.UserScript` | 无（新加入） |
| 修改已有节点池 | `ModifyClusterNodePool` | `UserScript` | 仅对新扩容节点生效，不影响已有节点 |

> **关键字段**：`UserScript` 必须为 **Base64 编码** 的脚本内容。`PreStartUserScript`（启动前脚本）在初始化前执行，目前仅对添加已有节点生效。

## 前置条件

- [环境准备](../../../../环境准备.md)

### 环境检查

```bash
# 1. 检查 tccli 版本
tccli --version
# expected: tccli version >= 1.0.0

# 2. 检查当前凭据和地域
tccli configure list
# expected: secretId、secretKey、region 均已配置

# 3. 检查 CAM 权限 — 需要以下 Action 名
#    tke:DescribeClusters, tke:DescribeClusterNodePools
#    tke:CreateClusterNodePool, tke:ModifyClusterNodePool
#    tke:DescribeClusterNodePoolDetail, tke:DescribeClusterInstances
# 验证：执行 DescribeClusters 确认权限
tccli tke DescribeClusters --region <Region>
# expected: exit 0，返回集群列表

# 4. 检查 base64 工具
echo 'test' | base64
# expected: dGVzdAo=
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
# 5. 查询目标集群
tccli tke DescribeClusters --region <Region>
# expected: 至少 1 个 Running 集群，记录 ClusterId

# 6. 查询已有节点池（修改场景）
tccli tke DescribeClusterNodePools --region <Region> \
    --ClusterId CLUSTER_ID
# expected: 返回节点池列表，记录 NodePoolId

# 7. 查询节点池详情（确认当前 UserScript）
tccli tke DescribeClusterNodePoolDetail --region <Region> \
    --ClusterId CLUSTER_ID \
    --NodePoolId NODEPOOL_ID
# expected: 返回节点池详情，UserScript 字段为当前脚本（明文）
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

### 脚本编写约束

- **幂等性**：脚本需可重复执行不产生副作用，因为节点初始化可能重试。
- **返回码**：脚本 return code 必须为 0（非 0 会导致节点初始化失败）。
- **执行时机**：`UserScript` 在 k8s 组件运行后执行；`PreStartUserScript` 在初始化前执行（仅添加已有节点生效）。
- **脚本路径**：执行日志固定写入 `/usr/local/qcloud/tke/userscript`，不可自定义。
- **编码**：传入 `UserScript` 字段时必须为 **Base64 编码**。

## 控制台与 CLI 参数映射

| 控制台操作 | tccli 命令 | 幂等 |
|-----------|-----------|:--:|
| 新建节点池时设置 | `CreateClusterNodePool` → `InstanceAdvancedSettings.UserScript` | 否 |
| 新建游离节点时设置 | `CreateClusterInstances` → `InstanceAdvancedSettings.UserScript` | 否 |
| 添加已有 CVM 时设置 | `AddExistedInstances` → `InstanceAdvancedSettings.UserScript` | 否 |
| 修改已有节点池 | `ModifyClusterNodePool` → `UserScript` | 否 |
| 启动前脚本 | `InstanceAdvancedSettings.PreStartUserScript`（仅添加已有节点生效） | 否 |
| 查看节点池脚本 | `DescribeClusterNodePoolDetail` → `UserScript` | 是 |

## 关键字段说明

以下说明 `CreateClusterNodePool` / `ModifyClusterNodePool` 中与启动脚本相关的主要参数。完整参数定义见 `tccli tke CreateClusterNodePool help --detail`。

| 字段 | 类型 | 必填 | 取值与约束 | 错误后果 |
|------|------|:--:|------|------|
| `UserScript` | String | 否 | Base64 编码的自定义脚本。在 k8s 组件运行后执行。需保证幂等性和重试逻辑 | 未 Base64 编码 → 脚本乱码或执行失败；return code 非 0 → 节点初始化失败 |
| `PreStartUserScript` | String | 否 | Base64 编码的启动前脚本。在初始化前执行。**仅对添加已有节点生效** | 用于新建节点池/游离节点时不生效（无报错，静默忽略） |
| `Unschedulable` | Integer | 否 | 0（参与调度，默认）或非 0（不参与调度）。配合 UserScript 使用：脚本完成后手动 uncordon | 设为 0 但脚本未完成 → Pod 可能调度到未就绪节点 |
| `ClusterId` | String | 是 | 格式 `cls-xxxxxxxx`。`tccli tke DescribeClusters` 获取 | 不存在 → `ResourceNotFound` |
| `NodePoolId` | String | ModifyClusterNodePool 必填 | 格式 `np-xxxxxxxx`。`tccli tke DescribeClusterNodePools` 获取 | 不存在 → `ResourceNotFound` |
| `InstanceAdvancedSettings` | Object | CreateClusterNodePool 必填 | 含 `UserScript`、`PreStartUserScript`、`Unschedulable`、`Labels`、`Taints` 等子字段 | `UserScript` 须嵌套在此对象内（CreateClusterNodePool）；ModifyClusterNodePool 的 `UserScript` 在顶层 |
| `IgnoreExistedNode` | Boolean | 否 | `true`：修改 Label/Taint 时跳过存量节点；`false`（默认）：滚动更新所有节点 | 默认 `false` 会触发滚动更新，生产环境建议 `true` |

> **字段位置差异**：`CreateClusterNodePool` 中 `UserScript` 嵌套在 `InstanceAdvancedSettings.UserScript`；`ModifyClusterNodePool` 中 `UserScript` 在**顶层**字段。两者都需 Base64 编码。

## 操作步骤

### 步骤 1：准备并 Base64 编码脚本

#### 选择依据

- **脚本内容**：示例安装 `jq` 并写入初始化标记日志。生产脚本按实际需求编写（挂载磁盘、内核调优、预拉镜像等）。
- **幂等性**：脚本开头加 `set -e`（任一命令失败即退出），但需注意 `yum install` 已安装包的返回码处理。
- **日志路径**：TKE 自动将脚本及日志写入 `/usr/local/qcloud/tke/userscript`，同时在脚本内写 `/var/log/` 便于排查。

#### 最小脚本（安装 jq 并写日志）

`init-script-minimal.sh`：

```bash
#!/bin/bash
set -e
yum install -y jq
echo "$(date): node initialized successfully" >> /var/log/tke-init.log
```

Base64 编码：

```bash
base64 -w 0 init-script-minimal.sh
# expected: 输出单行 Base64 字符串，无换行
```

**预期输出**：

```text
IyEvYmluL2Jhc2gKc2V0IC1lCnl1bSBpbnN0YWxsIC15IGpxCmVjaG8gIiQoZGF0ZSk6IG5vZGUgaW5pdGlhbGl6ZWQgc3VjY2Vzc2Z1bGx5IiA+PiAvdmFyL2xvZy90a2UtaW5pdC5sb2cK
```

#### 增强脚本（挂载数据盘 + 内核调优 + 预拉镜像）

`init-script-enhanced.sh`：

```bash
#!/bin/bash
set -e

# 挂载数据盘
mkfs.ext4 /dev/vdb
mount /dev/vdb /data
echo '/dev/vdb /data ext4 defaults 0 2' >> /etc/fstab

# 内核参数调优
sysctl -w net.core.somaxconn=65535
sysctl -w vm.swappiness=10

# 预拉镜像
crictl pull docker.io/library/nginx:1.25

echo "$(date): node initialized with enhanced config" >> /var/log/tke-init.log
```

Base64 编码：

```bash
base64 -w 0 init-script-enhanced.sh
# expected: 输出单行 Base64 字符串，无换行
```

> **macOS 兼容性**：macOS 默认 `base64` 不支持 `-w 0` 参数。替代方案：`base64 init-script-minimal.sh | tr -d '\n'`。

### 步骤 2：新建节点池时设置启动脚本

#### 选择依据

- **参数数量**：`CreateClusterNodePool` 含 `ClusterId`、`Name`、`AutoScalingGroupPara`、`LaunchConfigurePara`、`InstanceAdvancedSettings` 等 ≥4 个参数，使用 `--cli-input-json file://`。
- **UserScript 位置**：嵌套在 `InstanceAdvancedSettings.UserScript` 内。
- **Base64 编码**：`UserScript` 值必须为 Base64 编码后的字符串，不能是明文。

#### 最小创建（含启动脚本的节点池）

`create-nodepool-script-minimal.json`：

```json
{
  "ClusterId": "CLUSTER_ID",
  "Name": "np-with-script",
  "EnableAutoscale": false,
  "AutoScalingGroupPara": "{\"MaxSize\":2,\"MinSize\":0,\"DesiredCapacity\":1,\"VpcId\":\"VPC_ID\",\"SubnetIds\":[\"SUBNET_ID\"]}",
  "LaunchConfigurePara": "{\"InstanceType\":\"S5.MEDIUM2\",\"InstanceChargeType\":\"POSTPAID_BY_HOUR\",\"SystemDisk\":{\"DiskType\":\"CLOUD_PREMIUM\",\"DiskSize\":50},\"SecurityGroupIds\":[\"SECURITY_GROUP_ID\"]}",
  "InstanceAdvancedSettings": {
    "Unschedulable": 0,
    "UserScript": "USERSCRIPT_BASE64"
  },
  "ContainerRuntime": "containerd",
  "RuntimeVersion": "1.6.9"
}
```

```bash
tccli tke CreateClusterNodePool --region <Region> \
    --cli-input-json file://create-nodepool-script-minimal.json
# expected: exit 0, 返回 NodePoolId
```

**预期输出**：

```json
{
    "Response": {
        "NodePoolId": "np-example",
        "RequestId": "a1b2c3d4-e5f6-7890-abcd-ef1234567890"
    }
}
```

#### 增强配置（含 Label + Taint + 启动脚本）

`create-nodepool-script-enhanced.json`：

```json
{
  "ClusterId": "CLUSTER_ID",
  "Name": "gpu-pool-with-script",
  "EnableAutoscale": false,
  "AutoScalingGroupPara": "{\"MaxSize\":2,\"MinSize\":0,\"DesiredCapacity\":1,\"VpcId\":\"VPC_ID\",\"SubnetIds\":[\"SUBNET_ID\"]}",
  "LaunchConfigurePara": "{\"InstanceType\":\"GN10S.2XLARGE40\",\"InstanceChargeType\":\"POSTPAID_BY_HOUR\",\"SystemDisk\":{\"DiskType\":\"CLOUD_SSD\",\"DiskSize\":100},\"SecurityGroupIds\":[\"SECURITY_GROUP_ID\"]}",
  "InstanceAdvancedSettings": {
    "Unschedulable": 0,
    "UserScript": "USERSCRIPT_BASE64"
  },
  "ContainerRuntime": "containerd",
  "RuntimeVersion": "1.6.9",
  "Labels": [
    {"Name": "gpu", "Value": "true"}
  ],
  "Taints": [
    {"Key": "nvidia.com/gpu", "Value": "true", "Effect": "NoSchedule"}
  ],
  "Tags": [{"Key": "env", "Value": "ml"}]
}
```

```bash
tccli tke CreateClusterNodePool --region <Region> \
    --cli-input-json file://create-nodepool-script-enhanced.json
# expected: exit 0, 返回 NodePoolId
```

| 占位符 | 说明 | 约束 | 获取方式 |
|--------|------|------|---------|
| `CLUSTER_ID` | 集群 ID | 格式 `cls-xxxxxxxx` | `tccli tke DescribeClusters` |
| `VPC_ID` | VPC ID | 格式 `vpc-xxxxxxxx` | `tccli vpc DescribeVpcs` |
| `SUBNET_ID` | 子网 ID | 格式 `subnet-xxxxxxxx` | `tccli vpc DescribeSubnets` |
| `SECURITY_GROUP_ID` | 安全组 ID | 格式 `sg-xxxxxxxx` | `tccli vpc DescribeSecurityGroups` |
| `USERSCRIPT_BASE64` | Base64 编码的脚本 | 单行字符串，无换行 | `base64 -w 0 init-script-minimal.sh` |

### 步骤 3：修改已有节点池的启动脚本

#### 选择依据

- **影响范围**：`ModifyClusterNodePool` 修改 `UserScript` 仅对之后扩容出的新节点生效，不影响已有节点。已有节点需手动登录执行或重建。
- **字段位置**：`ModifyClusterNodePool` 的 `UserScript` 在**顶层**字段（非 `InstanceAdvancedSettings` 内）。
- **参数数量**：含 `ClusterId`、`NodePoolId`、`UserScript` 等，使用 `--cli-input-json file://`。

#### 最小修改（仅更新 UserScript）

`modify-nodepool-script-minimal.json`：

```json
{
  "ClusterId": "CLUSTER_ID",
  "NodePoolId": "NODEPOOL_ID",
  "UserScript": "USERSCRIPT_BASE64"
}
```

```bash
tccli tke ModifyClusterNodePool --region <Region> \
    --cli-input-json file://modify-nodepool-script-minimal.json
# expected: exit 0, 返回 RequestId
```

**预期输出**：

```json
{
    "Response": {
        "RequestId": "b2c3d4e5-f6a7-8901-bcde-f12345678901"
    }
}
```

#### 增强修改（更新 UserScript + Label + 跳过存量节点）

`modify-nodepool-script-enhanced.json`：

```json
{
  "ClusterId": "CLUSTER_ID",
  "NodePoolId": "NODEPOOL_ID",
  "UserScript": "USERSCRIPT_BASE64",
  "Labels": [
    {"Name": "env", "Value": "staging"}
  ],
  "IgnoreExistedNode": true
}
```

```bash
tccli tke ModifyClusterNodePool --region <Region> \
    --cli-input-json file://modify-nodepool-script-enhanced.json
# expected: exit 0, 返回 RequestId
```

### 步骤 4：查看节点池当前脚本

```bash
tccli tke DescribeClusterNodePoolDetail --region <Region> \
    --ClusterId CLUSTER_ID \
    --NodePoolId NODEPOOL_ID
# expected: exit 0, UserScript 字段返回明文脚本内容
```

**预期输出**（截取关键字段）：

```json
{
    "Response": {
        "NodePool": {
            "NodePoolId": "np-example",
            "Name": "np-with-script",
            "UserScript": "#!/bin/bash\nset -e\nyum install -y jq\necho \"$(date): node initialized successfully\" >> /var/log/tke-init.log",
            "PreStartUserScript": "",
            "Unschedulable": 0
        },
        "RequestId": "c3d4e5f6-a7b8-9012-cdef-23456789012"
    }
}
```

> **注意**：`DescribeClusterNodePoolDetail` 返回的 `UserScript` 为**明文**（已解码），非 Base64。API 输入需 Base64 编码，输出返回明文。

## 验证

### 控制面（tccli）

```bash
# 1. 确认节点池 UserScript 已设置
tccli tke DescribeClusterNodePoolDetail --region <Region> \
    --ClusterId CLUSTER_ID \
    --NodePoolId NODEPOOL_ID
# expected: NodePool.UserScript 含设置的脚本内容（明文）

# 2. 确认节点池状态正常
tccli tke DescribeClusterNodePools --region <Region> \
    --ClusterId CLUSTER_ID
# expected: 节点池 LifeState 为 normal
```

```json
{
  "NodePool": "<NodePool>",
  "NodePoolId": "<NodePoolId>",
  "Name": "<Name>",
  "ClusterInstanceId": "<ClusterInstanceId>",
  "LifeState": "<LifeState>",
  "LaunchConfigurationId": "<LaunchConfigurationId>"
}
```

### 数据面（需节点登录权限）

```bash
# 3. 查看节点状态（脚本异常可能导致节点初始化失败）
tccli tke DescribeClusterInstances --region <Region> \
    --ClusterId CLUSTER_ID
# expected: 节点 InstanceState 为 running，FailedReason 为空

# 4. 登录节点检查脚本执行日志（需 SSH 或跳板机）
ls /usr/local/qcloud/tke/userscript/
# expected: 存在脚本和日志文件

cat /usr/local/qcloud/tke/userscript/*.log
# expected: 含初始化时间戳和成功标记
```

**预期输出**（日志文件）：

```text
Wed Jun 16 10:30:00 CST 2026: node initialized successfully
```

> **无法 SSH 登录节点**：若新建节点池时 `InternetAccessible.PublicIpAssigned` 为 `false` 且未配置跳板机，将无法直接访问节点。建议在启动脚本中加入写日志到 `/var/log/` 等可观测路径，或通过监控 agent 上报初始化状态。

## 清理

> **警告**：启动脚本本身无需删除资源。如需移除脚本逻辑，通过 `ModifyClusterNodePool` 传入空字符串或新脚本覆盖。通过脚本安装的软件需在节点上手动清理或重建节点。

### 控制面清理（tccli）

```bash
# 1. 清理前状态检查
tccli tke DescribeClusterNodePoolDetail --region <Region> \
    --ClusterId CLUSTER_ID \
    --NodePoolId NODEPOOL_ID
# 确认当前 UserScript 内容

# 2. 清空 UserScript（传入空字符串）
#    modify-nodepool-script-clear.json 内容见下方 JSON 代码块
tccli tke ModifyClusterNodePool --region <Region> \
    --cli-input-json file://modify-nodepool-script-clear.json
# expected: exit 0, 返回 RequestId

# 3. 验证已清空
tccli tke DescribeClusterNodePoolDetail --region <Region> \
    --ClusterId CLUSTER_ID \
    --NodePoolId NODEPOOL_ID
# expected: NodePool.UserScript 为空字符串
```

`modify-nodepool-script-clear.json`：

```json
{
  "ClusterId": "CLUSTER_ID",
  "NodePoolId": "NODEPOOL_ID",
  "UserScript": ""
}
```

### 数据面清理（需节点登录权限）

```bash
# 4. 清理脚本安装的软件（如 jq）——需逐节点登录执行
yum remove -y jq
# expected: jq 已卸载

# 5. 清理脚本写入的日志
rm -f /var/log/tke-init.log
# expected: 日志文件已删除
```

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `CreateClusterNodePool` 返回 `InvalidParameter` | 检查 JSON 中 `UserScript` 值是否为 Base64 编码 | 传入了明文脚本而非 Base64 编码 | 用 `base64 -w 0 script.sh` 重新编码后填入 `UserScript` 字段 |
| `ModifyClusterNodePool` 返回 `InvalidParameter` | 检查 JSON 中 `UserScript` 位置 | `ModifyClusterNodePool` 的 `UserScript` 在**顶层**，非 `InstanceAdvancedSettings` 内 | 将 `UserScript` 从 `InstanceAdvancedSettings` 移到顶层字段 |
| `CreateClusterNodePool` 返回 `ResourceNotFound` | `tccli tke DescribeClusters --region <Region>` 确认 ClusterId | ClusterId 不存在或地域不匹配 | 核对 ClusterId 和 region |
| Base64 编码输出含换行 | `echo '<base64>' \| base64 -d` 验证解码 | 使用了 `base64` 不带 `-w 0`，输出含换行符 | 用 `base64 -w 0 file` 或 `base64 file \| tr -d '\n'` 去除换行 |

### 操作成功但脚本未执行

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| 节点创建后 `FailedReason` 非空 | `tccli tke DescribeClusterInstances --region <Region> --ClusterId CLUSTER_ID` 查看 `FailedReason` | 启动脚本执行失败（return code 非 0） | 确保脚本 return 0；`set -e` 后注意 `yum install` 已安装包的返回码处理；将非关键命令改为 `command \|\| true` |
| 脚本未执行 | 登录节点 `ls /usr/local/qcloud/tke/userscript/` | JSON 字段名错误或 Base64 编码有误 | 用 `echo '<base64>' \| base64 -d` 验证解码内容；确认字段为 `UserScript`（非 `userData` 等控制台概念名） |
| 脚本执行但无效果 | 登录节点 `cat /etc/os-release` 检查镜像 | 脚本依赖的命令在节点镜像中不存在（如 `yum` 在 Ubuntu 不可用） | 根据镜像类型调整脚本：CentOS/TencentOS 用 `yum`，Ubuntu 用 `apt-get` |
| 修改 `ModifyClusterNodePool.UserScript` 后老节点不变 | `tccli tke DescribeClusterInstances --region <Region> --ClusterId CLUSTER_ID` 查看节点创建时间 | `UserScript` 仅对新扩容节点生效，不影响已有节点 | 老节点需手动登录执行相同脚本，或重建节点（先移出再重新加入节点池） |
| 脚本超时导致节点初始化卡住 | 登录节点查看 `/usr/local/qcloud/tke/userscript/` 日志 | 脚本执行时间过长（如大文件下载） | 优化脚本耗时逻辑，将耗时操作改为后台异步：`nohup long_task &` |

## 下一步

- [创建节点池](../../普通节点/创建节点池/tccli%20操作.md) -- page_id `43735`（新建含启动脚本的节点池）
- [设置节点 Label](../设置节点%20Label/tccli%20操作.md) -- page_id `32768`（为节点打标签）
- [调整节点池](../../普通节点/调整节点池/tccli%20操作.md)（修改节点池全局配置）
- [自定义 cgroup driver 配置](../自定义%20cgroup%20driver%20配置/tccli%20操作.md) -- page_id `132099`（运行时配置）
- [TKE 节点初始化流程 FAQ](../../普通节点/节点初始化流程%20FAQ/tccli%20操作.md)（初始化问题排查）

## 控制台替代

[容器服务控制台 - 节点管理](https://console.cloud.tencent.com/tke2/cluster?rid=1)
