# Serverless 集群自定义 DNS 服务（tccli）

> 对照官方：[Serverless 集群自定义 DNS 服务](https://cloud.tencent.com/document/product/457/63735) · page_id `63735`

## 概述

弹性容器实例支持通过 `DnsConfig` 参数自定义 DNS 解析，指定 nameservers、searches 和 options。

## 前置条件

- [环境准备](../../../../环境准备.md)

## 控制台与 CLI 参数映射

| 控制台操作 | CLI | 幂等 |
|-----------|-----|------|
| 设置自定义 DNS | `DnsConfig` 参数 {Nameservers, Searches, Options} | 否 |

## 操作步骤

```json
{
    "EksCiName": "custom-dns-test",
    "VpcId": "vpc-example",
    "SubnetId": "subnet-example",
    "SecurityGroupIds": ["sg-example"],
    "Cpu": 0.25,
    "Memory": 0.5,
    "DnsConfig": {
        "Nameservers": ["8.8.8.8", "114.114.114.114"],
        "Searches": ["mycompany.local"],
        "Options": [
            {"Name": "ndots", "Value": "2"},
            {"Name": "timeout", "Value": "1"}
        ]
    },
    "Containers": [{"Name": "app", "Image": "nginx:alpine"}]
}
```

## 验证

```bash
kubectl exec <pod> -- cat /etc/resolv.conf
```

## 清理

```bash
tccli tke DeleteEKSContainerInstances --region ap-guangzhou --cli-input-json "{\"EksCiIds\":[\"eksci-example\"]}"
```

## 下一步

- [快速创建一个容器实例](../../../容器实例管理/快速创建一个容器实例/tccli%20操作.md)
- [TKE DNS 最佳实践](../../网络/DNS%20相关/TKE%20DNS%20最佳实践/tccli%20操作.md)

## 控制台替代

控制台：弹性容器 → 新建 → 高级设置 → DNS 配置。
