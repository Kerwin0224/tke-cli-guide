# 在 TKE 上使用 Pod 独立安全组（tccli）

> 对照官方：[在 TKE 上使用 Pod 独立安全组](https://cloud.tencent.com/document/product/457/124998) · page_id `124998`

## 概述

VPC-CNI 模式下，Pod 拥有独立 ENI 和 IP，可绑定专属安全组实现精细化的 Pod 级别网络访问控制。

> **kubectl 数据面不可达**：CAM拒绝公网端点(strategyId:240463971)，内网需VPN/IOA。以下 kubectl 命令为参考格式，需在内网/VPN 环境下执行。

## 前置条件

### 环境检查

```bash
tccli --version
# expected: tccli version >= 1.0.0

tccli tke DescribeClusters --region <Region> --ClusterIds '["CLUSTER_ID"]'
# expected: ClusterStatus "Running", VPC-CNI 模式

tccli vpc DescribeSecurityGroups --region <Region>
# expected: exit 0，返回安全组列表
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

## 控制台与 CLI 参数映射

| 控制台操作 | tccli/kubectl 命令 | 幂等 |
|-----------|-------------------|:--:|
| 创建安全组 | `tccli vpc CreateSecurityGroup` | 否 |
| 配置安全组规则 | `tccli vpc CreateSecurityGroupPolicies` | 否 |
| 为 Pod 指定安全组 | `kubectl apply -f pod-sg.yaml`（需 VPN/IOA） | 是 |
| 查看安全组绑定 | `tccli vpc DescribeNetworkInterfaces` | 是 |

## 操作步骤

### 步骤 1：创建 Pod 专用安全组

#### 选择依据

- 独立安全组 vs 节点安全组：节点安全组影响节点上所有 Pod，独立安全组粒度更细
- 安全组规则：默认拒绝所有入站，按需开放服务端口
- 适用场景：多租户隔离、PCI-DSS 合规、微服务网络策略强制

```bash
cat > create-sg.json <<'EOF'
{
    "GroupName": "pod-isolated-sg",
    "GroupDescription": "独立安全组-Pod级别网络隔离",
    "ProjectId": "0"
}
EOF
tccli vpc CreateSecurityGroup --region <Region> --cli-input-json file://create-sg.json
# expected: 返回 SecurityGroup.SgId
```

### 步骤 2：配置安全组入站规则

```bash
cat > sg-rules.json <<'EOF'
{
    "SecurityGroupId": "SECURITY_GROUP_ID",
    "SecurityGroupPolicySet": {
        "Ingress": [
            {
                "Protocol": "TCP",
                "Port": "8080",
                "CidrBlock": "10.0.0.0/8",
                "Action": "ACCEPT",
                "PolicyDescription": "允许内网访问应用端口"
            }
        ],
        "Egress": [
            {
                "Protocol": "ALL",
                "CidrBlock": "0.0.0.0/0",
                "Action": "ACCEPT",
                "PolicyDescription": "允许所有出站"
            }
        ]
    }
}
EOF
tccli vpc CreateSecurityGroupPolicies --region <Region> --cli-input-json file://sg-rules.json
# expected: exit 0
```

### 步骤 3：为 Pod 指定安全组

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-with-sg
  namespace: NAMESPACE
  annotations:
    tke.cloud.tencent.com/networks: "tke-route-eni"
    tke.cloud.tencent.com/security-group-id: "SECURITY_GROUP_ID"
spec:
  containers:
  - name: app
    image: nginx:latest
    ports:
    - containerPort: 8080
```

```bash
kubectl apply -f pod-with-sg.yaml
# expected: pod/app-with-sg created
```

## 验证

### 控制面（tccli）

```bash
tccli vpc DescribeSecurityGroups --region <Region> --SecurityGroupIds '["SECURITY_GROUP_ID"]'
# expected: 安全组含配置的入站/出站规则
```

### 数据面（需 VPN/IOA）

```bash
kubectl get pod app-with-sg -n NAMESPACE -o yaml | grep security-group
# expected: 含安全组注解

kubectl exec app-with-sg -n NAMESPACE -- netstat -tlnp
# expected: 8080 端口监听
```

```text
NAME  STATUS  AGE
...
```

## 清理

```bash
kubectl delete pod app-with-sg -n NAMESPACE

tccli vpc DeleteSecurityGroup --region <Region> --SecurityGroupId SECURITY_GROUP_ID
# expected: exit 0
```

## 排障

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| Pod 网络不通 | `kubectl exec POD_NAME -- curl -s --connect-timeout 5 TARGET_IP:PORT` | 安全组入站规则未放行所需端口 | 添加对应端口和来源 CIDR 的 ACCEPT 规则 |
| 安全组绑定不生效 | `kubectl describe pod POD_NAME` 查看注解 | 非 VPC-CNI 模式，Pod 无独立 ENI | 确认集群为 VPC-CNI 网络模式 |
| 安全组规则冲突 | `tccli vpc DescribeSecurityGroupPolicies --SgId SgId` | 默认规则与自定义规则优先级冲突 | 安全组规则按顺序匹配，调整规则顺序 |

## 下一步

- [使用 Network Policy 进行网络访问控制](../使用%20Network%20Policy%20进行网络访问控制/tccli%20操作.md) -- page_id `19793`
- [Pod 安全组](../../安全/Pod%20安全组/tccli%20操作.md) -- page_id `80587`

## 控制台替代

[容器服务控制台](https://console.cloud.tencent.com/tke2/cluster) -> 集群 -> 工作负载 -> Pod -> 安全组配置。
