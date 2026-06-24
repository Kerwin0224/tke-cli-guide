# 非固定IP模式使用说明

> 对照官方：[非固定IP模式使用说明](https://cloud.tencent.com/document/product/457/64940) · page_id `64940`

## 概述

非固定IP模式是 VPC-CNI 网络模式的默认行为。创建 VPC-CNI 集群时不勾选「固定 Pod IP」即启用此模式。Pod IP 不固定，重建后可能分配新 IP。适用于无状态服务、批量计算等不依赖固定 IP 的场景。

核心机制：每个节点维护一个弹性伸缩的独占 ENI/IP 预热池，绑定数量保持在 `Pod数 + 最小预绑定` 与 `Pod数 + 最大预绑定` 之间。节点扩容时预绑定的 IP 可立即分配给新 Pod，避免等待 ENI 挂载。

## 前置条件

- [环境准备](../../../../../../环境准备.md)
- 已 [连接集群](../../../集群管理/连接集群/tccli%20操作.md)
- 集群网络模式为 VPC-CNI（创建时通过 `tccli tke CreateCluster` 指定 `NetworkType` 为 `VPC-CNI` 且不启用固定 IP）

验证集群网络模式：

```bash
tccli tke DescribeClusters --ClusterIds '["<ClusterId>"]' --region <Region>
# expected: NetworkType 为 VPC-CNI
```

```json
{
  "TotalCount": 1,
  "Clusters": [
    {
      "ClusterId": "cls-example",
      "ClusterName": "tke-demo",
      "ClusterVersion": "1.30.0",
      "ClusterStatus": "Running",
      "ClusterType": "MANAGED_CLUSTER",
      "ClusterNetworkSettings": {
        "VpcId": "vpc-example",
        "ClusterCIDR": "172.16.0.0/16",
        "NetworkType": "VPC-CNI",
        "VpcCniType": "tke-route-eni"
      }
    }
  ],
  "RequestId": "req-example-001"
}
```

验证 eniipamd 组件已安装：

```bash
tccli tke DescribeAddon --ClusterId <ClusterId> --AddonName eniipamd --region <Region>
# expected: Addons 列表非空，Phase 为 Succeeded
```

```json
{
  "Addons": [
    {
      "AddonName": "eniipamd",
      "AddonVersion": "3.5.0",
      "Phase": "Succeeded",
      "RawValues": "..."
    }
  ],
  "RequestId": "req-example-002"
}
```

## 控制台与 CLI 参数映射

| 控制台操作 | CLI | 幂等 |
|-----------|-----|------|
| 查看集群网络模式 | `tccli tke DescribeClusters` | 是 |
| 查看 eniipamd 组件状态 | `tccli tke DescribeAddon --AddonName eniipamd` | 是 |
| 查看 eniipamd 当前配置 | `tccli tke DescribeAddonValues --AddonName eniipamd` | 是 |
| 更新 eniipamd 组件配置（快释放/默认预绑定） | `tccli tke UpdateAddon --AddonName eniipamd` | 是（merge 策略） |
| 查看节点 NEC 对象 | `kubectl get nec` | 是 |
| 设置节点预绑定数量 | `kubectl annotate nec <节点名>` | 是（--overwrite） |
| 设置节点最大绑定数量 | `kubectl annotate nec <节点名>` | 是（--overwrite） |
| 查看节点详情（含注解） | `kubectl describe node <节点名>` | 是 |

## 操作步骤

### 1. 查看当前 eniipamd 配置

获取 eniipamd 组件的当前参数和默认值：

```bash
tccli tke DescribeAddonValues --ClusterId <ClusterId> --AddonName eniipamd --region <Region>
```

```json
{
  "Values": "{\"vpcCni\":{...}}",
  "DefaultValues": "{\"vpcCni\":{\"routeEni\":{\"ipMinWarmTarget\":5,\"ipMaxWarmTarget\":5},\"directEni\":{\"eniMinWarmTarget\":1,\"eniMaxWarmTarget\":1}}}",
  "RequestId": "req-example-003"
}
```

`Values` 中记录了当前已设置的参数（含 chart 默认值），`DefaultValues` 是 chart 原始默认值。更新时以 `Values` 中的键为基础进行 merge。

### 2. 启用快释放

快释放功能在每次检查周期（约 2 分钟）一次性释放所有超出预绑定上限的空闲 ENI/IP。默认关闭时每次仅释放 1 个。

#### 控制面（tccli，适用于 eniipamd >= v3.5.0）

通过 `UpdateAddon` 更新 eniipamd 配置。先查看当前 `Values`，提取后添加快释放字段、base64 编码后提交：

```bash
tccli tke UpdateAddon \
  --ClusterId <ClusterId> \
  --AddonName eniipamd \
  --RawValues "eyJ2cGNDbmkiOnsicm91dGVFbmkiOnsiZW5hYmxlUXVpY2tSZWxlYXNlIjp0cnVlfX19" \
  --UpdateStrategy merge \
  --region <Region>
```

```json
{
  "RequestId": "req-example-004"
}
```

> `RawValues` 的 base64 原文：`{"vpcCni":{"routeEni":{"enableQuickRelease":true}}}` — 共享 ENI 模式开启快释放。独占 ENI 模式下将 `routeEni` 替换为 `directEni`。

#### 数据面（kubectl，适用于 eniipamd < v3.5.0）

编辑 tke-eni-agent DaemonSet 添加快释放参数：

```bash
kubectl edit ds tke-eni-agent -n kube-system
```

在 `spec.template.spec.containers[0].args` 中添加：

```text
--enable-quick-release
```

编辑完成后 DaemonSet 触发滚动更新，现有 Pod 不受影响但新 Pod 启动可能短暂延迟。建议在低流量时段操作。

### 3. 指定某节点预绑定数量

通过 NEC CRD 注解为单个节点设置预绑定 IP/ENI 范围。

获取节点列表：

```bash
kubectl get nodes
```

```text
NAME           STATUS   ROLES    AGE   VERSION
10.0.0.1       Ready    <none>   5d    v1.30.0-tke.2
10.0.0.2       Ready    <none>   5d    v1.30.0-tke.2
```

查看节点当前的 NEC 注解：

```bash
kubectl get nec 10.0.0.1 -o yaml
```

```yaml
apiVersion: networking.tke.cloud.tencent.com/v1
kind: NodeENIConfig
metadata:
  name: 10.0.0.1
  annotations:
    tke.cloud.tencent.com/route-eni-ip-min-warm-target: "5"
    tke.cloud.tencent.com/route-eni-ip-max-warm-target: "5"
spec:
  ...
```

设置预绑定范围（共享 ENI 模式）：

```bash
kubectl annotate nec 10.0.0.1 \
  "tke.cloud.tencent.com/route-eni-ip-min-warm-target"="1" \
  --overwrite

kubectl annotate nec 10.0.0.1 \
  "tke.cloud.tencent.com/route-eni-ip-max-warm-target"="3" \
  --overwrite
```

```text
nodeenicconfig.networking.tke.cloud.tencent.com/10.0.0.1 annotated
```

独占 ENI 模式下使用 `tke.cloud.tencent.com/direct-eni-min-warm-target` 和 `tke.cloud.tencent.com/direct-eni-max-warm-target`。

> 两个注解必须同时存在且满足 `0 <= 最小 <= 最大`，否则修改失败。

### 4. 指定某节点最大绑定数量

限制单个节点的最大 ENI 或 IP 绑定数。修改值须大于等于当前已用量，否则失败。

共享 ENI 模式：

```bash
kubectl annotate nec 10.0.0.1 \
  "tke.cloud.tencent.com/route-eni-max-attach"="2" \
  --overwrite

kubectl annotate nec 10.0.0.1 \
  "tke.cloud.tencent.com/max-ip-per-route-eni"="10" \
  --overwrite
```

独占 ENI 模式：

```bash
kubectl annotate nec 10.0.0.1 \
  "tke.cloud.tencent.com/direct-eni-max-attach"="4" \
  --overwrite
```

修改后若当前绑定数超过新上限，系统将逐步释放多余 ENI/IP 直至符合上限。

### 5. 配置默认预绑定数量

默认预绑定数量仅影响**新加入集群的节点**。存量节点需用第 3 节的方法逐个设置。

#### 控制面（tccli，适用于 eniipamd >= v3.5.0）

通过 `UpdateAddon` 配置默认值。创建 JSON 配置文件后提交：

`eniipamd-warm-target.json`：

```json
{
  "vpcCni": {
    "routeEni": {
      "ipMinWarmTarget": 3,
      "ipMaxWarmTarget": 3
    },
    "directEni": {
      "eniMinWarmTarget": 1,
      "eniMaxWarmTarget": 1
    }
  }
}
```

将文件内容 base64 编码后通过 `--cli-input-json` 提交：

```bash
tccli tke UpdateAddon --cli-input-json file://update-ipamd-warm-target.json --region <Region>
```

`update-ipamd-warm-target.json`：

```json
{
  "ClusterId": "<ClusterId>",
  "AddonName": "eniipamd",
  "RawValues": "eyJ2cGNDbmkiOnsicm91dGVFbmkiOnsiaXBNaW5XYXJtVGFyZ2V0IjozLCJpcE1heFdhcm1UYXJnZXQiOjN9LCJkaXJlY3RFbmkiOnsiZW5pTWluV2FybVRhcmdldCI6MSwiZW5pTWF4V2FybVRhcmdldCI6MX19fQ==",
  "UpdateStrategy": "merge"
}
```

```json
{
  "RequestId": "req-example-005"
}
```

> `RawValues` 的 base64 原文同 `eniipamd-warm-target.json` 内容。

#### 数据面（kubectl，适用于 eniipamd < v3.5.0）

编辑 eniipamd Deployment：

```bash
kubectl edit deploy tke-eni-ipamd -n kube-system
```

在 `spec.template.spec.containers[0].args` 中添加：

```text
--ip-min-warm-target=3
--ip-max-warm-target=3
--eni-min-warm-target=1
--eni-max-warm-target=1
```

编辑触发滚动更新，建议低流量时段操作。

### 6. 配置预热IP自动同步到存量节点（eniipamd >= v3.10.0）

启用后，修改集群默认预绑定配置时自动同步到所有存量节点，无需手动逐个设置。

#### 控制面（tccli）

通过 `UpdateAddon` 的 ConfigMap 键开启：

`eniipamd-auto-sync.json`：

```json
{
  "vpcCni": {
    "routeEni": {
      "enableIpWarmTargetAutoSync": true
    }
  }
}
```

将上述 JSON base64 编码后通过 `--cli-input-json` 提交：

```bash
tccli tke UpdateAddon --cli-input-json file://update-auto-sync.json --region <Region>
```

`update-auto-sync.json`：

```json
{
  "ClusterId": "<ClusterId>",
  "AddonName": "eniipamd",
  "RawValues": "eyJ2cGNDbmkiOnsicm91dGVFbmkiOnsiZW5hYmxlSXBXYXJtVGFyZ2V0QXV0b1N5bmMiOnRydWV9fX0=",
  "UpdateStrategy": "merge"
}
```

```json
{
  "RequestId": "req-example-006"
}
```

> `RawValues` 的 base64 原文：`{"vpcCni":{"routeEni":{"enableIpWarmTargetAutoSync":true}}}`。

## 验证

### 控制面（tccli）

验证 eniipamd 组件状态及配置：

```bash
tccli tke DescribeAddon --ClusterId <ClusterId> --AddonName eniipamd --region <Region>
```

```json
{
  "Addons": [
    {
      "AddonName": "eniipamd",
      "AddonVersion": "3.5.0",
      "Phase": "Succeeded",
      "RawValues": "..."
    }
  ],
  "RequestId": "req-example-007"
}
```

确认 `Phase` 为 `Succeeded`，表示配置已生效。

验证更新后的配置参数：

```bash
tccli tke DescribeAddonValues --ClusterId <ClusterId> --AddonName eniipamd --region <Region>
```

```json
{
  "Values": "<Values>",
  "DefaultValues": "<DefaultValues>",
  "RequestId": "<RequestId>"
}
```

检查 `Values` 中包含已修改的键（如 `enableQuickRelease`、`ipMinWarmTarget` 等）。

### 数据面（kubectl）

验证节点 NEC 注解：

```bash
kubectl get nec 10.0.0.1 -o jsonpath='{.metadata.annotations}' | python3 -m json.tool
```

```json
{
  "tke.cloud.tencent.com/route-eni-ip-min-warm-target": "1",
  "tke.cloud.tencent.com/route-eni-ip-max-warm-target": "3",
  "tke.cloud.tencent.com/route-eni-max-attach": "2"
}
```

验证快释放参数在 DaemonSet 中生效：

```bash
kubectl get ds tke-eni-agent -n kube-system -o jsonpath='{.spec.template.spec.containers[0].args}'
```

```text
["--enable-quick-release","--v=4"]
```

验证 eniipamd Deployment 默认预绑定参数：

```bash
kubectl get deploy tke-eni-ipamd -n kube-system -o jsonpath='{.spec.template.spec.containers[0].args}'
```

```text
["--ip-min-warm-target=3","--ip-max-warm-target=3","--v=4"]
```

## 清理

### 控制面（tccli）

恢复 eniipamd 默认配置（移除自定义预热参数）。通过 `UpdateAddon` 重新提交不含自定义键的配置。注意：此操作会清除所有自定义 eniipamd 设置，非仅预热相关配置。

```bash
tccli tke UpdateAddon --cli-input-json file://reset-ipamd-warm.json --region <Region>
```

`reset-ipamd-warm.json`：

```json
{
  "ClusterId": "<ClusterId>",
  "AddonName": "eniipamd",
  "RawValues": "eyJ2cGNDbmkiOnsicm91dGVFbmkiOnt9LCJkaXJlY3RFbmkiOnt9fX0=",
  "UpdateStrategy": "replace"
}
```

> `RawValues` 的 base64 原文：`{"vpcCni":{"routeEni":{},"directEni":{}}}`。使用 `replace` 策略完全替换配置，而非 merge。此操作将移除所有自定义预热/快释放/自动同步配置，恢复 chart 默认值。

```json
{
  "RequestId": "req-example-008"
}
```

> 警告：`replace` 策略会覆盖 eniipamd 的全部配置。如对安全组、子网等其他键做过修改，这些修改也会一并丢失。建议先用 `DescribeAddonValues` 获取当前 `Values`，仅移除与预热相关的键后重新提交。

### 数据面（kubectl）

移除节点的 NEC 预绑定注解（恢复为集群默认值）：

```bash
kubectl annotate nec 10.0.0.1 \
  "tke.cloud.tencent.com/route-eni-ip-min-warm-target"- \
  "tke.cloud.tencent.com/route-eni-ip-max-warm-target"- \
  "tke.cloud.tencent.com/route-eni-max-attach"-
```

移除 DaemonSet/Deployment 中手动添加的参数（如有）。通过 `kubectl edit` 删除 `--enable-quick-release`、`--ip-min-warm-target` 等自定义参数行后保存。

> 无计费影响：非固定IP模式本身不产生额外费用。ENI 和 IP 资源费用按 [VPC 计费说明](https://cloud.tencent.com/document/product/215/20096) 收取。

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `tccli tke UpdateAddon` 返回 `UnknownParameter` | 检查 `RawValues` 中的键名是否正确 | eniipamd 不支持该配置键或键名拼写错误 | 用 `DescribeAddonValues` 获取支持的键列表，核对键名与层级 |
| `kubectl annotate nec` 返回拒绝 | `kubectl describe nec <节点名>` 查看当前注解 | 缺少 `--overwrite` 标志，或注解已存在冲突 | 添加 `--overwrite` 重新执行 |
| `kubectl annotate nec` 设置预绑定范围失败 | 验证是否同时设置了最小和最大注解 | 两个注解必须成对存在，或值不满足 `0 <= 最小 <= 最大` | 确保同时设置且满足约束 |
| `kubectl annotate nec` 设置最大绑定数失败 | `kubectl get nec <节点名> -o yaml` 检查当前绑定数 | 新上限小于当前已用 ENI/IP 数 | 设置更大的上限值 |
| `kubectl edit ds tke-eni-agent` 后 Pod 未重启 | `kubectl get pods -n kube-system -l app=tke-eni-agent` 查看 Pod 状态 | DaemonSet 滚动更新策略为 OnDelete 或存在阻止更新的条件 | 手动删除 Pod 触发重建，或确认更新策略 |
| `tccli tke DescribeClusters` 未包含 `NetworkType` 字段 | API 返回的字段取决于集群创建时的参数 | 旧版集群可能未在此 API 中填充 `NetworkType` | 通过控制台或 `kubectl get nodes -o wide` 查看节点 IP 段推断 |
| kubectl 命令不可用（公网端点被拒绝） | 执行 `kubectl cluster-info` 验证连通性 | CAM 策略 `tke:clusterExtranetEndpoint` 被 deny，公网端点未开启 | 在 VPC 内 CVM 或通过 VPN/专线执行 kubectl 命令；或联系管理员开启公网端点 |

### 配置生效但状态异常

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| 修改默认预绑定后新节点未生效 | `kubectl get ds tke-eni-agent -n kube-system -o yaml` 检查 eniipamd 版本 | eniipamd < v3.10.0 默认预绑定值不影响存量节点；或仅修改了 NEC 注解但漏了 eniipamd 参数 | 升级 eniipamd 到 v3.10.0+ 并开启自动同步；或检查 DaemonSet/Deployment 参数 |
| 快释放开启后 Pod 重建变慢 | `kubectl describe pod <Pod名>` 查看 Events | 刚释放的 IP 尚未归还 VPC，新 Pod 创建时无可用 IP 需重新申请 | 适当调高最小预绑定值作为缓冲；非高频重建场景可忽略 |
| 预热池 IP 频繁不足 | 查看节点 NEC 注解的预绑定设置和当前 ENI IP 使用情况 | 最小/最大预绑定设置过低，突发扩容超过池容量 | 适当调高预绑定范围 |
| 节点预热 IP 占用过多导致 IP 浪费 | 查看节点上实际运行的 Pod 数量 vs 预绑定 IP 数量 | 最大预绑定值设置过高 | 调低最大预绑定值 |
| 预绑定数量设置为 0 后修改失败 | 阅读错误信息确认约束 | 非固定IP模式不支持全按需分配（预绑定值不能为 0） | 设置最小预绑定 >= 1 |
| `enableIpWarmTargetAutoSync` 开启后存量节点未更新 | `kubectl get nec -o yaml` 批量检查节点注解 | 组件版本 < v3.10.0 不支持此功能，或 ConfigMap 键名有误 | 确认组件版本 >= v3.10.0；检查键名 `vpc-cni.route-eni.enable-ip-warm-target-auto-sync` |

## 下一步

- [固定IP模式使用说明](../固定IP模式使用说明/tccli%20操作.md)
- [VPC-CNI 模式介绍](https://cloud.tencent.com/document/product/457/50355)
- [Pod 网络无法访问排查处理](../../../../故障处理/Pod%20网络无法访问排查处理/tccli%20操作.md)
- [容器服务安全组设置](../../../../安全和稳定性/容器服务安全组设置/tccli%20操作.md)

## 控制台替代

控制台：集群 → 组件管理 → 找到 `eniipamd` → 点击「更新配置」，在弹窗中修改快释放开关、共享/独占 ENI 模式预绑定默认值、预热 IP 自动同步开关后点击「完成」。节点级别的预绑定/最大绑定数需通过 kubectl 设置 NEC 注解，控制台暂不支持。
