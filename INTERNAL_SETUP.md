# 内部部署 LLM 配置指南

本项目已针对公司内部环境进行深度改造，移除了所有外部 LLM 供应商依赖，并禁用了外网访问功能。本文档将指导您如何配置内部模型服务。

## 1. 默认行为

在未进行任何配置的情况下，OpenCode 将默认尝试连接 DeepSeek 接口的模型服务：

- **默认地址 (Base URL)**: `https://api.deepseek.com`
- **默认密钥 (API Key)**: `sk-internal`

如果您的本地环境已运行 Ollama 等服务且监听该端口，直接启动 OpenCode 即可使用。

## 2. 自定义配置

如果您的内部模型部署在其他服务器，或需要特定的 API Key，可以通过以下两种方式进行配置。

### 方式一：配置文件 (推荐)

您可以修改项目根目录下的 `opencode.json`，或者在用户主目录下的全局配置 `~/.config/opencode/opencode.json` 中添加以下内容：

```json
{
  "provider": {
    "internal": {
      "options": {
        "baseURL": "http://192.168.1.100:8080/v1",
        "apiKey": "your-company-secret-key"
      }
    }
  }
}
```

> **注意**: 请将 `baseURL` 替换为您实际的内部模型 API 地址（通常以 `/v1` 结尾）。

### 方式二：环境变量

在启动 OpenCode 之前，您也可以通过设置环境变量来临时覆盖配置：

- **`INTERNAL_API_BASE`**: 设置模型 API 地址
- **`INTERNAL_API_KEY`**: 设置 API 密钥

**Linux / macOS 示例:**

```bash
export INTERNAL_API_BASE="http://ai-gateway.internal/v1"
export INTERNAL_API_KEY="sk-prod-key"
opencode
```

**Windows PowerShell 示例:**

```powershell
$env:INTERNAL_API_BASE="http://ai-gateway.internal/v1"
$env:INTERNAL_API_KEY="sk-prod-key"
opencode
```

## 3. 功能限制说明

为适应内网安全要求，本版本包含以下硬性限制：

*   **仅限内部模型**: 只能使用 `internal` provider，所有外部供应商（OpenAI, Anthropic 等）已被代码级移除。
*   **禁用外网搜索**: `websearch` (网络搜索) 和 `webfetch` (网页抓取) 工具已被禁用，Agent 无法访问互联网。
*   **禁用自动更新**: 自动检查更新功能默认关闭，以防止向外部服务器发起请求。
