# 部署 Agent 代码沙箱工具（tccli）

> 对照官方：[Agent 代码沙箱工具部署](https://cloud.tencent.com/document/product/457/123916) · page_id `123916`

## 概述

在 TKE 集群上使用 llm-sandbox 和 LangChain 构建可调用代码沙箱的 AI Agent。代码沙箱为 AI Agent 提供安全隔离的云端执行环境，支持 Python、Java、JavaScript、C++、Go、Ruby 等多种语言。

本页面的控制面操作通过 tccli 完成（集群管理、kubeconfig 获取），代码沙箱的运行时部署由 `llm-sandbox` 库自动管理 K8s Pod，无需手动创建 Deployment/Service。

> **kubectl 数据面不可达**：CAM 拒绝公网端点（strategyId:240463971），内网需 VPN/IOA。llm-sandbox 库通过 K8s API 创建沙箱 Pod，需要有效的 kubeconfig。

## 前置条件

- [环境准备](../../../环境准备.md)

### 环境检查

```bash
# 1. 检查 tccli 版本
tccli --version
# expected: tccli version >= 1.0.0

# 2. 检查当前凭据和地域
tccli configure list
# expected: secretId、secretKey、region 均已配置，region 为目标地域

# 3. 检查 CAM 权限 — 需要以下 Action 名
#    tke:DescribeClusters, tke:DescribeClusterKubeconfig
#    以及集群内的 K8s RBAC 权限（create/delete pods）
# 验证
tccli tke DescribeClusters --region <Region>
# expected: exit 0，返回集群列表

# 4. 检查 Python 版本（推荐 3.11.6+）
python3 --version
# expected: Python 3.11.6 或更高

# 5. 检查 pip 版本
pip --version
# expected: pip 23.3.1 或更高

# 如缺失 Python/pip，安装：
# yum install python3 python3-pip -y
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
# 6. 查询可用 TKE 集群
tccli tke DescribeClusters --region <Region>
# expected: 至少 1 个 Running 状态集群

# 7. 检查集群节点资源 — 沙箱 Pod 需要 CPU 和内存配额
tccli tke DescribeClusterInstances --region <Region> \
    --ClusterId CLUSTER_ID
# expected: 至少 2 个 Running 节点，推荐实例类型 SA5.LARGE8

# 8. 检查集群内网访问状态（llm-sandbox 需连 ApiServer）
tccli tke DescribeClusterEndpointStatus --region <Region> \
    --ClusterId CLUSTER_ID
# expected: 内网端点已开启
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

### 版本与规格选择

- 集群类型：`MANAGED_CLUSTER`（托管集群）
- 推荐 K8s 版本：>= 1.30.0
- 推荐节点类型：SA5.LARGE8（高性价比，满足沙箱计算需求）
- 推荐 Python 版本：3.11.6+

## 控制台与 CLI 参数映射

| 控制台操作 | tccli 命令 | 幂等 |
|-----------|-----------|:--:|
| 查看集群列表 | `DescribeClusters` | 是 |
| 查看集群节点 | `DescribeClusterInstances` | 是 |
| 获取 kubeconfig | `DescribeClusterKubeconfig` | 是 |
| 查看集群端点状态 | `DescribeClusterEndpointStatus` | 是 |

> 代码沙箱的 Pod 创建/销毁由 `llm-sandbox` Python 库通过 K8s API 自动管理，不经过 tccli。以下"操作步骤"聚焦于 Python 客户端侧配置。

## 操作步骤

### 步骤 1：确认集群并获取 kubeconfig

```bash
tccli tke DescribeClusters --region <Region> \
    --ClusterIds '["CLUSTER_ID"]'
# expected: ClusterStatus "Running"
```

**预期输出**（以真实集群 `cls-xxxxxxxx` 为例）：

```json
{
    "TotalCount": 1,
    "Clusters": [
        {
            "ClusterId": "cls-xxxxxxxx",
            "ClusterName": "my-tke-cluster",
            "ClusterStatus": "Running",
            "ClusterVersion": "1.30.0",
            "ClusterType": "MANAGED_CLUSTER"
        }
    ],
    "RequestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

获取 kubeconfig 供 llm-sandbox 库使用：

```bash
tccli tke DescribeClusterKubeconfig --region <Region> --ClusterId CLUSTER_ID \
    | jq -r '.Kubeconfig' > ~/.kube/config
# expected: kubeconfig 写入成功
```

```json
{
  "Kubeconfig": "<Kubeconfig>",
  "RequestId": "<RequestId>"
}
```

| 占位符 | 说明 | 约束 | 获取方式 |
|--------|------|------|---------|
| `REGION` | 地域 | 如 `ap-guangzhou` | `tccli configure list` |
| `CLUSTER_ID` | 集群 ID | 格式 `cls-xxxxxxxx` | `tccli tke DescribeClusters` |

### 步骤 2：安装 llm-sandbox 和 LangChain 依赖

```bash
pip install 'llm-sandbox[k8s]' langchain langchain_openai
# expected: 所有依赖安装成功，无报错
```

如 pip 找不到 `llm-sandbox` 包，检查 Python 版本：

```bash
python3 --version
# 需要 >= 3.11，否则升级：
# yum install python3.11 python3.11-pip -y
```

### 步骤 3：编写 Agent 代码

#### 选择依据

- **SandboxBackend**：选择 `SandboxBackend.KUBERNETES`，利用 TKE 集群作为沙箱后端。每个代码执行创建独立 Pod，完成后自动回收。
- **namespace**：默认使用 `default` namespace。生产环境建议使用独立 namespace 隔离沙箱 Pod。
- **verbose**：`False` 表示静默模式，不输出调试日志。排查问题时改为 `True`。

**最小可运行脚本**（接任意 OpenAI 兼容 API）：

`agent-sandbox-minimal.py`：

```python
import logging
from langchain import hub
from langchain.agents import AgentExecutor, create_tool_calling_agent
from langchain_core.tools import tool
from langchain_openai import ChatOpenAI
from llm_sandbox import SandboxSession, SandboxBackend

logging.basicConfig(level=logging.INFO, format="%(message)s")
logger = logging.getLogger(__name__)


@tool
def run_code(lang: str, code: str, libraries: list | None = None) -> str:
    """Run code in a sandboxed environment.
    :param lang: The language of the code, must be one of
        ['python', 'java', 'javascript', 'cpp', 'go', 'ruby'].
    :param code: The code to run.
    :param libraries: The libraries to use, it is optional.
    :return: The output of the code.
    """
    with SandboxSession(
        lang=lang,
        backend=SandboxBackend.KUBERNETES,
        kube_namespace="default",
        verbose=False
    ) as session:
        return session.run(code, libraries).stdout


if __name__ == "__main__":
    llm = ChatOpenAI(
        model="MODEL_NAME",
        temperature=0,
        max_retries=2,
        api_key="API_KEY",
        base_url="BASE_URL",
    )
    prompt = hub.pull("hwchase17/openai-functions-agent")
    tools = [run_code]

    agent = create_tool_calling_agent(llm, tools, prompt)
    agent_executor = AgentExecutor(agent=agent, tools=tools, verbose=True)

    output = agent_executor.invoke({
        "input": "Write python code to calculate Pi number"
                 " by Monte Carlo method then run it."
    })
    logger.info("Agent: %s", output)
```

| 占位符 | 说明 | 约束 | 获取方式 |
|--------|------|------|---------|
| `MODEL_NAME` | LLM 模型名称 | 如 `deepseek-chat`、`qwen3-coder` | 从模型提供商获取 |
| `API_KEY` | LLM API 密钥 | 必填 | 从模型提供商获取 |
| `BASE_URL` | LLM API 端点 | 如 `https://api.deepseek.com` | 从模型提供商获取 |

> **注意**：`MODEL_NAME`、`API_KEY`、`BASE_URL` 可以是 Deepseek、OpenAI、或 TKE 上自托管的 vllm 推理服务（参见 [Embedding 模型部署](../../AI%20模型部署/Embedding%20模型部署/tccli%20操作.md)）。

### 步骤 4：运行 Agent

```bash
python3 agent-sandbox-minimal.py
# expected: Agent 调用代码沙箱执行 Python 代码，返回计算结果（Monte Carlo Pi 约 3.14）
```

Agent 执行流程：
1. LangChain Agent 接收用户输入
2. LLM 判断需要执行代码，生成 Python 代码
3. `run_code` 工具被调用，llm-sandbox 在 TKE 集群 `default` namespace 创建临时 Pod
4. Pod 执行代码并返回 stdout
5. Pod 自动销毁

## 验证

### 控制面（tccli）

```bash
# 1. 确认集群状态
tccli tke DescribeClusters --region <Region> --ClusterIds '["CLUSTER_ID"]'
# expected: ClusterStatus "Running"

# 2. 确认节点资源充足
tccli tke DescribeClusterInstances --region <Region> --ClusterId CLUSTER_ID
# expected: 节点状态 running，资源充足
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

### 数据面（需 VPN/IOA）

```bash
# 3. 运行脚本时，观察沙箱 Pod 创建
kubectl get pods -n default -w
# expected: 短暂出现 llm-sandbox-* Pod，状态 Running→Completed

# 4. 验证 Agent 输出
# expected: Agent 日志显示 Monte Carlo Pi 计算结果（约 3.142396）
```

```text
NAME  STATUS  AGE
...
```

### 测试用例

| 测试 | 输入 | 预期输出 |
|------|------|---------|
| 蒙特卡洛求 Pi | `"Write python code to calculate Pi number by Monte Carlo method then run it."` | 约 3.142396 |
| 阶乘计算 | `"Write python code to calculate the factorial of a number then run it."` | factorial of 5 is 120 |
| 斐波那契 | `"Write python code to calculate the Fibonacci sequence then run it."` | `[0,1,1,2,3,5,8,13,21,34]` |
| 求和 | `"Calculate the sum of the first 10000 numbers."` | 50,005,000 |

## 清理

> **警告**：llm-sandbox 的沙箱 Pod 在代码执行完成后通常会自动销毁（Completed 状态）。手动清理仅需删除残留的 Completed Pod，不影响运行中的业务。

```bash
# 1. 查看残留沙箱 Pod（须 VPN/IOA）
kubectl get pods -n default -l app=llm-sandbox
# expected: 可能返回空列表或 Completed 状态的 Pod

# 2. 删除所有 Completed 沙箱 Pod（须 VPN/IOA）
kubectl delete pods -n default --field-selector=status.phase=Succeeded
# expected: 残留 Pod 已清理

# 3. 验证已清理（须 VPN/IOA）
kubectl get pods -n default | grep llm-sandbox
# expected: 无输出
```

```text
NAME  STATUS  AGE
...
```

> 集群本身无需删除（资源归属集群，不作为本页清理范围）。

## 排障

### 命令返回错误

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| `pip install` 找不到 `llm-sandbox` | `python3 --version` | Python 版本过低（< 3.11） | 升级至 Python 3.11+：`yum install python3.11 python3.11-pip -y` |
| `hub.pull` 失败 | `curl -I https://smith.langchain.com` 测试连通性 | 网络不可达 LangChain Hub | 配置 HTTP 代理或使用离线 prompt 模板 |
| 脚本报 permission error | `ls -la ~/.kube/config` 检查 kubeconfig 文件 | kubeconfig 未正确配置或权限不足 | `tccli tke DescribeClusterKubeconfig --region <Region> --ClusterId CLUSTER_ID \| jq -r '.Kubeconfig' > ~/.kube/config` 并确保 chmod 600 |
| `InsecureRequestWarning` | 检查 kubeconfig 中 ApiServer URL | 集群内网端点未开启（公网被 CAM 拒绝，strategyId:240463971） | 通过 VPN/IOA 连接集群内网，或开启集群内网访问端点 |

### 脚本运行成功但功能异常

| 现象 | 诊断 | 根因 | 修复 |
|------|------|------|------|
| 沙箱 Pod 一直 Pending | `kubectl describe pod -l app=llm-sandbox`（须 VPN/IOA） | 集群资源不足或镜像拉取慢 | 增加节点或降低资源请求；等待镜像拉取完成 |
| 沙箱 Pod CrashLoopBackOff | `kubectl logs POD_NAME`（须 VPN/IOA） | 沙箱内代码执行异常或镜像有问题 | 检查代码逻辑；确认镜像可正常拉取 |
| Agent 返回空结果 | `verbose=True` 重新运行脚本 | LLM API 不可达或 API Key 无效 | 检查 `BASE_URL` 可达性和 `API_KEY` 有效性 |
| 大量 Completed Pod 堆积 | `kubectl get pods \| grep llm-sandbox \| wc -l` | 历史沙箱 Pod 未自动清理 | `kubectl delete pods --field-selector=status.phase=Succeeded -n default` |

## 下一步

- [Embedding 模型部署](../../AI%20模型部署/Embedding%20模型部署/tccli%20操作.md) -- page_id `124002`（TKE 上自托管 LLM 推理服务）
- [AI 网关部署](../../AI%20模型部署/AI%20网关部署/tccli%20操作.md) -- page_id `123918`
- [MCP Server 托管](../../MCP%20Server%20应用/MCP%20Server%20托管/tccli%20操作.md) -- page_id `124005`
- [LangChain 官方文档](https://python.langchain.com/)
- [llm-sandbox GitHub](https://github.com/tyler-suard/llm-sandbox)

## 控制台替代

[容器服务控制台](https://console.cloud.tencent.com/tke2/cluster)
