# 使用 Terraform 管理 TKE 集群和节点池（tccli）

> 对照官方：[使用 Terraform 管理 TKE 集群和节点池](https://cloud.tencent.com/document/product/457/83830) · page_id `83830`

## 概述

使用 Terraform（IaC）管理 TKE 集群和节点池的生命周期：创建 VPC/子网 → 创建托管集群（GlobalRouter / VPC-CNI）→ 创建节点池 → 销毁资源。相比控制台操作，Terraform 提供声明式、可版本管理、可审计的基础设施管理能力。

> **注意**：本页面向 Terraform 用户，提供 TKE Provider 声明和完整 `.tf` 文件。tccli 仅用于验证（`DescribeClusters` / `DescribeClusterNodePools`），主体操作由 `terraform` CLI 完成。

## 前置条件

- 安装 [Terraform](https://developer.hashicorp.com/terraform/downloads) ≥ 1.0。
- 获取腾讯云 API 密钥（[SecretId/SecretKey](https://console.cloud.tencent.com/cam/capi)）。
- 设置环境变量 `TENCENTCLOUD_SECRET_ID` 和 `TENCENTCLOUD_SECRET_KEY`。
- 如首次使用 TKE，需完成 [服务授权](https://cloud.tencent.com/document/product/457/43416)。

```bash
terraform --version
```

```
Terraform v1.9.0
```

## 控制台与 CLI 参数映射

| 控制台操作 | CLI | 幂等 |
|-----------|-----|------|
| 创建 VPC/子网 | `terraform apply`（tencentcloud_vpc / tencentcloud_subnet） | 否（名称冲突） |
| 创建托管集群 | `terraform apply`（tencentcloud_kubernetes_cluster） | 否 |
| 创建节点池 | `terraform apply`（tencentcloud_kubernetes_node_pool） | 否 |
| 验证集群 | `tccli tke DescribeClusters --ClusterIds '["<id>"]'` | 是 |
| 删除所有资源 | `terraform destroy` | 否（不可逆） |

## 操作步骤

### 1. 配置文件结构

```
tke-terraform/
├── main.tf          # VPC + 子网 + 集群
├── nodepool.tf      # 节点池
├── cam.tf           # CAM 角色授权（首次使用）
├── provider.tf      # Provider 配置
```

### 2. Provider 和认证

```hcl
# provider.tf
terraform {
  required_providers {
    tencentcloud = {
      source = "tencentcloudstack/tencentcloud"
    }
  }
}
```

```bash
export TENCENTCLOUD_SECRET_ID="<SecretId>"
export TENCENTCLOUD_SECRET_KEY="<SecretKey>"
```

### 3. 创建集群（main.tf）

```hcl
locals {
  region         = "ap-guangzhou"
  zone1          = "ap-guangzhou-6"
  vpc_name       = "tke-tf-demo"
  vpc_cidr_block = "10.0.0.0/16"
  subnet1_name   = "tke-tf-demo-sub1"
  subnet1_cidr   = "10.0.1.0/24"
  cluster_name   = "tke-tf-demo-cluster"
  network_type   = "GR"
  cluster_cidr   = "172.26.0.0/20"
  cluster_version = "1.28.3"
}

provider "tencentcloud" {
  region = local.region
}

# VPC
resource "tencentcloud_vpc" "vpc_example" {
  name       = local.vpc_name
  cidr_block = local.vpc_cidr_block
}

# 子网
resource "tencentcloud_subnet" "subnet_example" {
  availability_zone = local.zone1
  cidr_block        = local.subnet1_cidr
  name              = local.subnet1_name
  vpc_id            = tencentcloud_vpc.vpc_example.id
}

# 托管集群（GlobalRouter）
resource "tencentcloud_kubernetes_cluster" "managed_cluster_example" {
  vpc_id         = tencentcloud_vpc.vpc_example.id
  cluster_name   = local.cluster_name
  network_type   = local.network_type
  cluster_cidr   = local.cluster_cidr
  cluster_version = local.cluster_version
}
```

```bash
terraform init
terraform plan
terraform apply -auto-approve
```

```
Apply complete! Resources: 3 added, 0 changed, 0 destroyed.
```

### 4. 创建节点池（nodepool.tf）

```hcl
locals {
  node_pool_name   = "tke-tf-demo-node-pool"
  max_node_size    = 6
  min_node_size    = 1
  cvm_instance_type = "S5.MEDIUM2"
  cvm_pass_word    = "<Password8-16chars>"
  security_group_ids = ["sg-xxxxxxxx"]
}

resource "tencentcloud_kubernetes_node_pool" "example_node_pool" {
  cluster_id          = tencentcloud_kubernetes_cluster.managed_cluster_example.id
  delete_keep_instance = false
  max_size            = local.max_node_size
  min_size            = local.min_node_size
  name                = local.node_pool_name
  vpc_id              = tencentcloud_vpc.vpc_example.id
  subnet_ids          = [tencentcloud_subnet.subnet_example.id]
  auto_scaling_config {
    instance_type     = local.cvm_instance_type
    password          = local.cvm_pass_word
    security_group_ids = local.security_group_ids
  }
}
```

```bash
terraform apply -auto-approve
```

```
Apply complete! Resources: 1 added, 0 changed, 0 destroyed.
```

### 5. 验证

```bash
tccli tke DescribeClusters --region ap-guangzhou --output json \
  --filter "Clusters[?ClusterName=='tke-tf-demo-cluster']|[0].{ClusterId:ClusterId,ClusterStatus:ClusterStatus}"

tccli tke DescribeClusterNodePools --ClusterId <ClusterId> --region ap-guangzhou --output json \
  --filter "NodePoolSet[0].{Name:Name,NodeCountSummary:NodeCountSummary}"
```

```
{
  "ClusterId": "cls-xxxxxxxx",
  "ClusterStatus": "Running"
}
{
  "Name": "tke-tf-demo-node-pool",
  "NodeCountSummary": { "Total": 1 }
}
```

### 6. VPC-CNI 模式集群（可选）

```hcl
resource "tencentcloud_kubernetes_cluster" "managed_cluster_example" {
  vpc_id         = tencentcloud_vpc.vpc_example.id
  cluster_name   = local.cluster_name
  network_type   = "VPC-CNI"
  eni_subnet_ids = [tencentcloud_subnet.subnet_example.id]
  service_cidr   = "172.16.0.0/24"
  cluster_version = local.cluster_version
}
```

### 7. CAM 角色授权（首次使用）

```hcl
# cam.tf — 首次使用 TKE 时创建服务角色
resource "tencentcloud_cam_role" "TKE_QCSRole" {
  name        = "TKE_QCSRole"
  document    = <<EOF
{
  "statement": [{
    "action": "name/sts:AssumeRole",
    "effect": "allow",
    "principal": { "service": "ccs.qcloud.com" }
  }],
  "version": "2.0"
}
EOF
  description = "TKE 服务角色"
}

data "tencentcloud_cam_policies" "qca" {
  name = "QcloudAccessForTKERole"
}

resource "tencentcloud_cam_role_policy_attachment" "QCS_QCA" {
  role_id   = tencentcloud_cam_role.TKE_QCSRole.id
  policy_id = data.tencentcloud_cam_policies.qca.policy_list[0].policy_id
}
```

## 验证

```bash
tccli tke DescribeClusters --region ap-guangzhou --output json \
  --filter "length(Clusters[?ClusterName=='tke-tf-demo-cluster']) > `0`"
```

```
true
```

## 排障

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `kubectl` 连接超时/拒绝 | `tccli tke DescribeClusters` 返回集群 Running 但 kubectl 不可达 | CAM 策略 240463971 拒绝公网访问 APIServer 端点 | 通过 VPN/IOA 接入内网后执行 kubectl 命令 |
| `terraform init` 下载 Provider 慢 | `terraform init` 日志查看下载耗时 | 网络访问 HashiCorp 官方源慢 | 使用镜像源或设置 `TF_PLUGIN_CACHE_DIR` |
| `apply` 报 `AuthFailure.UnauthorizedOperation` | `terraform plan` 查看具体报错 | SecretId/SecretKey 无效或缺少 TKE 权限 | 确认密钥正确且已授予 TKE 相关权限 |
| `apply` 报 `role not exist` | `terraform plan` 查看具体报错 | 首次使用 TKE 未完成服务授权 | 添加 `cam.tf` 或先通过控制台完成服务授权 |
| `Error: InvalidNetworkType` | `terraform plan` 查看具体报错 | `network_type` 值非 API 枚举值 | 确认 `network_type` 为 `GR` 或 `VPC-CNI`（非 `GlobalRouter`） |

## 清理

```bash
terraform destroy -auto-approve
```

```
Destroy complete! Resources: 4 destroyed.
```

> **注意**：如果节点池 `delete_keep_instance = true`，`destroy` 不会删除 CVM 实例，需手动释放。

## 下一步

- [在 containerd 集群中使用 Docker 做镜像构建服务](../../DevOps/在 containerd 集群中使用 Docker 构建镜像/tccli 操作.md)（page_id `50868`）
- [快速创建一个标准集群](../../../快速入门/快速创建一个标准集群/tccli 操作.md)（page_id `32189`）

## 控制台替代

控制台 **集群 → 新建** 依次完成网络、机型、组件配置后创建；**节点池 → 新建** 加入节点。
