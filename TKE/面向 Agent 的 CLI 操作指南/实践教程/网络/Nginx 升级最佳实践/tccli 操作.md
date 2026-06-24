# Nginx 升级最佳实践（tccli）

> 对照官方：[Nginx 升级最佳实践](https://cloud.tencent.com/document/product/457/90454) · page_id `90454`

## 概述

Nginx Ingress Controller 版本升级涉及 CRD 更新、镜像切换、配置兼容性检查。推荐使用金丝雀（Canary）部署策略逐步切换流量，降低升级风险。

## 前置条件

- [环境准备](../../../环境准备.md)
- 当前 NginxIngress 组件已安装
- 备份当前 Ingress 配置：`kubectl get ingress -A -o yaml > ingress-backup.yaml`

## 控制台与 CLI 参数映射

| 控制台操作 | CLI | 幂等 |
|-----------|-----|------|
| 查看当前版本 | `tccli tke DescribeAddon --AddonName nginx-ingress` | 是 |
| 升级组件 | `tccli tke UpdateAddon --cli-input-json file://update.json` | 否 |
| 金丝雀部署 | `kubectl apply -f canary-ingress.yaml` | 是 |
| 回滚版本 | `tccli tke UpdateAddon` 指定旧版本 | 否 |

## 操作步骤

### 1. 升级前检查

```bash
tccli tke DescribeAddon --region ap-guangzhou --cli-input-json file://describe-addon.json
```

```json
{"ClusterId": "<ClusterId>", "AddonName": "nginx-ingress"}
```

检查当前版本及可用版本列表。

### 2. 备份现有配置

```bash
kubectl get ingress -A -o yaml > ingress-backup-$(date +%Y%m%d).yaml
kubectl get configmap -n kube-system nginx-configuration -o yaml > nginx-config-backup.yaml
```

```text
NAME  STATUS  AGE
...
```

### 3. 执行升级

```bash
tccli tke UpdateAddon --region ap-guangzhou --cli-input-json file://update-addon.json
```

```json
{
    "ClusterId": "<ClusterId>",
    "AddonName": "nginx-ingress",
    "AddonVersion": "<NewVersion>",
    "UpdateStrategy": "RollingUpdate"
}
```

```output
{"RequestId": "..."}
```

### 4. 金丝雀验证（推荐）

部署新版本 Ingress Controller 并只处理带特定 Header 的请求：

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: canary-test
  annotations:
    kubernetes.io/ingress.class: nginx
    nginx.ingress.kubernetes.io/canary: "true"
    nginx.ingress.kubernetes.io/canary-by-header: "X-Canary"
    nginx.ingress.kubernetes.io/canary-by-header-value: "true"
spec:
  rules:
  - host: app.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: app-service-v2
            port:
              number: 80
```

验证金丝雀版本：

```bash
curl -H "X-Canary: true" http://<Ingress-IP>/
```

### 5. 全量切换

确认金丝雀版本正常后，将主 Service 指向新版本 Deployment：

```bash
kubectl patch svc nginx-ingress-controller -n kube-system -p '{"spec":{"selector":{"app.kubernetes.io/version":"v2"}}}'
```

## 验证

```bash
tccli tke DescribeAddon --region ap-guangzhou --cli-input-json file://describe-addon.json --filter "AddonVersion"
kubectl get pods -n kube-system -l app=nginx-ingress -o jsonpath='{.items[0].spec.containers[0].image}'
```

```json
{
  "Addons": [],
  "AddonName": "<AddonName>",
  "AddonVersion": "<AddonVersion>",
  "RawValues": "<RawValues>",
  "Phase": "<Phase>",
  "Reason": "<Reason>",
  "CreateTime": "<CreateTime>",
  "RequestId": "<RequestId>"
}
```

## 清理

升级成功后清理备份文件及金丝雀 Ingress。

## 排障

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `kubectl` 命令超时或连接被拒绝 | `tccli tke DescribeClusters --region ap-guangzhou --ClusterIds '["cls-xxxxxxxx"]'`（控制面返回 Running） | CAM 策略 strategyId:240463971 拒绝公网端点，数据面需内网访问 | 通过 VPN/IOA 接入 VPC 内网后执行 kubectl；或使用控制台替代 |
| 升级后 Ingress 不生效 | `kubectl get crd ingressroutes.traefik.io` 或 `kubectl get ingressclass` | CRD 版本未随镜像更新 | 更新 CRD：`kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/.../deploy/static/provider/cloud/deploy.yaml` |
| Connection Refused | `kubectl get pods -n kube-system -l app=nginx-ingress` | 新版镜像不兼容或启动失败 | 回滚到旧版本 |
| 金丝雀 Header 不生效 | `kubectl describe ingress <canary-name>` 查看 annotations | `canary-by-header` 配置错误 | 确认 `canary-by-header` 注解键名与值正确 |

## 下一步

- [Nginx Ingress 最佳实践](../Nginx%20Ingress%20最佳实践/tccli%20操作.md)
- [Nginx Ingress 高并发实践](../Nginx%20Ingress%20高并发实践/tccli%20操作.md)

## 控制台替代

控制台：集群 → 组件管理 → NginxIngress → 升级。
