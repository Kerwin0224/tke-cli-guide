# 容器镜像签名及验证（tccli）

> 对照官方：[容器镜像签名及验证](https://cloud.tencent.com/document/product/457/80909) · page_id `80909`

## 概述

镜像签名和验签功能可避免中间人攻击和非法镜像的更新及运行，实现镜像从分发到部署的全链路一致性。本文涵盖两阶段：**TCR 侧签名**（TCR 高级版自动签名）和 **TKE 侧验证**（Cerberus 组件验签）。两者分别指向对应的产品文档页，本页提供总览和 CLI 入口。

## 前置条件

- 已创建 [TCR 企业版高级版实例](https://cloud.tencent.com/document/product/1141/40716)（签名功能仅高级版支持）。
- 已创建 TKE 集群，kubectl 可达。
- 已 [连接集群](../../../集群配置/集群管理/连接集群/tccli%20操作.md)。

```bash
tccli tke DescribeClusters --region ap-guangzhou --output json \
  --filter "Clusters[?ClusterId=='<ClusterId>'] | [0].{ClusterId:ClusterId,ClusterStatus:ClusterStatus}"
```

```
{
  "ClusterId": "cls-example",
  "ClusterStatus": "Running"
}
```

## 控制台与 CLI 参数映射

| 控制台操作 | CLI | 幂等 |
|-----------|-----|------|
| 创建 TCR 签名策略 | TCR 控制台 / `tccli tcr CreateSignaturePolicy` | 否 |
| 推送镜像触发签名 | `docker push` | 是（覆盖同 tag） |
| 安装 Cerberus 组件 | `tccli tke InstallAddon --AddonName Cerberus` | 是（已安装则跳过） |
| 检查组件状态 | `tccli tke DescribeAddon --ClusterId <ClusterId> --AddonName Cerberus` | 是 |

## 操作步骤

### 1. 容器镜像签名（TCR 侧）

TCR **高级版**支持开启命名空间级别的镜像自动签名。推送镜像到仓库时自动匹配签名策略完成加签。

详细操作参见 [容器镜像签名](https://cloud.tencent.com/document/product/1141/80862)（TCR 产品文档）。

**限制条件**：
- 仅 TCR **高级版**实例支持。
- 需先在 TCR 控制台配置签名策略，再推送镜像触发自动签名。

### 2. 镜像签名验证（TKE 侧）

TKE 提供 **Cerberus** 组件，对签名镜像进行可信验证，确保只在集群中部署可信授权方签名的容器镜像。

#### 2.1 安装 Cerberus 组件

```bash
tccli tke InstallAddon \
  --ClusterId <ClusterId> \
  --AddonName Cerberus \
  --region ap-guangzhou
```

```
{
  "RequestId": "xxxx-xxxx-xxxx-xxxx"
}
```

#### 2.2 查看组件状态

```bash
tccli tke DescribeAddon --ClusterId <ClusterId> --AddonName Cerberus --region ap-guangzhou --output json \
  --filter "{Name:Name,Status:Status,Version:Version}"
```

```
{
  "Name": "Cerberus",
  "Status": "Running",
  "Version": "1.0.0"
}
```

### 3. 验证签名镜像部署策略

Cerberus 安装后，集群中的工作负载若引用了未签名或签名不匹配的镜像，将根据策略被阻止部署。

详细配置参见 [Cerberus 组件使用说明](https://cloud.tencent.com/document/product/457/80899)。

## 验证

### Control plane (tccli)

```bash
tccli tke DescribeAddon --ClusterId <ClusterId> --AddonName Cerberus --region ap-guangzhou --output json \
  --filter "{Status:Status}"
```

```
{
  "Status": "Running"
}
```

### Data plane (kubectl)

```bash
kubectl get pods -n cerberus-system
```

```
NAME                          READY   STATUS    RESTARTS   AGE
cerberus-controller-xxx-yyy   1/1     Running   0          2m
```

## 排障

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `kubectl` 连接超时/拒绝 | `tccli tke DescribeClusters` 返回集群 Running 但 kubectl 不可达 | CAM 策略 240463971 拒绝公网访问 APIServer 端点 | 通过 VPN/IOA 接入内网后执行 kubectl 命令 |
| `InstallAddon Cerberus` 报错 | `tccli tke DescribeAddon --ClusterId <ClusterId> --AddonName Cerberus` 查看状态 | 集群版本不满足组件最低要求或集群状态非 Running | 升级集群版本；确认集群状态为 Running |
| 镜像签名不触发 | TCR 控制台查看命名空间签名策略 | TCR 实例非高级版或命名空间未配置签名策略 | 确认 TCR 实例为高级版；在命名空间配置签名策略后重新推送镜像 |
| Cerberus 组件状态非 Running | `kubectl describe pod -n cerberus-system` 查看事件日志 | 组件 Pod 启动失败 | 查看 Pod Events 和日志排查；确认集群资源充足 |
| Pod 因验签失败被拒绝 | `kubectl describe pod <pod>` 查看 Events | 镜像未签名或签名密钥不匹配 | 检查镜像仓库中的签名状态；确认签名密钥与验签策略一致 |

## 清理

### Control plane (tccli)

```bash
tccli tke DeleteAddon --ClusterId <ClusterId> --AddonName Cerberus --region ap-guangzhou
```

## 下一步

- [容器镜像签名（TCR 产品文档）](https://cloud.tencent.com/document/product/1141/80862)
- [Cerberus 组件使用说明](https://cloud.tencent.com/document/product/457/80899)
- [Pod 使用 CAM 对数据库身份验证](../Pod 使用 CAM 数据库验证/tccli 操作.md)（page_id `81989`）

## 控制台替代

TCR 控制台 **命名空间 → 镜像签名策略 → 新建策略**；TKE 控制台 **集群 → 组件管理 → 新建 → Cerberus**。
