# 使用 Ansible 批量操作 TKE 节点（tccli）

> 对照官方：[使用 Ansible 批量操作 TKE 节点](https://cloud.tencent.com/document/product/457/48973) · page_id `48973`

## 概述

通过 Ansible 动态清单（Dynamic Inventory）从 TKE API 获取节点列表，批量执行运维任务（系统更新、安全加固等）。

## 前置条件

- 已安装 Ansible（`pip install ansible`）

## 控制台与 CLI 参数映射

| 控制台操作 | CLI | 幂等 |
|-----------|-----|------|
| 获取节点列表 | `tccli tke DescribeClusterInstances` | 是 |
| Ansible Ping | `ansible all -i inventory.sh -m ping` | 是 |
| 批量执行 | `ansible-playbook -i inventory.sh playbook.yaml` | 否 |

## 操作步骤

### 1. 创建动态清单脚本

```bash
#!/bin/bash
# inventory.sh
tccli tke DescribeClusterInstances --region ap-guangzhou --ClusterId <ClusterId> | \
  jq -r '.InstanceSet[] | "\(.InstanceId) ansible_host=\(.LANIP) ansible_user=root"'
```

```json
{
  "TotalCount": 0,
  "InstanceSet": [],
  "InstanceId": "<InstanceId>",
  "InstanceRole": "<InstanceRole>",
  "FailedReason": "<FailedReason>",
  "InstanceState": "<InstanceState>",
  "DrainStatus": "<DrainStatus>",
  "InstanceAdvancedSettings": "<InstanceAdvancedSettings>"
}
```

### 2. Ping 验证连通性

```bash
ansible all -i inventory.sh -m ping
```

```output
ins-example | SUCCESS => {"changed": false, "ping": "pong"}
```

### 3. 执行批量任务

```yaml
# playbook.yaml
- hosts: all
  tasks:
  - name: Update system
    apt: update_cache=yes upgrade=dist
```

```bash
ansible-playbook -i inventory.sh playbook.yaml
```

## 验证

```bash
ansible all -i inventory.sh -m shell -a "uptime"
```

## 排障

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `inventory.sh` 输出为空 | `tccli tke DescribeClusters --region ap-guangzhou --ClusterId cls-xxxxxxxx --filter "Clusters[].ClusterStatus"` | 集群未 Running 或 CAM 权限不足导致 `DescribeClusterInstances` 无返回 | 确认集群 Running；CAM 授予 `tke:DescribeClusterInstances` |
| `ansible -m ping` 返回 `UNREACHABLE` | `ansible all -i inventory.sh -m ping -vvv` | 节点 SSH 未开放或 `ansible_user` 错误 | 确认 CVM 安全组放行 22 端口；核对 `ansible_user` 与密钥 |
| kubectl 相关任务报 `Unable to connect to the server` | `tccli tke DescribeClusters --region ap-guangzhou --ClusterId cls-xxxxxxxx --filter "Clusters[].ClusterStatus"` | CAM 拒绝公网端点（strategyId 240463971），数据面不可达 | 接入 VPN/IOA 后在节点上执行 kubectl 任务 |

## 清理

无需清理。

## 下一步

- [TKE 集群中节点移出再移入操作指引](../TKE%20集群中节点移出再移入操作指引/tccli%20操作.md)
- [使用 kubecm 管理多集群 kubeconfig](../使用%20kubecm%20管理多集群%20kubeconfig/tccli%20操作.md)

## 控制台替代

无控制台界面。
