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

## 5. 分发与交付指南

本项目支持两种分发模式：**EXE 便携版**（适合无 Node 环境用户）和 **NPM 包**（适合开发者）。

### 5.1 模式一：发布为 NPM 包 (推荐)

这种方式最符合开发者的习惯，用户可以通过 `npm i -g @yunfan/opencode` 安装，且能自动处理 PATH 和工作目录问题。

**步骤 1: 修改包信息**
编辑 `packages/opencode/package.json`：
```json
{
  "name": "@yunfan/opencode",
  "version": "1.0.0",  // 每次发布前递增
  "publishConfig": {
    "registry": "https://your-internal-registry.com/" // 如果有内部私服
  }
}
```

**步骤 2: 构建产物**
在 `packages/opencode` 目录下运行：
```bash
# 确保先安装依赖
bun install
# 执行构建 (注意：NPM 发布不需要构建成单一 exe，而是编译成 JS)
bun run build
```
*(注意：如果是标准 NPM 发布，通常只需要编译 TypeScript 到 dist/index.js，这里的 build 脚本是为 exe 设计的。对于 NPM 发布，其实直接发布源码或简单的 tsc 编译产物即可。如果是发布官方那种 CLI，通常会包含 postinstall 脚本来下载二进制，或者直接发布纯 JS 版本。鉴于我们的修改，建议直接发布 `dist` 目录下的内容，或者参考下文的 EXE 打包。)*

**简化版 NPM 发布流程 (纯 JS 模式)**:
如果不希望依赖复杂的二进制下载逻辑，最简单的方法是保留当前的源码发布模式。确保 `bin` 字段指向正确的入口文件。

```bash
cd packages/opencode
npm publish --access public
```

### 5.2 模式二：构建 EXE 便携版

适合分发给所有用户，解压即用。

**步骤 1: 构建二进制文件**
```bash
cd packages/opencode
bun run script/build.ts --single --skip-install
```

**步骤 2: 准备交付包**
创建一个文件夹（如 `OpenCode_Internal_v1.0`），放入以下文件：
1.  `opencode.exe` (从 `dist/opencode-windows-x64/bin` 复制)
2.  `opencode.json` (预配置文件，含 `internal` provider 设置)
3.  `install_right_click_menu.bat` (右键菜单安装脚本)

**步骤 3: 压缩分发**
将该文件夹压缩为 ZIP 发送给用户。

## 6. 配置与数据存储 (Windows)

在 Windows 平台上，OpenCode 使用类 Unix/XDG 风格的目录结构，而非标准的 `AppData\Roaming`。

### 6.1 关键路径
*   **配置文件 (`opencode.json`)**:
    `C:\Users\<User>\.config\opencode\opencode.json`
    *(注意：不是 AppData)*

*   **认证信息 (`auth.json`)**:
    `C:\Users\<User>\.local\share\opencode\auth.json`
    *(存储 Connect 界面输入的 API Key 和 BaseURL)*

*   **日志文件**:
    `C:\Users\<User>\.local\share\opencode\log`

### 6.2 快速定位命令
如果需要在用户机器上确认实际使用的路径，可以运行以下命令：

```powershell
.\opencode.exe debug paths
```

输出示例：
```
home       C:\Users\User
data       C:\Users\User\.local\share\opencode
config     C:\Users\User\.config\opencode
...
```

---
*文档生成日期: 2026-01-18*