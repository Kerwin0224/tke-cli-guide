# 通过 Terraform 安装 Nginx 插件和实例（tccli）

> 对照官方：[通过 Terraform 安装 Nginx 插件和实例](https://cloud.tencent.com/document/product/457/90887) · page_id `90887`

## 概述

> **注意：** NginxIngress 扩展组件已停止更新，详情见 [NginxIngress 扩展组件停止更新公告](https://cloud.tencent.com/document/product/457/108517)。

使用 Terraform 声明式安装 TKE NginxIngress 插件（Addon）和 Nginx 实例，实现基础设施即代码（IaC）管理。

## 前置条件

- [环境准备](../../../../../环境准备.md)
- 已 [连接集群](../../../../../集群配置/集群管理/连接集群/tccli 操作.md)，kubectl 可正常访问 APIServer
- 已安装 Terraform（>= v1.6.6）
- 已配置腾讯云 Provider 凭证

## 控制台与 CLI 参数映射

| Console 操作 | Terraform 等价操作 | 幂等 | 说明 |
|---|---|---|---|------|
| 安装 Nginx 插件 | `tencentcloud_kubernetes_addon_attachment` | 是(apply) | 集群插件安装 |
| 创建 Nginx 实例 | `kubernetes_manifest` (kind: NginxIngress) | 是(apply) | CRD 资源 |
| 配置 HPA | `spec.workLoad.hpa` | 是(apply) | 弹性伸缩 |
| 配置 LB | `spec.service.annotation` | 是(apply) | LB 类型与参数 |

## 操作步骤

### 步骤 1：安装 Terraform

```bash
wget https://releases.hashicorp.com/terraform/1.6.6/terraform_1.6.6_linux_amd64.zip
unzip terraform_1.6.6_linux_amd64.zip
sudo mv terraform /usr/local/bin/
```

### 步骤 2：安装 Nginx Addon 插件

```hcl
# provider.tf
terraform {
  required_providers {
    tencentcloud = {
      source  = "tencentcloudstack/tencentcloud"
      version = "1.81.61"
    }
  }
}

provider "tencentcloud" {
  secret_id  = "<secret_id>"
  secret_key = "<secret_key>"
  region     = "ap-shanghai"
}

resource "tencentcloud_kubernetes_addon_attachment" "addon_ingressnginx" {
  cluster_id   = "cls-xxxxxxxx"
  name         = "ingressnginx"
  request_body = "{\"kind\":\"App\",\"spec\":{\"chart\":{\"chartName\":\"ingressnginx\",\"chartVersion\":\"1.4.1\"}}}"
}
```

### 步骤 3：声明式安装 Nginx 实例

```hcl
provider "kubernetes" {
  config_path = "~/.kube/config"
}

resource "kubernetes_manifest" "nginxingress_demo" {
  manifest = {
    "apiVersion" = "cloud.tencent.com/v1alpha1"
    "kind"       = "NginxIngress"
    "metadata" = {
      "name" = "demo"
    }
    "spec" = {
      "ingressClass" = "demo"
      "service" = {
        "annotation" = {
          "service.kubernetes.io/service.extensiveParameters" = "{\"InternetAccessible\":{\"InternetChargeType\":\"TRAFFIC_POSTPAID_BY_HOUR\",\"InternetMaxBandwidthOut\":10}}"
        }
        "type" = "LoadBalancer"
      }
      "workLoad" = {
        "hpa" = {
          "enable"      = true
          "maxReplicas" = 2
          "metrics" = [
            {
              "pods" = {
                "metricName"         = "k8s_pod_rate_cpu_core_used_limit"
                "targetAverageValue" = "80"
              }
              "type" = "Pods"
            },
          ]
          "minReplicas" = 1
        }
        "template" = {
          "affinity" = {}
          "container" = {
            "image" = "ccr.ccs.tencentyun.com/tkeimages/nginx-ingress-controller:v1.6.4"
            "resources" = {
              "limits" = {
                "cpu"    = "0.5"
                "memory" = "1024Mi"
              }
              "requests" = {
                "cpu"    = "0.25"
                "memory" = "256Mi"
              }
            }
          }
        }
        "type" = "deployment"
      }
    }
  }
}
```

```bash
terraform init
terraform plan
terraform apply -auto-approve
```

## 验证

### 数据面（kubectl）

```bash
kubectl get nginxingress -A
kubectl get pods -n kube-system | grep nginx-ingress
```

```text
NAME  STATUS  AGE
...
```

```bash
terraform state list
```

## 清理

### 数据面（kubectl）

```bash
terraform destroy -auto-approve
```

## 排障

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `Unable to connect to the server` | `kubectl cluster-info` | 公网端点被 CAM 策略 strategyId:240463971 以 `tke:clusterExtranetEndpoint=true` 条件拒绝 | 通过 IOA/VPN 连接内网，或使用同 VPC CVM 执行 kubectl |
| Provider 认证失败 | `secret_id` | `secret_id` 和 `secret_key` 正确 | 确认 `secret_id` 和 `secret_key` 正确 |
| 插件安装失败 | `kubectl describe <resource>` | 集群 ID 和 chart 版本兼容 | 确认集群 ID 和 chart 版本兼容 |
| HPA 不生效 | `spec.workLoad.hpa.enable` | `spec.workLoad.hpa.enable` 是否为 true | 检查 `spec.workLoad.hpa.enable` 是否为 true |
| 镜像拉取失败 | `kubectl describe pod` | 镜像版本与 K8s 版本兼容 | 确认镜像版本与 K8s 版本兼容 |

## 下一步

- [安装 NginxIngress 实例](../安装%20NginxIngress%20实例/tccli 操作.md)
- [使用 NginxIngress 对象接入集群外部流量](../使用%20NginxIngress%20对象接入集群外部流量/tccli 操作.md)
- [NginxIngress 日志配置](../NginxIngress%20日志配置/tccli 操作.md)

## 控制台替代

在控制台 **组件管理** 中安装 NginxIngress 组件，再在 **服务与路由 > NginxIngress** 中创建实例。
