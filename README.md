#

# Claude Code 接入第三方 Anthropic 兼容 API 完整配置指南

本指南总结了在 **Windows / macOS / Linux**（特别是国内网络环境）下安装 **Claude Code**，并接入第三方 Anthropic 兼容中转站（支持 `/v1/messages` 协议）的完整实战流程与避坑方案。

## 目录

1. [前置准备](https://gemini.google.com/app/cdc6c707b3551d69#1-%E5%89%8D%E7%BD%AE%E5%87%86%E5%A4%87 "null")

2. [安装 Claude Code（含国内免代理方案）](https://gemini.google.com/app/cdc6c707b3551d69#2-%E5%AE%89%E8%A3%85-claude-code%E5%90%AB%E5%9B%BD%E5%86%85%E5%85%8D%E4%BB%A3%E7%90%86%E6%96%B9%E6%A1%88 "null")

3. [核心一步：一键写入永久配置并绕过官方验证](https://gemini.google.com/app/cdc6c707b3551d69#3-%E6%A0%B8%E5%BF%83%E4%B8%80%E6%AD%A5%E4%B8%80%E9%94%AE%E5%86%99%E5%85%A5%E6%B0%B8%E4%B9%85%E9%85%8D%E7%BD%AE%E5%B9%B6%E7%BB%95%E8%BF%87%E5%AE%98%E6%96%B9%E9%AA%8C%E8%AF%81 "null")

4. [启动、切换模型与开启高推理强度](https://gemini.google.com/app/cdc6c707b3551d69#4-%E5%90%AF%E5%8A%A8%E5%88%87%E6%8D%A2%E6%A8%A1%E5%9E%8B%E4%B8%8E%E5%BC%80%E5%90%AF%E9%AB%98%E6%8E%A8%E7%90%86%E5%BC%BA%E5%BA%A6 "null")

5. [真假 Claude 鉴别与工具调用测试](https://gemini.google.com/app/cdc6c707b3551d69#5-%E7%9C%9F%E5%81%87-claude-%E9%89%B4%E5%88%AB%E4%B8%8E%E5%B7%A5%E5%85%B7%E8%B0%83%E7%94%A8%E6%B5%8B%E8%AF%95 "null")

6. [常见报错与实战避坑指南（FAQ）](https://gemini.google.com/app/cdc6c707b3551d69#6-%E5%B8%B8%E8%A7%81%E6%8A%A5%E9%94%99%E4%B8%8E%E5%AE%9E%E6%88%98%E9%81%BF%E5%9D%91%E6%8C%87%E5%8D%97faq "null")

## 1. 前置准备

在开始安装前，请确保你的电脑满足以下基础条件：

1. **Git 环境**：Windows 用户需提前安装 [Git for Windows](https://git-scm.com/download/win "null")（安装时一路默认下一步即可）。

2. **Node.js 环境（国内推荐）**：建议安装 [Node.js LTS](https://nodejs.org/zh-cn "null")（自带 `npm`，方便使用国内镜像极速安装）。

3. **中转站信息**：
- **Base URL（接口地址）**：确认中转站支持 Anthropic 原生 `/v1/messages` 接口。通常直接填写中转站根域名（如 `https://api.example.com`，末尾**不要**加 `/v1` 或 `/v1/messages`）。

- **API Key（令牌）**：在中转站后台生成的 `sk-xxx` 密钥。

## 2. 安装 Claude Code（含国内免代理方案）

### 方案 A：使用 NPM + 国内淘宝镜像安装（最推荐，国内直连不卡顿）

#### Windows (PowerShell)

1. **解除 PowerShell 脚本执行限制**（防止报错 `因为在此系统上禁止运行脚本`）：

```
Set-ExecutionPolicy -Scope CurrentUser -ExecutionPolicy RemoteSigned -Force
```

2. **通过国内镜像全局安装并允许初始化脚本**：

```
npm install -g @anthropic-ai/claude-code --allow-scripts=@anthropic-ai/claude-code --registry=https://registry.npmmirror.com
```

> **提示**：如果此前安装时跳过了 `postinstall` 脚本（出现 `npm warn install-scripts`），务必加上 `--allow-scripts=@anthropic-ai/claude-code` 重新运行一次以完成激活。

#### macOS / Linux / WSL

```
npm install -g @anthropic-ai/claude-code --allow-scripts=@anthropic-ai/claude-code --registry=https://registry.npmmirror.com
```

### 方案 B：使用官方原生安装脚本（需海外网络或开启代理 TUN 模式）

- **macOS / Linux / WSL**：

  ```
  curl -fsSL https://claude.ai/install.sh | bash
  ```

- **Windows PowerShell**（注意：PowerShell 中 `curl` 是别名，不可直接运行 `curl -fsSL`）：

  ```
  irm https://claude.ai/install.ps1 | iex
  ```

- **Windows WinGet**：

  ```
  winget install Anthropic.ClaudeCode
  ```

### 验证安装

安装完成后，输入以下命令确认能正常输出版本号：

```
claude --version
```

## 3. 核心一步：一键写入永久配置并绕过官方验证

直接手动修改配置文件极易踩中 **3 个常见大坑**：

1. Windows PowerShell 默认 `UTF8` 带有 **BOM 字节头**，会导致 Claude Code 无法解析 `settings.json`，从而退回直连官方 `api.anthropic.com` 并报错 **`Status 403`**。

2. 首次启动未标记 `hasCompletedOnboarding`，强制触发官方联网引导检查。

3. 新版 Claude Code 未在 `.claude.json` 中将自定义 Key 尾号标记为 `approved`，导致进入后提示 **`Not logged in · Please run /login`**。

下面提供跨平台的一键配置脚本，自动解决以上所有问题：

### Windows (PowerShell) 一键配置脚本

将下方前两行的 `\$baseUrl` 和 `\$apiKey` 替换为你自己的中转站地址和密钥，整段复制到 PowerShell 中回车执行：

```
\$baseUrl = "https://你的中转站域名"
\$apiKey  = "sk-你的中转站API_Key"
$keySuffix = $apiKey.Substring([Math]::Max(0, \$apiKey.Length - 20))

# 1. 清理可能冲突的旧环境变量（避免 AUTH_TOKEN 与 API_KEY 冲突，避免锁死退役模型）
[Environment]::SetEnvironmentVariable("ANTHROPIC_AUTH_TOKEN", \$null, "User")
[Environment]::SetEnvironmentVariable("ANTHROPIC_MODEL", \$null, "User")
[Environment]::SetEnvironmentVariable("ANTHROPIC_DEFAULT_HAIKU_MODEL", \$null, "User")
Remove-Item Env:ANTHROPIC_AUTH_TOKEN -ErrorAction SilentlyContinue
Remove-Item Env:ANTHROPIC_MODEL -ErrorAction SilentlyContinue
Remove-Item Env:ANTHROPIC_DEFAULT_HAIKU_MODEL -ErrorAction SilentlyContinue

# 2. 写入用户环境变量与当前会话变量
[Environment]::SetEnvironmentVariable("ANTHROPIC_BASE_URL", \$baseUrl, "User")
[Environment]::SetEnvironmentVariable("ANTHROPIC_API_KEY", \$apiKey, "User")
$env:ANTHROPIC_BASE_URL = $baseUrl
$env:ANTHROPIC_API_KEY = $apiKey
\$env:CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC = "1"

# 3. 写入无 BOM 的 .claude.json（跳过首次引导 + 自动批准当前 API Key，免除 /login）
\$claudeJson = @"
{
  "hasCompletedOnboarding": true,
  "customApiKeyResponses": {
    "approved": [
      "\$keySuffix"
    ],
    "rejected": []
  }
}
"@
[System.IO.File]::WriteAllText("$HOME\.claude.json", $claudeJson, (New-Object System.Text.UTF8Encoding($false)))

# 4. 写入无 BOM 的 ~/.claude/settings.json 永久配置
New-Item -ItemType Directory -Force -Path "\$HOME\.claude" | Out-Null
\$settingsJson = @"
{
  "env": {
    "ANTHROPIC_BASE_URL": "\$baseUrl",
    "ANTHROPIC_API_KEY": "\$apiKey",
    "CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC": "1"
  }
}
"@
[System.IO.File]::WriteAllText("$HOME\.claude\settings.json", $settingsJson, (New-Object System.Text.UTF8Encoding(\$false)))
```

### macOS / Linux / WSL 一键配置脚本

将下方前两行的 `BASE_URL` 和 `API_KEY` 替换为你自己的信息，复制到终端运行：

```
BASE_URL="https://你的中转站域名"
API_KEY="sk-你的中转站API_Key"
KEY_SUFFIX="\${API_KEY: -20}"

mkdir -p ~/.claude

cat > ~/.claude.json <<EOF
{
  "hasCompletedOnboarding": true,
  "customApiKeyResponses": {
    "approved": [
      "\$KEY_SUFFIX"
    ],
    "rejected": []
  }
}
EOF

cat > ~/.claude/settings.json <<EOF
{
  "env": {
    "ANTHROPIC_BASE_URL": "\$BASE_URL",
    "ANTHROPIC_API_KEY": "\$API_KEY",
    "CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC": "1"
  }
}
EOF

chmod 600 ~/.claude/settings.json ~/.claude.json
```

## 4. 启动、切换模型与开启高推理强度

### 4.1 首次启动与目录授权

进入你的代码项目目录并启动：

```bash
cd /path/to/your/project
claude
```

1. **目录信任检查（Quick safety check）**：首次进入文件夹时，按键盘 **`↓`（下方向键）** 将光标移至 **`Yes, I trust this folder`**，按 **`Enter`** 确认。

2. **检查连接状态**：在对话框输入 `/status` 并回车，确认 `API Base URL` 已显示为你的中转站地址。

### 4.2 选择模型与调整推理强度（Reasoning Effort）

不要在环境变量中死锁旧版或已退役的模型名称（如已退役的 `claude-sonnet-4` 或中转站不支持的别名，会导致后台子代理报 `404 model_not_found`）。

推荐直接在交互界面中切换：

1. 输入 `/model` 并回车。

2. 使用上下方向键选择 **`Default (recommended)`**、**`Opus`** 或 **`Sonnet`**。

3. 在支持推理强度调节的模型上，可根据底部提示切换推理档位，按 **`Enter`** 设为默认模型。

4. **单次深度思考技巧**：遇到复杂逻辑或架构问题时，直接在提问中加入 **`ultrathink`** 或 **`think hard`** 关键词，即可在当前回合拉满思考预算。

## 5. 真假 Claude 鉴别与工具调用测试

配通中转站后，建议用以下 3 个测试验证中转站是否为原生满血模型以及是否完整支持工具调用（Tool Use）：

### 测试 1：心智理论测试（Sally-Anne 糖果盒升级版）

> **Prompt**:
> Sally, Anne, and Jack are in a room with a red box, a blue basket, and a green jar. Sally puts a piece of candy in the red box and leaves the room. While she is gone, Anne moves the candy from the red box to the green jar, and then leaves. Next, Jack takes the candy out of the green jar, eats it, and puts the wrapper into the blue basket. Sally returns to the room and wants her candy. Where will she look for it first, and why?

- **合格表现**：第一句直接回答 `Sally will look in the red box first.`，清晰区分“主观信念”与“客观事实”，并在最后指出糖果已被 Jack 吃掉、只剩糖纸在蓝篮子里。

### 测试 2：物理条件陷阱推理测试（摸糖果问题）

> **Prompt**:
> 在一个黑色的袋子里放有三种口味的糖果，每种糖果有两种不同的形状(圆形和五角星形，不同的形状靠手感可以分辨)。现已知不同口味和形状的数量如下：圆形（苹果7、桃子9、西瓜8），五角星形（苹果7、桃子6、西瓜4）。参赛者需要在活动前决定摸出的糖果数目，最少取出多少个糖果才能保证手中同时拥有不同形状的苹果味和桃子味的糖？

- **鉴别要点**：普通模式容易忽略“靠手感可以分辨形状”而算出盲抓的 `29` 个；开启高推理强度或提醒审题后，应能利用手感策略（取 12 个五角星形 + 9 个圆形）算出最优解 **`21` 个**。

### 测试 3：SVG 动画绘制与本地文件写入测试

> **Prompt**:
> Create an HTML file named `pelican.html` containing a 2D SVG animation of a pelican riding a bicycle.

- **合格表现**：能正常弹出 `Write` 文件写入工具调用请求，生成的鹈鹕具备完整的喉囊、车架、车轮旋转及踩踏板动画。

## 6. 常见报错与实战避坑指南（FAQ）

| **报错现象 / 提示**                                          | **根本原因**                                                                   | **解决方案**                                                                         |
| ------------------------------------------------------ | -------------------------------------------------------------------------- | -------------------------------------------------------------------------------- |
| `Invoke-WebRequest : 找不到与参数名称“fsSL”匹配的参数`              | 在 Windows PowerShell 中直接运行了 Linux 的 `curl -fsSL` 命令                        | 改用 `irm https://claude.ai/install.ps1 \| iex` 或使用上文的 **NPM 镜像安装法**               |
| `无法加载文件 npm.ps1，因为在此系统上禁止运行脚本`                         | PowerShell 默认执行策略拦截了 `.ps1` 脚本                                             | 运行 `Set-ExecutionPolicy -Scope CurrentUser -ExecutionPolicy RemoteSigned -Force` |
| `Failed to connect to api.anthropic.com: Status 403`   | `settings.json` 含有 UTF-8 BOM 头导致解析失败，或未跳过首次引导                              | 使用本文第 3 节的无 BOM 写入脚本重新生成 `.claude.json` 和 `settings.json`                        |
| `Not logged in · Please run /login`                    | 新版客户端未在 `.claude.json` 中将 API Key 尾号加入 `approved` 列表                       | 使用本文第 3 节脚本将 Key 后 20 位写入 `customApiKeyResponses.approved`                       |
| `Both ANTHROPIC_AUTH_TOKEN and ANTHROPIC_API_KEY set`  | 同时设置了两种认证变量产生冲突                                                            | 删除 `ANTHROPIC_AUTH_TOKEN`，仅保留 `ANTHROPIC_API_KEY`（若中转站只认 Bearer 认证则反之）           |
| `Agent terminated early... model_not_found (HTTP 404)` | 环境变量 `ANTHROPIC_MODEL` 或 `ANTHROPIC_DEFAULT_HAIKU_MODEL` 锁死了中转站不存在或已退役的模型名 | 清除这两个环境变量，进入 Claude Code 后使用 `/model` 选择内置模型                                     |
| 粘贴命令后卡住不动 / 无反应                                        | 鼠标误触了 PowerShell 窗口内部触发了“选择暂停”模式，或最后一行未回车                                  | 点击窗口内部按一下 `Enter` 或 `Esc` 键恢复运行                                                  |

## 附：如何查询当前中转站支持的 Claude 模型列表

如果不确定中转站后台具体开放了哪些模型 ID，可在 PowerShell 中运行以下命令直接拉取清单：

```
(Invoke-RestMethod -Uri "https://你的中转站域名/v1/models" -Headers @{Authorization="Bearer sk-你的API_Key"}).data.id | Where-Object { \$_ -match "claude" }
```
