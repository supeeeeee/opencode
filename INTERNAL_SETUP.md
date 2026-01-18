# 内部部署使用与配置指南

本项目已针对公司内部环境进行深度改造，移除了所有外部 LLM 供应商依赖，禁用了外网访问，并增强了对内网模型的适配能力。

## 1. 快速开始 (便携模式)

本程序支持“绿色版”运行。

1.  将 `opencode.exe` 和 `opencode.json` (可选) 放在同一个文件夹中。
2.  双击运行 `opencode.exe`，它会自动读取同级目录下的配置。
3.  **推荐**: 运行 `install_right_click_menu.bat` (如果提供)，可在任意文件夹右键选择 "Open with OpenCode"。

## 2. 连接内部模型

### 方式一：一键连接 (UI 界面)

初次运行或未配置时，程序会提示连接供应商。选择 **"Internal Server"**。

在 API Key 输入框中，支持以下三种格式：

*   **仅 API Key**: `sk-my-key`
    *   使用默认地址 (https://api.deepseek.com/v1) 和默认模型 (deepseek-chat)。
*   **地址 + Key**: `http://192.168.1.10:8000/v1|sk-my-key`
    *   自动连接指定服务器。如果服务器支持自动发现，会自动列出所有模型。
*   **地址 + Key + 模型**: `http://192.168.1.10:8000/v1|sk-my-key|qwen-72b`
    *   直接指定使用 `qwen-72b` 模型，适合不支持自动发现的服务器。

### 方式二：配置文件 (opencode.json)

在 `opencode.exe` 同级目录或用户目录 (`~/.config/opencode/opencode.json`) 创建文件：

**基础配置：**
```json
{
  "provider": {
    "internal": {
      "options": {
        "baseURL": "http://192.168.1.50:8000/v1",
        "apiKey": "sk-internal-key"
      }
    }
  }
}
```

**高级配置 (手动声明多个模型)：**
如果您的服务器不支持 `/models` 自动发现接口，您可以手动列出模型：

```json
{
  "provider": {
    "internal": {
      "options": {
        "baseURL": "http://192.168.1.50:8000/v1",
        "apiKey": "sk-key"
      },
      "models": {
        "qwen-turbo": { "name": "通义千问 Turbo" },
        "llama3-70b": { "name": "Llama3 70B" },
        "deepseek-coder": { "name": "DeepSeek Coder" }
      }
    }
  }
}
```

### 方式三：环境变量

适合临时测试或 CI/CD 环境：

*   `INTERNAL_API_BASE`: API 地址 (如 `http://localhost:11434/v1`)
*   `INTERNAL_API_KEY`: API 密钥
*   `INTERNAL_MODEL`: 默认模型 ID (如 `llama3`)

## 3. 功能限制与安全说明

为适应内网安全要求，本版本包含以下硬性限制：

*   **网络隔离**:
    *   **Provider**: 仅允许使用 `internal` provider，所有外部供应商 (OpenAI, Anthropic 等) 已被代码级移除。
    *   **工具**: `websearch` (网络搜索) 和 `webfetch` (网页抓取) 工具已被移除，Agent 无法访问互联网。
*   **数据防泄露 (DLP)**:
    *   **分享禁用**: 会话分享功能 (`share`) 默认强制设为 `"disabled"`，防止代码上下文上传。
*   **运维管控**:
    *   **自动更新**: 自动更新检查默认关闭 (`autoupdate: false`)。

## 4. 常见问题

**Q: 为什么 `/connect` 里只有一个选项？**
A: 这是特意设计的。为了防止员工连接未经授权的外部 AI 服务，我们移除了所有其他选项。

**Q: 我的模型叫 `custom-model`，为什么连不上？**
A: 请尝试在连接字符串末尾加上模型 ID：`URL|Key|custom-model`。或者在 `opencode.json` 中显式定义它。

**Q: 我想用 DeepSeek 官方服务，怎么配？**
A: 直接在 `/connect` 中输入您的 DeepSeek API Key 即可，其他留空。程序默认配置就是 DeepSeek。
