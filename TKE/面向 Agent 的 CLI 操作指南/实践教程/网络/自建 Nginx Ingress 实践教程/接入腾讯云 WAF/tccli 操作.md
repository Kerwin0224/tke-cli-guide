# 接入腾讯云 WAF（tccli）

> 对照官方：[接入腾讯云 WAF](https://cloud.tencent.com/document/product/457/104862) · page_id `104862`

## 概述

将自建 Nginx Ingress Controller 的 CLB 接入腾讯云 Web 应用防火墙（WAF），实现对 HTTP/HTTPS 流量的安全防护：SQL 注入、XSS、CC 攻击等。WAF 支持两种接入模式：CLB 旁路模式和 SaaS 引流模式。

**接入模式对比**：

| 模式 | 原理 | 适用场景 | 配置复杂度 |
|------|------|---------|-----------|
| CLB 型 WAF 旁路 | WAF 旁路监听 CLB 流量，检测+拦截 | CLB 已存在，最小改造 | 低（CLB 级配置） |
| SaaS 型 WAF 引流 | DNS 指向 WAF 实例，WAF 反向代理回源 | 需要域名级防护，精细规则 | 中（DNS 变更） |

选择 CLB 型 WAF：已有 CLB 且不希望修改 DNS。选择 SaaS 型 WAF：需要域名粒度的独立防护策略。

> **kubectl 数据面不可达**：CAM 拒绝公网端点（strategyId:240463971），内网需 VPN/IOA。以下 kubectl 命令为参考格式，需在内网/VPN 环境下执行。

## 前置条件

- [环境准备](../../../../环境准备.md)

### 环境检查

```bash
# 1. 检查 tccli 版本
tccli --version
# expected: tccli version >= 1.0.0

# 2. 检查当前凭据和地域
tccli configure list
# expected: secretId、secretKey、region 均已配置

# 3. 检查 CAM 权限 — 需要以下 Action 名
#    waf:CreateAccessExport, waf:DescribeDomains, waf:AddSpartaProtection
#    waf:DescribeInstances, waf:ModifyDomainClbMode
#    clb:DescribeLoadBalancers, tke:DescribeClusters
# 验证
tccli tke DescribeClusters --region <Region>
# expected: exit 0

tccli clb DescribeLoadBalancers --region <Region>
# expected: exit 0

# WAF 服务在不同地域
tccli waf DescribeInstances --region ap-guangzhou
# expected: exit 0，返回实例列表

# 4. 检查 kubectl 和 Helm（须 VPN/IOA）
kubectl version --client
# expected: Client Version v1.30+
```

```json
{
  "TotalCount": "<TotalCount>",
  "Clusters": "<Clusters>",
  "ClusterId": "<ClusterId>",
  "ClusterName": "<ClusterName>",
  "ClusterDescription": "<ClusterDescription>",
  "ClusterVersion": "<ClusterVersion>"
}
```

### 资源检查

```bash
# 5. 确认 Nginx Ingress Controller 已运行
helm list -n ingress-nginx
# expected: nginx-ingress 状态 deployed

# 6. 获取 CLB ID
kubectl -n ingress-nginx get svc nginx-ingress-ingress-nginx-controller \
    -o jsonpath='{.metadata.annotations.service\.kubernetes\.io/loadbalance-id}'
# expected: lb-xxxxxxxx

# 7. 确认 CLB 监听器已配置
tccli clb DescribeListeners --region <Region> \
    --LoadBalancerId CLB_ID
# expected: HTTP:80 和/或 HTTPS:443 监听器

# 8. 确认 WAF 实例
tccli waf DescribeInstances --region ap-guangzhou
# expected: 至少 1 个 WAF 实例
```

## 控制台与 CLI 参数映射

| 控制台操作 | CLI | 幂等 |
|-----------|-----|:--:|
| 查看 WAF 实例 | `tccli waf DescribeInstances` | 是 |
| 添加防护域名（CLB 模式） | `tccli waf AddSpartaProtection` | 否 |
| 查看防护域名 | `tccli waf DescribeDomains` | 是 |
| 修改域名 CLB 模式 | `tccli waf ModifyDomainClbMode` | 否 |
| 查看 CLB 列表 | `tccli clb DescribeLoadBalancers` | 是 |
| 查看 CLB 监听器 | `tccli clb DescribeListeners` | 是 |

## 关键字段说明

### WAF API 参数（`AddSpartaProtection`）

| 字段 | 类型 | 必填 | 取值与约束 | 错误后果 |
|------|------|:--:|------|------|
| `Domain` | String | 是 | 防护域名，如 `www.example.com`。支持泛域名 `*.example.com` | 域名格式错误 → `InvalidParameter.Domain` |
| `CertType` | Integer | 是 | 0（不启用 HTTPS 或不使用 WAF 证书），1（使用 WAF 管理证书），2（使用托管证书） | 填错 → HTTPS 防护不生效 |
| `IsCdn` | Integer | 是 | 0（非 CDN 源），1（CDN 源），3（TKE Deployment）。TKE 环境选 3 | 选错 → 回源逻辑异常 |
| `UpstreamType` | Integer | 是 | 0（IP 回源），1（域名回源），2（COS 回源），3（CLB 回源） | 选错 → 流量无法回源 |
| `UpstreamDomain` | String | 是（domain 回源时） | 回源域名或 IP | 填错 → 回源失败 |
| `LoadBalance` | String | 是（CLB 模式） | `default`（IP hash），`balance`（加权轮询），`latency`（最快响应） | 选错 → 负载不均 |
| `Ports` | Array | 是 | 监听端口列表，如 `[{"Port":"80","Protocol":"http"}]` | 端口与 CLB 不一致 → 流量不通 |
| `HttpsRewrite` | Integer | 否 | HTTP→HTTPS 重定向，0（不重定向），1（重定向） | 设 1 但 HTTPS 端口未配 → 访问异常 |
| `SrcList` | Array | 否 | 回源 IP 列表或域名（upstream 为 IP 时） | 回源地址不可达 → 流量不通 |

## 操作步骤

### 步骤 1：查询 WAF 实例 ID

```bash
tccli waf DescribeInstances --region ap-guangzhou
```

```json
{
  "Total": 1,
  "Instances": [
    {
      "InstanceId": "waf-example",
      "InstanceName": "WAF_INSTANCE_NAME",
      "Level": 3,
      "Status": 1
    }
  ]
}
```

### 步骤 2：接入 WAF（CLB 旁路模式）

#### 选择依据

- **CLB 旁路模式**：WAF 节点旁路接入 CLB 流量，对现有架构改动最小。CLB 监听器不变，WAF 在 CLB 维度进行检测和拦截。
- **防护范围**：CLB 模式下，WAF 对该 CLB 上所有域名进行防护。如需域名粒度隔离，使用 SaaS 模式。
- **回源方式**：TKE 环境 WAF 回源到 CLB，选 `UpstreamType=3`（CLB 回源）、`IsCdn=3`（TKE Deployment）。

`waf-add-domain.json`：

```json
{
  "Domain": "WAF_DOMAIN",
  "InstanceId": "WAF_INSTANCE_ID",
  "IsCdn": 3,
  "UpstreamType": 3,
  "Ports": [
    {"Port": "80", "Protocol": "http"}
  ],
  "LoadBalance": "default",
  "SrcList": ["CLB_VIP"]
}
```

```bash
tccli waf AddSpartaProtection --region ap-guangzhou --cli-input-json file://waf-add-domain.json
# expected: exit 0，返回 DomainId
```

```json
{
  "DomainId": "DOMAIN_ID"
}
```

| 占位符 | 说明 | 约束 | 获取方式 |
|--------|------|------|---------|
| `WAF_DOMAIN` | 防护域名 | 域名已备案，或使用测试域名 | 自定义 |
| `WAF_INSTANCE_ID` | WAF 实例 ID | 格式 `waf-xxxxxxxx` | `tccli waf DescribeInstances --region ap-guangzhou` |
| `CLB_VIP` | CLB VIP 地址 | WAF 回源到该 IP | `kubectl -n ingress-nginx get svc` EXTERNAL-IP 列 |

### 步骤 3：配置 HTTPS 防护（可选）

如果 Nginx Ingress 已配置 SSL 证书，WAF 上也要配置对应的 HTTPS 端口：

`waf-add-https.json`：

```json
{
  "Domain": "WAF_DOMAIN",
  "InstanceId": "WAF_INSTANCE_ID",
  "IsCdn": 3,
  "UpstreamType": 3,
  "CertType": 0,
  "Ports": [
    {"Port": "80", "Protocol": "http"},
    {"Port": "443", "Protocol": "https"}
  ],
  "HttpsRewrite": 1,
  "LoadBalance": "default",
  "SrcList": ["CLB_VIP"]
}
```

> **CertType=0**：当 SSL 证书在 CLB/Ingress 层面终结时（非 WAF 管理），选 0。

### 步骤 4：验证 WAF 接入状态

```bash
tccli waf DescribeDomains --region ap-guangzhou \
    --InstanceId WAF_INSTANCE_ID
```

```json
{
  "Total": 1,
  "Domains": [
    {
      "Domain": "WAF_DOMAIN",
      "DomainId": "DOMAIN_ID",
      "InstanceId": "waf-example",
      "Cname": "WAF_DOMAIN.waf.qcloud.com",
      "Edition": "clb-waf",
      "State": 1,
      "ClsStatus": 1
    }
  ]
}
```

## 验证

### 控制面（tccli）

```bash
# 验证 WAF 域名接入状态
tccli waf DescribeDomains --region ap-guangzhou \
    --InstanceId WAF_INSTANCE_ID
# expected: State 为 1（正常），Edition 为 "clb-waf"

# 验证 CLB 状态
tccli clb DescribeLoadBalancers --region <Region> \
    --LoadBalancerIds '["CLB_ID"]'
# expected: Status 为 1
```

### 数据面（需 VPN/IOA）

```bash
# 测试 WAF 防护是否生效（发送测试攻击请求）
curl -X GET "http://EXTERNAL_IP/?id=1' OR '1'='1" -H "Host: WAF_DOMAIN"
# expected: WAF 拦截返回 403 或 WAF 拦截页面

# 测试正常请求
curl -X GET "http://EXTERNAL_IP/" -H "Host: WAF_DOMAIN"
# expected: 正常响应 200

# 查看 Nginx Ingress 日志确认流量经过 WAF
kubectl -n ingress-nginx logs -l app.kubernetes.io/name=ingress-nginx --tail=5 | grep "WAF_DOMAIN"
# expected: 正常请求日志（WAF 放行后到 Nginx Ingress）
```

## 清理

### 控制面（tccli）

> **⚠️ 警告**：从 WAF 移除域名后，该域名的流量不再受 WAF 防护，可能暴露于攻击。确认有替代安全方案后再操作。

```bash
# 1. 删除 WAF 防护域名
tccli waf DeleteSpartaProtection --region ap-guangzhou \
    --Domain WAF_DOMAIN \
    --InstanceId WAF_INSTANCE_ID
# expected: exit 0

# 2. 验证域名已移除
tccli waf DescribeDomains --region ap-guangzhou \
    --InstanceId WAF_INSTANCE_ID
# expected: 列表中不包含 WAF_DOMAIN
```

### 数据面（需 VPN/IOA）

```bash
# CLB 和 Nginx Ingress 不受 WAF 删除影响，无需操作
# 验证 Nginx Ingress 直接访问正常
curl -X GET "http://EXTERNAL_IP/" -H "Host: WAF_DOMAIN"
# expected: 正常响应 200（不再经过 WAF）
```

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `AddSpartaProtection` 返回 `InvalidParameter.Domain` | 检查域名格式是否合法 | 域名格式错误（含非法字符或未备案） | 使用已备案的合规域名格式 |
| `AddSpartaProtection` 返回 `LimitExceeded` | `tccli waf DescribeDomains --region ap-guangzhou` 检查已接入数量 | WAF 域名数量配额达到上限（环境限制） | 删除不再使用的域名后重试，或升级 WAF 版本 |
| `AddSpartaProtection` 返回 `Domain already exists` | 同上的 DescribeDomains | 域名已在 WAF 中接入 | 使用 `tccli waf ModifyDomainClbMode` 修改现有配置而非重新添加 |

### 配置生效后功能异常

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| 接入 WAF 后网站不可达 | `tccli waf DescribeDomains --region ap-guangzhou` 检查状态 | WAF 回源到 CLB 失败，可能 `SrcList` 填错 | 确认 `SrcList` 为 CLB VIP（非域名），CLB VIP 可达 |
| WAF 未拦截测试攻击 | `curl -v http://EXTERNAL_IP/?id=1'OR'1` 检查响应 | WAF 规则未启用或 Host 头未匹配 WAF 域名 | 确认请求的 Host 头与 WAF 接入域名一致；在 WAF 控制台启用基础规则 |
| HTTPS 站点访问异常 | 检查 WAF Ports 配置是否包含 443 | WAF 未配置 HTTPS 端口，或证书未配置 | 添加 443 端口到 Ports 数组；如证书在 CLB 侧，`CertType=0` |
| 部分请求超时 | `kubectl -n ingress-nginx logs --tail=50` | WAF 与 CLB 间网络延迟增加（正常旁路行为） | 增加 Nginx `proxy-read-timeout` 并确认 WAF 节点与 CLB 同地域 |

## 下一步

- [可观测性集成](https://cloud.tencent.com/document/product/457/104861) — WAF 攻击日志可接入 CLS 统一分析
- [自定义负载均衡器](https://cloud.tencent.com/document/product/457/104858) — CLB 配置调优
- [高可用配置优化](https://cloud.tencent.com/document/product/457/104860) — 确保 WAF + Nginx Ingress 整体高可用

## 控制台替代

通过 [WAF 控制台](https://console.cloud.tencent.com/guanjia/tea-overview) 管理防护域名、查看攻击日志和配置防护规则。
