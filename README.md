# TKE + TCR · 面向 Agent 的 CLI 操作指南

腾讯云容器产品线 `tccli` 命令行操作指南，覆盖容器服务（TKE）和容器镜像服务（TCR），面向 AI Agent 和运维人员。

<table>
<tr>
<td width="50%" valign="top">

### 🚀 TKE 容器服务

管理 Kubernetes 集群和节点。

| 分类 | 场景数 |
|------|:----:|
| 快速入门 | 6 |
| 集群配置 | 100+ |
| 应用配置 | 80+ |
| 调度配置 | 60+ |
| 安全和稳定性 | 90+ |
| 故障处理 | 30+ |
| Data / AI 实践 | 40+ |

[→ 进入 TKE 文档](TKE/%E9%9D%A2%E5%90%91%20Agent%20%E7%9A%84%20CLI%20%E6%93%8D%E4%BD%9C%E6%8C%87%E5%8D%97/README.md)

```bash
tccli tke CreateCluster --ClusterName "my-cluster" ...
```

</td>
<td width="50%" valign="top">

### 📦 TCR 容器镜像服务

管理容器镜像仓库和分发。

| 分类 | 场景数 |
|------|:----:|
| 快速入门 | 2 |
| 实例管理 | 2 |
| 访问配置 | 8 |
| 镜像管理 | 13 |
| 个人版操作 | 6 |
| 实践教程 | 10 |

[→ 进入 TCR 文档](TCR/%E9%9D%A2%E5%90%91%20Agent%20%E7%9A%84%20CLI%20%E6%93%8D%E4%BD%9C%E6%8C%87%E5%8D%97/README.md)

```bash
tccli tcr CreateInstance --RegistryName "my-registry" ...
```

</td>
</tr>
</table>

---

### 🔗 常见交叉场景

| 场景 | 涉及产品 | 链接 |
|------|:------:|------|
| TKE 集群使用 TCR 内网免密拉取镜像 | 🚀 + 📦 | [TKE 侧](TKE/%E9%9D%A2%E5%90%91%20Agent%20%E7%9A%84%20CLI%20%E6%93%8D%E4%BD%9C%E6%8C%87%E5%8D%97/%E5%AE%9E%E8%B7%B5%E6%95%99%E7%A8%8B/TKE%20%E9%9B%86%E7%BE%A4%E4%BD%BF%E7%94%A8%20TCR%20%E6%8F%92%E4%BB%B6%E5%86%85%E7%BD%91%E5%85%8D%E5%AF%86%E6%8B%89%E5%8F%96%E5%AE%B9%E5%99%A8%E9%95%9C%E5%83%8F/tccli%20%E6%93%8D%E4%BD%9C.md) · [TCR 侧](TCR/%E9%9D%A2%E5%90%91%20Agent%20%E7%9A%84%20CLI%20%E6%93%8D%E4%BD%9C%E6%8C%87%E5%8D%97/%E5%AE%9E%E8%B7%B5%E6%95%99%E7%A8%8B/TKE%20%E9%9B%86%E7%BE%A4%E4%BD%BF%E7%94%A8%20TCR%20%E6%8F%92%E4%BB%B6%E5%86%85%E7%BD%91%E5%85%8D%E5%AF%86%E6%8B%89%E5%8F%96%E5%AE%B9%E5%99%A8%E9%95%9C%E5%83%8F/tccli%20%E6%93%8D%E4%BD%9C.md) |
| TKE Serverless 集群拉取 TCR 镜像 | 🚀 + 📦 | [TKE 侧](TKE/%E9%9D%A2%E5%90%91%20Agent%20%E7%9A%84%20CLI%20%E6%93%8D%E4%BD%9C%E6%8C%87%E5%8D%97/%E5%AE%9E%E8%B7%B5%E6%95%99%E7%A8%8B/TKE%20Serverless%20%E9%9B%86%E7%BE%A4%E6%8B%89%E5%8F%96%20TCR%20%E5%AE%B9%E5%99%A8%E9%95%9C%E5%83%8F/tccli%20%E6%93%8D%E4%BD%9C.md) · [TCR 侧](TCR/%E9%9D%A2%E5%90%91%20Agent%20%E7%9A%84%20CLI%20%E6%93%8D%E4%BD%9C%E6%8C%87%E5%8D%97/%E5%AE%9E%E8%B7%B5%E6%95%99%E7%A8%8B/TKE%20Serverless%20%E9%9B%86%E7%BE%A4%E6%8B%89%E5%8F%96%20TCR%20%E5%AE%B9%E5%99%A8%E9%95%9C%E5%83%8F/tccli%20%E6%93%8D%E4%BD%9C.md) |
| 从自建 Harbor 同步镜像到 TCR → TKE 部署 | 🚀 + 📦 | [TCR 侧](TCR/%E9%9D%A2%E5%90%91%20Agent%20%E7%9A%84%20CLI%20%E6%93%8D%E4%BD%9C%E6%8C%87%E5%8D%97/%E5%AE%9E%E8%B7%B5%E6%95%99%E7%A8%8B/%E4%BB%8E%E8%87%AA%E5%BB%BA%20Harbor%20%E5%90%8C%E6%AD%A5%E9%95%9C%E5%83%8F%E5%88%B0%20TCR%20%E4%BC%81%E4%B8%9A%E7%89%88/tccli%20%E6%93%8D%E4%BD%9C.md) |

---

### 🛠 环境准备

两个产品线共享同一套 `tccli` 配置：

```bash
# 安装 tccli
pip install tccli

# 配置凭据
tccli configure set secretId <SecretId> secretKey <SecretKey>
tccli configure set region ap-guangzhou
```

详见 [环境准备 (TKE)](TKE/%E9%9D%A2%E5%90%91%20Agent%20%E7%9A%84%20CLI%20%E6%93%8D%E4%BD%9C%E6%8C%87%E5%8D%97/%E7%8E%AF%E5%A2%83%E5%87%86%E5%A4%87.md) · [环境准备 (TCR)](TCR/%E9%9D%A2%E5%90%91%20Agent%20%E7%9A%84%20CLI%20%E6%93%8D%E4%BD%9C%E6%8C%87%E5%8D%97/%E7%8E%AF%E5%A2%83%E5%87%86%E5%A4%87.md)
