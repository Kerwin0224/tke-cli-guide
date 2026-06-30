# 镜像分层最佳实践（tccli）

> 对照官方：[镜像分层最佳实践](https://cloud.tencent.com/document/product/457/68383) · page_id `68383`

## 概述

将业务镜像按 **Base（系统层）→ Ops（运维层）→ Lang（语言层）→ App（应用层）** 分层构建，利用 Docker 多阶段构建和 TCR 企业版管理各类容器镜像。分层构建可提升资源共享率、规范化镜像管理，配合 TCR 免运维和镜像加速能力，大幅提升大规模镜像分发速度。

## 前置条件

- 已 [创建 TCR 企业版实例](https://cloud.tencent.com/document/product/1141/40716) 并完成登录。
- 本地安装 Docker 客户端。
- 如需在 TKE 集群内构建，已 [连接集群](../../../集群配置/集群管理/连接集群/tccli%20操作.md)。

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
| 构建 Base 镜像 | `docker build -t <registry>/<repo>:<tag> -f Dockerfile .` | 否（覆盖 tag） |
| 推送镜像 | `docker push <registry>/<repo>:<tag>` | 是（覆盖） |
| 查看 TCR 仓库 | `tccli tcr DescribeRepositories` | 是 |
| 查看命名空间 | `tccli tcr DescribeNamespaces` | 是 |

## 操作步骤

### 1. 分层结构概览

```
f3s-docker-files/
├── 0.base/                        # Base 层：操作系统镜像
│   ├── alpine/Dockerfile          #   alpine:3.13 + 运维工具
│   └── centos-7.8/Dockerfile      #   CentOS 7.8 + 运维工具
├── 1.ops/                         # 运维层：运维工具扩展
│   └── Dockerfile-alpine          #   alpine-base + glibc + bash + vim
├── 2.lang/                        # 语言层：语言运行时
│   └── Dockerfile-alpine-kona     #   ops + TencentKona JDK8
└── 3.app/                         # 应用层：业务应用
    ├── jmeter/                    #   JMeter 压测镜像系列
    ├── nginx/                     #   Nginx 镜像
    └── skywalking/                #   SkyWalking + Kona
```

### 2. Base 层构建（Alpine）

```dockerfile
FROM alpine:3.13
ENV TZ=Asia/Shanghai
RUN echo 'http://mirrors.tencent.com/alpine/v3.13/main/' > /etc/apk/repositories \
    && apk --no-cache add curl jq tzdata iputils wget \
    && ln -sf /usr/share/zoneinfo/${TZ} /etc/localtime \
    && rm -rf /var/cache/apk/*
```

```bash
docker build -t f3s-docker-file.tencentcloudcr.com/f3s-tcr/alpine:v3.13 -f Dockerfile .
docker push f3s-docker-file.tencentcloudcr.com/f3s-tcr/alpine:v3.13
```

```
v3.13: digest: sha256:abc123... size: 1234
```

### 3. 语言层构建（Alpine + Kona JDK）

```dockerfile
FROM f3s-docker-file.tencentcloudcr.com/f3s-tcr/alpine:latest
ENV LANG=C.UTF-8
RUN cd /opt \
    && wget https://github.com/Tencent/TencentKona-8/releases/download/8.0.5-GA/TencentKona8.0.5.b12_jdk_linux-x86_64_8u282.tar.gz \
    && tar -xvf TencentKona8.0.5.b12_jdk_linux-x86_64_8u282.tar.gz \
    && ln -nfs /opt/TencentKona-8.0.5-282 /opt/jdk
ENV JAVA_HOME=/opt/jdk
ENV PATH=$JAVA_HOME/bin:$PATH
```

```bash
docker build --no-cache -t f3s-docker-file.tencentcloudcr.com/f3s-tcr/alpine-kona:latest -f Dockerfile-alpine-kona .
docker push f3s-docker-file.tencentcloudcr.com/f3s-tcr/alpine-kona:latest
```

### 4. 应用层构建（Nginx）

```dockerfile
FROM f3s-docker-file.tencentcloudcr.com/f3s-tcr/alpine-kona:latest
RUN apk --no-progress --no-cache add nginx
COPY nginx.conf /etc/nginx/nginx.conf
EXPOSE 80 443
CMD ["nginx", "-g", "daemon off;"]
```

```bash
docker build --no-cache -t f3s-docker-file.tencentcloudcr.com/f3s-tcr/alpine-nginx:latest -f Dockerfile-alpine-nginx .
docker push f3s-docker-file.tencentcloudcr.com/f3s-tcr/alpine-nginx:latest
```

## 验证

```bash
docker images | grep f3s-docker-file
```

```
f3s-docker-file.tencentcloudcr.com/f3s-tcr/alpine-nginx    latest    sha256:...
f3s-docker-file.tencentcloudcr.com/f3s-tcr/alpine-kona      latest    sha256:...
f3s-docker-file.tencentcloudcr.com/f3s-tcr/alpine           v3.13     sha256:...
```

## 清理

> 本文操作为只读/非持久化操作，无需清理。

## 排障

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `kubectl` 连接超时/拒绝 | `tccli tke DescribeClusters` 返回集群 Running 但 kubectl 不可达 | CAM 策略 240463971 拒绝公网访问 APIServer 端点 | 通过 VPN/IOA 接入内网后执行 kubectl 命令 |
| `docker push` 报 `unauthorized` | `docker login <registry>` 重新登录 TCR | TCR 登录凭证过期或未登录 | 执行 `docker login <registry>` 重新登录 |
| 镜像构建失败（网络） | 构建日志查看 `wget`/`apk` 下载报错 | 包源走公网访问慢或被限速 | 将包源切换为腾讯云镜像（如 `mirrors.tencent.com`） |
| 镜像体积过大 | `docker history <image>` 查看各层大小 | RUN 指令过多产生中间层 | 合并 RUN 指令减少层数；清理缓存（`rm -rf /var/cache/apk/*`） |
| JDK 镜像中文字体乱码 | 容器内执行 `fc-list :lang=zh` 查看可用字体 | 镜像缺少中文字体包 | 安装 `kde-l10n-Chinese` 和 `glibc-common` 并设置 `LANG=zh_CN.utf8` |

## 下一步

- [境外镜像拉取加速](../境外镜像拉取加速/tccli 操作.md)（page_id `51237`）
- [资源利用率提升工具大全](../../成本管理/资源利用率分析和提升最佳实践/tccli 操作.md)（page_id `80787`）
- [TCR 企业版指南](https://cloud.tencent.com/document/product/1141/39287)

## 控制台替代

控制台 TCR **命名空间 → 镜像仓库**查看推送的镜像列表；本地使用 Docker Desktop 构建后 `docker push`。
