# OpenCode 内部版二次开发指南

本文档旨在为开发人员提供维护、扩展和定制 OpenCode 内部发行版所需的核心知识。本项目已基于 OpenCode 开源版本进行了深度改造，以适应企业内网环境。

## 1. 核心架构改造概览

为了实现“零外部依赖”和“内网安全”，我们主要对 **Provider (模型供应商)** 和 **Permission (权限控制)** 两个模块进行了外科手术式的修改。

### 1.1 Provider 拦截与注入
我们通过修改 `packages/opencode/src/provider/provider.ts` 实现了以下逻辑：
1.  **Loader 拦截**: 注册了一个名为 `internal` 的自定义加载器 (CustomLoader)。
2.  **动态发现**: 该加载器会尝试请求 `GET /models` 端点，自动注册服务器支持的所有模型。
3.  **强制过滤**: 在 `state()` 函数的最后，我们添加了强制过滤逻辑，**删除了除 `internal` 以外的所有 Provider**。这确保了用户界面中不会出现 OpenAI, Anthropic 等外部选项。

### 1.2 网络功能剥离
我们通过两层防御禁用了外网访问：
1.  **注册层**: 在 `packages/opencode/src/tool/registry.ts` 中，移除了 `WebSearchTool` 和 `WebFetchTool` 的注册。
2.  **权限层**: 在 `packages/opencode/src/agent/agent.ts` 中，将 `websearch` 和 `webfetch` 的默认权限显式设为 `deny`。

## 2. 关键文件路径索引

| 模块 | 文件路径 | 作用 |
| :--- | :--- | :--- |
| **后端逻辑** | `packages/opencode/src/provider/provider.ts` | **核心!** 定义 `internal` loader，注入默认配置，过滤外部 Provider。 |
| **API 路由** | `packages/opencode/src/server/routes/provider.ts` | 控制 `/connect` 列表的返回内容。确保这里与 `provider.ts` 定义一致。 |
| **前端配置** | `packages/app/src/components/dialog-connect-provider.tsx` | 定制 `/connect` 弹窗中的提示文案 (如 `BaseURL|ApiKey` 说明)。 |
| **前端钩子** | `packages/app/src/hooks/use-providers.ts` | 控制前端“热门供应商”列表，防止 UI 尝试加载不存在的图标。 |
| **权限控制** | `packages/opencode/src/agent/agent.ts` | 定义 Agent 的默认权限 (Allow/Deny)。 |
| **构建脚本** | `packages/opencode/script/build.ts` | 控制打包流程。 |

## 3. 开发与构建指令手册

所有命令均需在安装了 `bun` 环境的终端中执行。

### 环境准备
```bash
# 根目录安装依赖
bun install
```

### 启动开发模式
最快的调试方式，直接运行源码，支持热重载。
```bash
cd packages/opencode
bun run dev
```

### 构建发行版 (Windows)
生成可分发的 `.exe` 文件。
```bash
cd packages/opencode
# --single: 仅构建当前平台
# --skip-install: 跳过重复的依赖安装 (节省时间)
bun run script/build.ts --single --skip-install
```
> **产物位置**: `packages/opencode/dist/opencode-windows-x64/bin/opencode.exe`

### 常见维护场景

#### 场景 A: 恢复外部网络搜索功能
如果您决定放开网络权限：
1.  编辑 `packages/opencode/src/tool/registry.ts`: 将 `WebSearchTool` 加回列表。
2.  编辑 `packages/opencode/src/agent/agent.ts`: 将 `websearch` 权限改为 `allow` 或 `ask`。

#### 场景 B: 修改默认 API 地址
编辑 `packages/opencode/src/provider/provider.ts`:
搜索 `baseURL = baseURL ??`，修改其后的字符串即可。

#### 场景 C: 修复 "File in use" 构建错误
构建脚本 `rm -rf dist` 失败通常是因为旧版 exe 正在运行。
**解决方法**: 手动关闭所有 `opencode` 窗口，或在 PowerShell 运行：
```powershell
Stop-Process -Name opencode -ErrorAction SilentlyContinue
```

## 4. 版本控制建议

由于这是对开源项目的 Fork 修改，建议维护一个独立的 `internal-dev` 分支。
*   **上游同步**: 定期从 `anomalyco/opencode` 的 `main` 分支拉取更新。
*   **冲突处理**: 重点关注 `provider.ts` 的冲突，因为这是我们修改最重的地方。

---
*文档生成日期: 2026-01-17*
