# 构建深度学习容器镜像（tccli）
> 对照官方：[构建深度学习容器镜像](https://cloud.tencent.com/document/product/457/60220) · page_id `60220`
## 概述
构建包含深度学习框架（PyTorch/TensorFlow）、CUDA 运行时和训练代码的容器镜像，推送至腾讯云容器镜像服务（TCR），为 TKE Serverless GPU 训练任务提供可复用的镜像工件。

**构建流程**: 编写 Dockerfile -> `docker build` -> `docker tag` -> `docker push`（推送至 TCR）

**核心 API**:
- 控制面: `tccli tcr DescribeInstances`, `tccli tcr DescribeNamespaces`, `tccli tcr CreateNamespace`, `tccli tcr CreateRepository`, `tccli tcr DescribeRepositories`, `tccli tcr DescribeImages`, `tccli tcr CreateInstanceToken`
- 数据面: `docker build`, `docker tag`, `docker login`, `docker push`, `docker pull`

## 前置条件

### 环境检查

```bash
# 1. 验证 tccli 版本
tccli --version
# expected: tccli version 1.x.x

# 2. 验证 Docker 环境
docker --version
# expected: Docker version 20.10.x or later

# 3. 验证 Docker daemon 运行
docker info --format '{{.ServerVersion}}'
# expected: (返回 Docker Engine 版本号)

# 4. 验证凭证有效性
tccli sts GetCallerIdentity
# expected:
# {
#     "AccountId": "1xxxxxxxxx",
#     "Arn": "arn:aws:sts::1xxxxxxxxx:assumed-role/...",
#     "UserId": "1xxxxxxxxx",
#     "RequestId": "xxx"
# }

# 5. 验证 CAM Action — TCR 读权限
tccli tcr DescribeInstances \
    --region <Region> \
    --Limit 1
# expected: (含有 RequestId，无 UnauthorizedOperation)
```

```json
{
  "TotalCount": "<TotalCount>",
  "Registries": "<Registries>",
  "RegistryId": "<RegistryId>",
  "RegistryName": "<RegistryName>",
  "RegistryType": "<RegistryType>",
  "Status": "<Status>"
}
```

### 资源检查

```bash
# 6. 查询已有 TCR 企业版实例
tccli tcr DescribeInstances \
    --region <Region> \
    --Limit 20
# expected:
# {
#     "Registries": [
#         {
#             "RegistryId": "tcr-xxxxxxxx",
#             "RegistryName": "...",
#             "RegistryType": "basic",
#             "Status": "Running",
#             "PublicDomain": "xxx.tencentcloudcr.com"
#         }
#     ],
#     "TotalCount": 1,
#     "RequestId": "xxx"
# }

# 7. 查询已有命名空间（检查是否需要创建新命名空间）
tccli tcr DescribeNamespaces \
    --region <Region> \
    --RegistryId tcr-xxxxxxxx
# expected:
# {
#     "NamespaceList": [],
#     "TotalCount": 0,
#     "RequestId": "xxx"
# }

# 8. 查询已有镜像仓库
tccli tcr DescribeRepositories \
    --region <Region> \
    --RegistryId tcr-xxxxxxxx \
    --NamespaceName NAMESPACE
# expected:
# {
#     "RepositoryList": [],
#     "TotalCount": 0,
#     "RequestId": "xxx"
# }

# 9. 检查本地磁盘空间（镜像构建需要 20GB+）
df -h .
# expected: (Available >= 20Gi)
```

```json
{
  "TotalCount": "<TotalCount>",
  "Registries": "<Registries>",
  "RegistryId": "<RegistryId>",
  "RegistryName": "<RegistryName>",
  "RegistryType": "<RegistryType>",
  "Status": "<Status>"
}
```

## 控制台与 CLI 参数映射

| 控制台参数 | CLI 参数 | Idempotency 行为 | 约束 |
|---|---|---|---|
| TCR 实例名 | `RegistryName` (CreateInstance) | 实例级，同名不可重复 | 全局唯一，2-30 字符 |
| 命名空间名称 | `NamespaceName` | 同名冲突报错 | 2-30 字符，小写字母/数字/_- |
| 仓库名称 | `RepositoryName` | 同名冲突报错 | 2-255 字符，小写字母/数字/_-/. |
| 镜像版本 | `docker tag TAG` | 同 Tag docker push 覆盖 | 1-128 字符 |
| 访问凭证 | `CreateInstanceToken` -> `docker login` | Token 可重复创建（不同 TokenId） | Token 类型: temp/permanent |
| Dockerfile 基础镜像 | `FROM <image>` | N/A | nvidia/cuda, pytorch/pytorch 等 |
| 地域 | `--region` | 必填 | REGION |

## 操作步骤

### 占位符说明

| 占位符 | 说明 | 约束 | 获取方式 |
|---|---|---|---|
| `REGION` | TCR 实例地域 | ap-guangzhou | 固定值 |
| `NAMESPACE` | TCR 命名空间 | 2-30 字符，小写字母 | `DescribeNamespaces` 或自定义创建 |
| `TCR_DOMAIN` | TCR 实例公网域名 | xxx.tencentcloudcr.com | `DescribeInstances.PublicDomain` |
| `INSTANCE_ID` | TCR 实例 ID | 格式 tcr-xxxxxxxx | `DescribeInstances.RegistryId` |
| `REGISTRY_ID` | 同 INSTANCE_ID | tcr-xxxxxxxx | `DescribeInstances` |

#### 选择依据

深度学习镜像构建的核心决策在于基础镜像和框架版本的选择:

1. **基础镜像选择**:
   - `nvidia/cuda:12.1.0-cudnn8-devel-ubuntu22.04` — 通用 CUDA 开发环境，需自行安装 PyTorch/TensorFlow
   - `pytorch/pytorch:2.1.0-cuda12.1-cudnn8-devel` — PyTorch 官方镜像，含 PyTorch + CUDA + cuDNN
   - `tensorflow/tensorflow:2.14.0-gpu` — TensorFlow 官方 GPU 镜像
2. **多阶段构建 vs 单阶段构建**: 多阶段构建可显著减小最终镜像体积（开发阶段保留编译工具链，运行阶段仅保留运行时），推荐生产使用
3. **TCR 命名空间规划**: 建议按项目/环境划分（如 `dl-training/prod`），便于权限和镜像生命周期管理
4. **镜像 Tag 策略**: 推荐 `{框架}-{版本}-{日期}` 格式（如 `pytorch-2.1.0-20260617`），避免 `latest` 漂移

#### 最小创建

```bash
# 步骤 1: 创建 TCR 命名空间（如不存在）
tccli tcr CreateNamespace \
    --region <Region> \
    --RegistryId REGISTRY_ID \
    --NamespaceName NAMESPACE \
    --IsPublic false
# expected:
# {
#     "RequestId": "xxx"
# }

# 步骤 2: 在命名空间下创建镜像仓库
tccli tcr CreateRepository \
    --region <Region> \
    --RegistryId REGISTRY_ID \
    --NamespaceName NAMESPACE \
    --RepositoryName "dl-training" \
    --BriefDescription "Deep learning training image" \
    --DetailDescription "PyTorch/TensorFlow training image with CUDA 12.1"
# expected:
# {
#     "RequestId": "xxx"
# }

# 步骤 3: 验证仓库创建成功
tccli tcr DescribeRepositories \
    --region <Region> \
    --RegistryId REGISTRY_ID \
    --NamespaceName NAMESPACE \
    --RepositoryName "dl-training"
# expected:
# {
#     "RepositoryList": [
#         {
#             "Name": "dl-training",
#             "Namespace": "NAMESPACE",
#             "Public": false,
#             "BriefDescription": "Deep learning training image",
#             "CreationTime": "..."
#         }
#     ],
#     "TotalCount": 1,
#     "RequestId": "xxx"
# }
```

```json
{
  "RepositoryList": "<RepositoryList>",
  "Name": "<Name>",
  "Namespace": "<Namespace>",
  "CreationTime": "<CreationTime>",
  "Public": "<Public>",
  "Description": "<Description>"
}
```

#### 增强配置

```bash
# 步骤 4: 编写深度学习 Dockerfile
# 以下使用 PyTorch 官方基础镜像（含 CUDA 12.1 + cuDNN 8）
cat <<'EOF' > Dockerfile
FROM pytorch/pytorch:2.1.0-cuda12.1-cudnn8-devel

# 安装额外 Python 依赖
RUN pip install --no-cache-dir \
    tensorflow==2.14.0 \
    transformers==4.36.0 \
    datasets==2.16.0 \
    accelerate==0.25.0 \
    wandb==0.16.0

# 设置工作目录
WORKDIR /workspace

# 复制训练脚本
COPY train.py /workspace/train.py
COPY requirements.txt /workspace/requirements.txt

# 安装项目依赖
RUN pip install --no-cache-dir -r /workspace/requirements.txt

# 设置默认命令
ENTRYPOINT ["python3", "/workspace/train.py"]
EOF

# 步骤 5: 构建镜像（数据面 docker build）
docker build \
    --tag dl-training:pytorch-2.1.0-cuda12.1 \
    --file Dockerfile \
    .
# expected:
# [+] Building ...
# [+] ... Successfully built xxxxxxxxxxxx
# [+] ... Successfully tagged dl-training:pytorch-2.1.0-cuda12.1

# 步骤 6: 获取 TCR 登录凭证
# >=4 参数，使用 --cli-input-json
cat > create-tcr-token.json <<'EOF'
{
    "RegistryId": "REGISTRY_ID",
    "TokenType": "temp",
    "Desc": "dl-image-build-token",
    "ExpireTime": 86400
}
EOF
tccli tcr CreateInstanceToken \
    --region <Region> \
    --cli-input-json file://create-tcr-token.json
# expected:
# {
#     "Username": "1xxxxxxxxx",
#     "Token": "eyJhbGciOi...",
#     "ExpTime": 1718668800,
#     "RequestId": "xxx"
# }

# 步骤 7: 登录 TCR 并推送镜像
docker login TCR_DOMAIN \
    --username 1xxxxxxxxx \
    --password-stdin <<< "TOKEN_VALUE"
# expected: Login Succeeded

# 标记镜像为 TCR 路径
docker tag \
    dl-training:pytorch-2.1.0-cuda12.1 \
    TCR_DOMAIN/NAMESPACE/dl-training:pytorch-2.1.0-cuda12.1

# 推送镜像到 TCR
docker push TCR_DOMAIN/NAMESPACE/dl-training:pytorch-2.1.0-cuda12.1
# expected:
# The push refers to repository [TCR_DOMAIN/NAMESPACE/dl-training]
# ...
# pytorch-2.1.0-cuda12.1: digest: sha256:xxxxxxxx size: xxxx

# 步骤 8: 验证镜像已推送至 TCR
tccli tcr DescribeImages \
    --region <Region> \
    --RegistryId REGISTRY_ID \
    --NamespaceName NAMESPACE \
    --RepositoryName "dl-training"
# expected:
# {
#     "ImageInfoList": [
#         {
#             "ImageVersion": "pytorch-2.1.0-cuda12.1",
#             "Digest": "sha256:xxxxxxxx",
#             "Size": 8589934592,
#             "UpdateTime": "..."
#         }
#     ],
#     "TotalCount": 1,
#     "RequestId": "xxx"
# }
```

```json
{
  "ImageInfoList": "<ImageInfoList>",
  "Digest": "<Digest>",
  "Size": "<Size>",
  "ImageVersion": "<ImageVersion>",
  "UpdateTime": "<UpdateTime>",
  "Kind": "<Kind>"
}
```

## 验证

### 控制面（tccli）

```bash
# 1. 验证镜像仓库列表
tccli tcr DescribeRepositories \
    --region <Region> \
    --RegistryId REGISTRY_ID \
    --NamespaceName NAMESPACE
# expected:
# {
#     "RepositoryList": [
#         {
#             "Name": "dl-training",
#             "Namespace": "NAMESPACE",
#             "Public": false,
#             "BriefDescription": "Deep learning training image"
#         }
#     ],
#     "TotalCount": 1,
#     "RequestId": "xxx"
# }

# 2. 验证镜像版本及元信息
tccli tcr DescribeImages \
    --region <Region> \
    --RegistryId REGISTRY_ID \
    --NamespaceName NAMESPACE \
    --RepositoryName "dl-training" \
    --ImageVersion "pytorch-2.1.0-cuda12.1"
# expected:
# {
#     "ImageInfoList": [
#         {
#             "ImageVersion": "pytorch-2.1.0-cuda12.1",
#             "Digest": "sha256:xxxxxxxx",
#             "Size": 8589934592,
#             "Kind": "DockerV2Schema2"
#         }
#     ],
#     "TotalCount": 1,
#     "RequestId": "xxx"
# }
```

```json
{
  "RepositoryList": "<RepositoryList>",
  "Name": "<Name>",
  "Namespace": "<Namespace>",
  "CreationTime": "<CreationTime>",
  "Public": "<Public>",
  "Description": "<Description>"
}
```

### 数据面（需 VPN/IOA）

> **kubectl 数据面不可达**：CAM 拒绝公网端点 (strategyId:240463971)，内网需 VPN/IOA。以下 docker 命令为参考格式。

```bash
# 3. 从 TCR 拉取镜像验证完整性
docker pull TCR_DOMAIN/NAMESPACE/dl-training:pytorch-2.1.0-cuda12.1
# expected:
# pytorch-2.1.0-cuda12.1: Pulling from NAMESPACE/dl-training
# Digest: sha256:xxxxxxxx
# Status: Downloaded newer image for TCR_DOMAIN/NAMESPACE/dl-training:pytorch-2.1.0-cuda12.1

# 4. 验证镜像中的 CUDA 版本
docker run --rm TCR_DOMAIN/NAMESPACE/dl-training:pytorch-2.1.0-cuda12.1 \
    python3 -c "import torch; print(f'PyTorch {torch.__version__}, CUDA available: {torch.cuda.is_available()}')"
# expected:
# PyTorch 2.1.0, CUDA available: True

# 5. 验证训练脚本可执行
docker run --rm TCR_DOMAIN/NAMESPACE/dl-training:pytorch-2.1.0-cuda12.1 \
    python3 -c "import torch; print('OK')"
# expected: OK
```

## 清理

> **计费警告**: TCR 按实例规格和存储容量计费。删除镜像可释放存储空间，但 TCR 实例（企业版）本身持续计费。如需完全清理，请一并删除 TCR 实例。

> **副作用警告**: 删除镜像版本后不可恢复，依赖该镜像的 EKS 训练任务将无法拉取。请确认无在运行或待调度的任务使用该镜像。

### 控制面（tccli）

```bash
# 1. 删除指定镜像版本
tccli tcr DeleteImage \
    --region <Region> \
    --RegistryId REGISTRY_ID \
    --NamespaceName NAMESPACE \
    --RepositoryName "dl-training" \
    --ImageVersion "pytorch-2.1.0-cuda12.1"
# expected:
# {
#     "RequestId": "xxx"
# }

# 2. 验证镜像已删除
tccli tcr DescribeImages \
    --region <Region> \
    --RegistryId REGISTRY_ID \
    --NamespaceName NAMESPACE \
    --RepositoryName "dl-training"
# expected:
# {
#     "ImageInfoList": [],
#     "TotalCount": 0,
#     "RequestId": "xxx"
# }

# 3. (可选) 删除仓库
tccli tcr DeleteRepository \
    --region <Region> \
    --RegistryId REGISTRY_ID \
    --NamespaceName NAMESPACE \
    --RepositoryName "dl-training"
# expected:
# {
#     "RequestId": "xxx"
# }

# 4. (可选) 删除命名空间
tccli tcr DeleteNamespace \
    --region <Region> \
    --RegistryId REGISTRY_ID \
    --NamespaceName NAMESPACE
# expected:
# {
#     "RequestId": "xxx"
# }

# 5. 验证仓库已删除
tccli tcr DescribeRepositories \
    --region <Region> \
    --RegistryId REGISTRY_ID \
    --NamespaceName NAMESPACE
# expected:
# {
#     "RepositoryList": [],
#     "TotalCount": 0,
#     "RequestId": "xxx"
# }
```

```json
{
  "ImageInfoList": "<ImageInfoList>",
  "Digest": "<Digest>",
  "Size": "<Size>",
  "ImageVersion": "<ImageVersion>",
  "UpdateTime": "<UpdateTime>",
  "Kind": "<Kind>"
}
```

### 数据面

```bash
# 6. 清理本地 Docker 镜像
docker rmi dl-training:pytorch-2.1.0-cuda12.1
docker rmi TCR_DOMAIN/NAMESPACE/dl-training:pytorch-2.1.0-cuda12.1
docker rmi pytorch/pytorch:2.1.0-cuda12.1-cudnn8-devel
# expected: (依次 Untagged/Deleted)

# 7. 清理 Docker 构建缓存（释放磁盘空间）
docker builder prune --force
# expected: Total reclaimed space: xxxGB
```

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|---|---|---|---|
| `docker build` 返回 `manifest unknown` | `docker pull` 确认基础镜像可访问 | 基础镜像 Tag 不存在或 Docker Hub 不可达 | 确认基础镜像 Tag 拼写正确；配置 Docker Hub 镜像加速器 |
| `docker build` 返回 `no space left on device` | `df -h` 检查磁盘空间 | 磁盘空间不足以存储中间层镜像 | `docker system prune -a` 清理无用镜像和缓存 |
| `docker push` 返回 `unauthorized` | `docker logout` + `docker login` 重新认证 | TCR Token 过期或用户名不匹配 | 重新创建 TCR Token，使用 `CreateInstanceToken` 获取新凭证 |
| `CreateNamespace` 返回 `ResourceConflict` | `DescribeNamespaces` 确认命名空间存在 | 同名命名空间已存在 | 复用已有命名空间或使用不同名称 |
| `CreateRepository` 返回 `InvalidParameterValue` | 仓库名称含非法字符 | 名称含大写字母或特殊字符 | 仓库名仅允许小写字母、数字、`_`、`-`、`.` |
| `CreateInstanceToken` 返回 `LimitExceeded` | Token 数量超出限制 | 实例 Token 配额已满 | 删除过期或不用的 Token |

### 创建成功但状态异常

| 现象 | 诊断 | 根因 | 修复 |
|---|---|---|---|
| docker build 成功但镜像体积过大 (>20GB) | `docker images dl-training` | 单阶段构建包含开发工具和中间文件 | 改用多阶段构建，运行阶段使用 `python:3.10-slim` + `pip install torch` |
| docker push 速度极慢 | 检查网络带宽，`docker push` 输出速度 | 公网上传大镜像速度受限 | 使用 TCR 内网域名推送（需 VPN/IOA），或在同地域 CVM 上构建推送 |
| DescribeImages 返回空列表 | 检查 Region 和 RegistryId 是否匹配 | 所用 Region 与 TCR 实例不在同一地域 | 确认 `--region` 与 TCR 实例创建地域一致 |
| 镜像拉取时提示 digest 不匹配 | 可能存在同名 Tag 覆盖 | Tag 被其他构建覆盖（非不可变 Tag） | 开启 TCR 的 Tag 不可变 (Immutable Tag) 功能；使用 Digest 拉取 |
| 容器内 CUDA 不可用 | `nvidia-smi` 或 `torch.cuda.is_available()` | 基础镜像未含 CUDA 驱动或使用 CPU-only 镜像 | 确认基础镜像为 `-devel` 或 `-runtime` 变体，非 `-slim` 变体 |

## 下一步

- [在 TKE Serverless 上运行深度学习](../在%20TKE%20Serverless%20上运行深度学习/tccli%20操作.md)：使用本镜像在 EKS GPU Pod 上运行训练任务
- [TCR 产品文档](https://cloud.tencent.com/document/product/1141)：容器镜像服务完整文档，含镜像安全扫描、同步复制
- [NVIDIA Container Toolkit](https://docs.nvidia.com/datacenter/cloud-native/)：GPU 容器运行时配置参考

## 控制台替代
控制台完成本节操作需在 **容器镜像服务 > 镜像仓库** 页面手动创建命名空间和仓库，然后在本地执行 docker 命令构建和推送。tccli 将 TCR 资源管理（命名空间、仓库、Token）与 docker 命令编排为自动化流水线，通过 `DescribeRepositories` / `DescribeImages` 提供状态闭环，适合 CI/CD pipeline（如 GitHub Actions、CODING DevOps）中的镜像构建和推送阶段。
