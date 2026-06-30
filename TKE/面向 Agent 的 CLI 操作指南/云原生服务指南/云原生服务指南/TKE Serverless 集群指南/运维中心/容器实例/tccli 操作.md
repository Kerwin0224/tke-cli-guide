# 容器实例（tccli）

> 对照官方：[容器实例](https://cloud.tencent.com/document/product/457/39820) · page_id `39820`

## 概述

弹性容器实例（EKS Container Instance，EKS CI）是 TKE Serverless 的核心运行单元，提供无需管理底层节点的容器运行环境。通过 `tccli tke` 系列 API 实现容器实例的全生命周期管理：创建、查询、删除和日志查看。

## 前置条件

- [环境准备](../../../环境准备.md)
- 已配置 `tccli`，可用 `tccli tke help` 验证
- 已有 VPC（`vpc-example`）、子网（`subnet-example`）、安全组（`sg-example`）
- 安全组已放通所需端口（至少放通容器服务端口）
- 容器镜像可正常拉取（公网镜像或已配置 TCR 镜像仓库访问）

> **注意：** kubectl 不可达（公网端点 CAM 拒绝 strategyId:240463971）。以下所有操作均通过 tccli API 完成。

## 控制台与 CLI 参数映射

| 控制台操作 | CLI | 幂等 |
|-----------|-----|------|
| 创建容器实例 | `tccli tke CreateEKSContainerInstances --cli-input-json file://eksci-config.json` | 否 |
| 查询容器实例列表 | `tccli tke DescribeEKSContainerInstances --EksCiIds '["eksci-example"]'` | 是 |
| 按条件过滤查询 | `tccli tke DescribeEKSContainerInstances --Filters '[{"Name":"Status","Values":["Running"]}]'` | 是 |
| 查看实例事件 | `tccli tke DescribeEKSContainerInstanceEvent --EksCiId "eksci-example"` | 是 |
| 查看容器日志 | `tccli tke DescribeEKSContainerInstanceLog --EksCiId "eksci-example" --ContainerName "<ContainerName>"` | 是 |
| 删除容器实例 | `tccli tke DeleteEKSContainerInstances --EksCiIds '["eksci-example"]'` | 否 |

## 操作步骤

### 1. 创建容器实例

创建输入 JSON 文件 `eksci-config.json`：

```json
{
    "VpcId": "vpc-example",
    "SubnetId": "subnet-example",
    "SecurityGroupIds": ["sg-example"],
    "Containers": [
        {
            "Image": "nginx:latest",
            "Name": "nginx-container",
            "Cpu": 0.5,
            "Memory": 1.0
        }
    ],
    "InstanceName": "eksci-example",
    "RestartPolicy": "Never",
    "Cpu": 0.5,
    "Memory": 1.0,
    "EksCiName": "eksci-example"
}
```

参数说明：

| 参数 | 必填 | 说明 |
|------|------|------|
| `VpcId` | 是 | VPC ID，如 `vpc-example` |
| `SubnetId` | 是 | 子网 ID，需与 VPC 在同一可用区 |
| `SecurityGroupIds` | 是 | 安全组 ID 列表 |
| `Containers[].Image` | 是 | 容器镜像地址 |
| `Containers[].Name` | 是 | 容器名称 |
| `Containers[].Cpu` | 否 | 容器 CPU（核），不指定则继承实例级 |
| `Containers[].Memory` | 否 | 容器内存（GB），不指定则继承实例级 |
| `EksCiName` | 是 | 实例名称 |
| `Cpu` | 是 | 实例级 CPU（核） |
| `Memory` | 是 | 实例级内存（GB） |
| `RestartPolicy` | 否 | 重启策略：`Never`/`OnFailure`/`Always`，默认 `Always` |

```bash
tccli tke CreateEKSContainerInstances \
    --region ap-guangzhou \
    --cli-input-json file://eksci-config.json
```

```output
{
    "EksCiIds": ["eksci-example"],
    "RequestId": "a1b2c3d4-e5f6-7890-abcd-ef1234567890"
}
```

### 2. 轮询实例状态

创建为异步操作。使用 `DescribeEKSContainerInstances` 轮询直至实例状态变为 `Succeeded`（终端成功状态）。

```bash
# 轮询循环：每 10 秒查询一次状态，最多等待 120 秒
for i in $(seq 1 12); do
    STATUS=$(tccli tke DescribeEKSContainerInstances \
        --EksCiIds '["eksci-example"]' \
        --region ap-guangzhou \
        | jq -r '.EksCis[0].Status')
    echo "[$(date +%H:%M:%S)] Status: ${STATUS}"
    if [ "${STATUS}" = "Succeeded" ]; then
        echo "Instance successfully created."
        break
    elif [ "${STATUS}" = "Failed" ]; then
        echo "Instance creation failed. Check events."
        break
    fi
    sleep 10
done
```

```json
{
  "TotalCount": 0,
  "EksCis": [],
  "AutoCreatedEipId": "<AutoCreatedEipId>",
  "CamRoleName": "<CamRoleName>",
  "Contain

```json
{
  "TotalCount": 0,
  "EksCis": [],
  "AutoCreatedEipId": "<AutoCreatedEipId>",
  "CamRoleName": "<CamRoleName>",
  "Containers": [],
  "Image": "<Image>",
  "Name": "<Name>",
  "Args": []
}
```ers": [],
  "Image": "<Image>",
  "Name": "<Name>",
  "Args": []
}
```

### 3. 查询容器实例详情

```bash
tccli tke DescribeEKSContainerInstances \
    --EksCiIds '["eksci-example"]' \
    --region ap-guangzhou
```

```output
{
    "TotalCount": 1,
    "EksCis": [
        {
            "EksCiId": "eksci-example",
            "EksCiName": "eksci-example",
            "Status": "Succeeded",
            "Cpu": 0.5,
            "Memory": 1.0,
            "VpcId": "vpc-example",
            "SubnetId": "subnet-example",
            "SecurityGroupIds": ["sg-example"],
            "PrivateIp": "10.0.1.15",
            "RestartPolicy": "Never",
            "Containers": [
                {
                    "Name": "nginx-container",
                    "Image": "nginx:latest",
                    "Cpu": 0.5,
                    "Memory": 1.0,
                    "CurrentState": {
                        "State": "running",
                        "StartTime": "2024-07-17T10:05:30Z",
                        "FinishTime": "",
                        "ExitCode": 0
                    },
                    "RestartCount": 0
                }
            ],
            "CreationTime": "2024-07-17T10:05:15Z",
            "CamRoleName": "",
            "Events": []
        }
    ],
    "RequestId": "b2c3d4e5-f6a7-8901-bcde-f12345678901"
}
```

### 4. 按条件过滤查询

查询所有 `Running` 状态的实例：

```bash
tccli tke DescribeEKSContainerInstances \
    --Filters '[{"Name":"Status","Values":["Succeeded"]}]' \
    --Limit 20 \
    --region ap-guangzhou
```

```output
{
    "TotalCount": 1,
    "EksCis": [
        {
            "EksCiId": "eksci-example",
            "EksCiName": "eksci-example",
            "Status": "Succeeded",
            "Cpu": 0.5,
            "Memory": 1.0,
            "PrivateIp": "10.0.1.15",
            "VpcId": "vpc-example",
            "CreationTime": "2024-07-17T10:05:15Z",
            "Containers": [
                {
                    "Name": "nginx-container",
                    "Image": "nginx:latest",
                    "CurrentState": {"State": "running", "StartTime": "2024-07-17T10:05:30Z"}
                }
            ]
        }
    ],
    "RequestId": "c3d4e5f6-a7b8-9012-cdef-123456789012"
}
```

支持过滤的 Name：

| Name | 说明 | 示例 Values |
|------|------|------------|
| `Status` | 实例状态 | `["Succeeded"]`, `["Failed"]`, `["Pending"]` |
| `EksCiName` | 实例名称 | `["eksci-example"]` |
| `VpcId` | VPC ID | `["vpc-example"]` |

### 5. 查看容器实例事件

```bash
tccli tke DescribeEKSContainerInstanceEvent \
    --EksCiId "eksci-example" \
    --region ap-guangzhou
```

```output
{
    "TotalCount": 3,
    "Events": [
        {
            "EksCiId": "eksci-example",
            "Message": "Successfully pulled image \"nginx:latest\"",
            "Reason": "Pulled",
            "Type": "Normal",
            "FirstTimestamp": "2024-07-17T10:05:20Z",
            "LastTimestamp": "2024-07-17T10:05:20Z",
            "Count": 1
        },
        {
            "EksCiId": "eksci-example",
            "Message": "Created container nginx-container",
            "Reason": "Created",
            "Type": "Normal",
            "FirstTimestamp": "2024-07-17T10:05:25Z",
            "LastTimestamp": "2024-07-17T10:05:25Z",
            "Count": 1
        },
        {
            "EksCiId": "eksci-example",
            "Message": "Started container nginx-container",
            "Reason": "Started",
            "Type": "Normal",
            "FirstTimestamp": "2024-07-17T10:05:30Z",
            "LastTimestamp": "2024-07-17T10:05:30Z",
            "Count": 1
        }
    ],
    "RequestId": "d4e5f6a7-b8c9-0123-defa-123456789013"
}
```

### 6. 查看容器日志

```bash
tccli tke DescribeEKSContainerInstanceLog \
    --EksCiId "eksci-example" \
    --ContainerName "nginx-container" \
    --TailLines 50 \
    --region ap-guangzhou
```

```output
{
    "ContainerName": "nginx-container",
    "LogContent": "/docker-entrypoint.sh: /docker-entrypoint.d/ is not empty, will attempt to perform configuration\n/docker-entrypoint.sh: Looking for shell scripts in /docker-entrypoint.d/\n/docker-entrypoint.sh: Launching /docker-entrypoint.d/10-listen-on-ipv6-by-default.sh\n10-listen-on-ipv6-by-default.sh: info: Getting the checksum of /etc/nginx/conf.d/default.conf\n10-listen-on-ipv6-by-default.sh: info: Enabled listen on IPv6 in /etc/nginx/conf.d/default.conf\n/docker-entrypoint.sh: Sourcing /docker-entrypoint.d/15-local-resolvers.envsh\n/docker-entrypoint.sh: Launching /docker-entrypoint.d/20-envsubst-on-templates.sh\n/docker-entrypoint.sh: Launching /docker-entrypoint.d/30-tune-worker-processes.sh\n/docker-entrypoint.sh: Configuration complete; ready for start up\n2024/07/17 10:05:31 [notice] 1#1: using the \"epoll\" event method\n2024/07/17 10:05:31 [notice] 1#1: nginx/1.25.2\n2024/07/17 10:05:31 [notice] 1#1: built by gcc 12.2.0 (Debian 12.2.0-14)\n2024/07/17 10:05:31 [notice] 1#1: OS: Linux 5.4.119-1-tlinux4-0010\n2024/07/17 10:05:31 [notice] 1#1: getrlimit(RLIMIT_NOFILE): 1048576:1048576\n2024/07/17 10:05:31 [notice] 1#1: start worker processes\n2024/07/17 10:05:31 [notice] 1#1: start worker process 29\n2024/07/17 10:05:31 [notice] 1#1: start worker process 30\n",
    "RequestId": "e5f6a7b8-c9d0-1234-efab-123456789014"
}
```

参数说明：

| 参数 | 必填 | 说明 |
|------|------|------|
| `EksCiId` | 是 | 容器实例 ID |
| `ContainerName` | 是 | 容器名称，对应创建时的 `Containers[].Name` |
| `TailLines` | 否 | 日志尾部行数，默认 200 |
| `Previous` | 否 | 是否查看上一个容器实例的日志（重启后） |

### 7. 删除容器实例

```bash
tccli tke DeleteEKSContainerInstances \
    --EksCiIds '["eksci-example"]' \
    --region ap-guangzhou
```

```output
{
    "RequestId": "f6a7b8c9-d0e1-2345-fabc-123456789015"
}
```

## 验证

### Control plane (tccli)

**验证实例不存在（删除后）或状态正确：**

```bash
tccli tke DescribeEKSContainerInstances \
    --EksCiIds '["eksci-example"]' \
    --region ap-guangzhou \
    | jq '{EksCiId: .EksCis[0].EksCiId, Status: .EksCis[0].Status, PrivateIp: .EksCis[0].PrivateIp, Containers: [.EksCis[0].Containers[] | {Name, Image, State: .CurrentState.State}]}'
```

预期输出（实例运行中）：

```output
{
    "EksCiId": "eksci-example",
    "Status": "Succeeded",
    "PrivateIp": "10.0.1.15",
    "Containers": [
        {
            "Name": "nginx-container",
            "Image": "nginx:latest",
            "State": "running"
        }
    ]
}
```

## 清理

### Control plane (tccli)

```bash
# 1. 确认实例 ID
tccli tke DescribeEKSContainerInstances \
    --EksCiIds '["eksci-example"]' \
    --region ap-guangzhou \
    | jq '.EksCis[0].EksCiId'

# 2. 删除实例
tccli tke DeleteEKSContainerInstances \
    --EksCiIds '["eksci-example"]' \
    --region ap-guangzhou

# 3. 验证已删除
tccli tke DescribeEKSContainerInstances \
    --EksCiIds '["eksci-example"]' \
    --region ap-guangzhou \
    | jq '.TotalCount'
# 预期返回 0
```

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `CreateEKSContainerInstances` 报 `InvalidParameterValue.VpcId` | `tccli vpc DescribeVpcs --VpcIds '["<VpcId>"]'` 确认 VPC 存在 | VpcId 与 SubnetId 不在同一可用区，或 VPC 不存在 | 确认 VPC 和子网在同一地域/可用区；用 `tccli vpc DescribeSubnets --SubnetIds '["<SubnetId>"]'` 查看子网所属可用区 |
| `CreateEKSContainerInstances` 报 `InvalidParameterValue.SecurityGroupId` | `tccli vpc DescribeSecurityGroups --SecurityGroupIds '["<SecurityGroupId>"]'` | 安全组不存在或未与 VPC 关联 | 确认安全组存在于目标 VPC 中 |
| `DeleteEKSContainerInstances` 报 `ResourceNotFound` | `tccli tke DescribeEKSContainerInstances --EksCiIds '["<EksCiId>"]'` 确认实例 | 实例已被删除或 `EksCiIds` 参数中 ID 不正确 | 重新查询实例 ID 后重试 |

### 创建成功但状态异常

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| 实例 Status 长时间 `Pending` | `tccli tke DescribeEKSContainerInstances --EksCiIds '["<EksCiId>"]'` 检查状态 | 子网 IP 不足、安全组规则阻止流量、或镜像拉取异常 | 检查子网可用 IP；确认安全组出规则放通镜像仓库；验证镜像名称和 tag 正确 |
| 实例 Status 变为 `Failed` | `tccli tke DescribeEKSContainerInstanceEvent --EksCiId <EksCiId>` 查看事件 | 常见：镜像拉取失败、资源不足、启动命令异常 | 根据事件原因修复（换镜像、增资源、修正命令） |
| `DescribeEKSContainerInstanceLog` 返回空 | `tccli tke DescribeEKSContainerInstances --EksCiIds '["<EksCiId>"]'` 检查 Containers[].CurrentState | 容器未启动或 `ContainerName` 不匹配，日志存在轻微延迟 | 确认容器已启动且 `ContainerName` 与创建时一致；等待 30 秒后重试 |
| 容器反复重启（`RestartCount` 递增） | `tccli tke DescribeEKSContainerInstanceEvent --EksCiId <EksCiId>` 查看事件和日志 | 启动命令错误、健康检查配置不合理、或应用崩溃 | 检查 `DescribeEKSContainerInstanceLog` 输出和事件消息，修正启动命令或健康检查 |

## 下一步

- [查看监控](../监控和告警/查看监控/tccli%20操作.md) — 查看容器实例的 CPU/内存/网络监控指标
- [日志采集](../日志采集/开通日志采集/tccli%20操作.md) — 将容器日志采集到 CLS 持久化存储
- [集群事件](../事件管理/集群事件/tccli%20操作.md) — 查看集群级别事件日志

## 控制台替代

容器服务控制台 → 弹性容器 → 容器实例 → 新建 / 实例列表（查看/删除/日志/事件）。
