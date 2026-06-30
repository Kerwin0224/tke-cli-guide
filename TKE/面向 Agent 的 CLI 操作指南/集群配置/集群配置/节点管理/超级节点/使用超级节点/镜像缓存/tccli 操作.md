# 镜像缓存（tccli）

> 对照官方：[镜像缓存](https://cloud.tencent.com/document/product/457/65908) · page_id `65908`  

## 概述

**镜像缓存**加速超级节点拉镜；通过 Pod Annotation（如 `cbs-reuse-key`）或镜像缓存资源（官方 CRD/API）实现。

## 前置条件

- [超级节点 Annotation 说明](../超级节点 Annotation 说明/tccli%20操作.md)

## 控制台与 CLI 参数映射

| 方式 | 说明 |
|------|------|
| Annotation | `cbs-reuse-key` 等（官方镜像缓存节） |
| 镜像缓存资源 | 控制台/ YAML 创建缓存 |

## 操作步骤

### 镜像复用 / 镜像缓存

在 Pod 模板添加官方文档指定的镜像缓存 Annotation；多 Pod 共享同一缓存键可复用层。

```bash
tccli tke DescribeClusterVirtualNodePools --ClusterId <ClusterId> --region ap-guangzhou --output json
```

```json
{
  "TotalCount": 0,
  "NodePoolSet": [],
  "NodePoolId": "<NodePoolId>",
  "SubnetIds": [],
  "Name": "<Name>",
  "LifeState": "<LifeState>",
  "Labels": []
}
```

## 验证

二次创建同镜像 Pod 启动时间明显缩短（数据面观察）。

## 清理

删除缓存资源与测试 Pod。

## 排障

缓存未命中：查 key 是否一致、镜像名/tag。

## 下一步

- [Annotation 说明](../超级节点 Annotation 说明/tccli%20操作.md)

## 控制台替代

**镜像缓存** 控制台页。
