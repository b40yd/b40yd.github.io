+++
title = "OpenCode 安装与使用"
date = 2026-01-23
lastmod = 2026-01-23T18:45:56+08:00
tags = ["opencode", "oh-my-opencode", "agent", "Claude Code"]
categories = ["opencode", "oh-my-opencode", "agent", "Claude Code"]
draft = false
author = "b40yd"
+++

![](/opencode-info.png)
### 1. OpenCode 简介

- OpenCode 是一个热门的 AI 编程工具，可视为开源版的 Claude Code。它具有以下优势：

*   **免费模型：** 开箱即用，内置主流免费模型（如 GLM-4.7, MiniMax-2.1）。
*   **顶级模型接入：** 通过简单配置可免费接入 GEMINI-3.5 Pro、Claude Opus-4.5、ChatGPT Codex 等顶级编程模型。
*   **对中国用户友好：** 避免 Claude Code 的限速、封号等问题。
*   **功能全面：** 具备 Claude Code 的一切功能，支持 Agent Skills、MCP、Session 管理等高级特性。


### 2. OpenCode 四种形态
- OpenCode 有四种运行形态：命令行、桌面客户端、插件、云端运行环境。视频主要推荐和演示命令行版本。


#### 2.1 命令行版本安装
- 这是视频推荐的主要安装方式。
1.  **安装 Node.js：**
    *   访问 [Node.js 官网](https://nodejs.org/)，根据你的操作系统下载并安装 Node.js。
- 2.  **安装 OpenCode：**
    *   打开终端（命令行窗口）。
    *   复制以下 NPM 安装命令并执行：
        ```bash
        npm install -g opencode-ai
        ```
    *   安装完成后，输入 `opencode` 即可启动。

#### 2.2 桌面客户端安装 (Beta 版，Bug较多)
1.  访问 OpenCode 官方页面，点击下载客户端。
2.  一路点击“下一步”完成安装。
3.  打开后选择一个文件夹作为项目目录即可使用。
    *   **注意：** 客户端功能基础，目前处于 Beta 测试版，Bug 较多，视频不推荐作为主力使用。


#### 2.3 插件版安装 (以 VS Code 为例)
- 插件版需要先安装命令行版 OpenCode。
1.  **打开 VS Code。**
2.  **安装 OpenCode 插件：**
    *   在左侧的 Extensions (扩展) 视图中搜索 `opencode`。
    *   找到 `OpenCode` 插件并点击安装。
3.  **启动插件：**
    *   安装完成后，按 `Ctrl + Shift + P` (macOS: `Cmd + Shift + P`) 打开命令面板。
    *   输入 `open open code` 并回车。
4.  **快捷操作：**
    *   插件可以自动关联左侧窗口打开的代码文件。
    *   选中代码后，按 `Ctrl + Alt + K` (macOS: `Cmd + Alt + K`) 可以快速粘贴代码到 OpenCode 聊天窗口。


#### 2.4 云端运行环境 (以 GitHub 为例)
- OpenCode 可以在 GitHub 云端运行，帮助解释问题、自动修复问题并创建 PR 等。
1.  **上传项目到 GitHub：**
    *   将你的代码项目发布到 GitHub 仓库（例如通过 VS Code 的 Source Control 功能）。
2.  **安装 OpenCode GitHub App：**
    *   在你的 GitHub 仓库页面，复制 OpenCode GitHub App 的安装命令（通常在 OpenCode 文档或 GitHub 集成页面提供）。
    *   在项目的根目录执行该命令，并按照指引完成安装（选择模型提供商、API Key 等）。
3.  **配置环境变量：**
    *   根据安装指引，将所需的 `API_KEY` 和 `PROJECT_ID` 等环境变量添加到你的 GitHub 仓库 `Settings > Secrets and variables > Actions > Repository secrets` 中。
4.  **在 Issue 中使用：**
    *   在 GitHub Issue 中，可以直接输入 `/opencode [你的需求]`（如 `/opencode fix bug: ...`），OpenCode 就会在云端执行任务，创建 Pull Request 来修复问题。


### 3. 免费模型和顶级模型接入


#### 3.1 使用内置免费模型
*   启动 OpenCode 后，输入命令 `/models`。
*   带有 `free` 标记的模型（如 GLM-4.7, MiniMax-2.1）都可以免费使用，无需额外配置。


#### 3.2 接入 Google Antigravity (GEMINI-3.5 Pro, Claude Opus-4.5)
- Antigravity 是 Google 推出的 AI 编程 IDE，慷慨地提供了 GEMINI-3.5 Pro 等。通过插件可免费接入 OpenCode。
1.  **安装插件：**
    *   访问 [opencode-antigravity-auth](https://github.com/NoeFabris/opencode-antigravity-auth) 获取安装提示词。
    *   在 OpenCode 中粘贴该提示词并执行。AI 会自动完成安装。

    ```text
    Install the opencode-antigravity-auth plugin and add the Antigravity model definitions to ~/.config/opencode/opencode.json by following: https://raw.githubusercontent.com/NoeFabris/opencode-antigravity-auth/dev/README.md
    ```

2.  **登录 Google 账户：**
    *   复制安装完成后提供的登录命令。
    *   打开一个新的终端窗口执行登录命令。
    *   选择 `Google` 作为模型供应商，选择 `Antigravity` 登录方式，`project id` 直接回车。
    *   在浏览器中登录你的 Google 账户，并将生成的 URL 粘贴回命令行。
    *   选择 `N` 回车完成配置。
3.  **验证：**
    *   重启 OpenCode。
    *   输入 `/models`，即可看到 GEMINI-3.5 Pro (Antigravity)可供选择。


#### 3.3 接入 OpenAI GPT Codex (需 ChatGPT Plus 订阅)
1.  **确保订阅 ChatGPT Plus：**
    *   你需要在 ChatGPT 官网订阅至少 Plus 级别的套餐。
2.  **连接 OpenAI：**
    *   在 OpenCode 中输入命令 `/connect` 并回车。
    *   选择 `OpenAI`，然后选择 `GPT Pro`。
    *   浏览器会自动打开链接，完成 OpenAI 账户登录。
3.  **验证：**
    *   回到 OpenCode，输入 `/models`，即可选择 ChatGPT 的各种模型（包括 Codex）。


#### 3.4 接入 OpenRouter (几乎所有主流大模型)
- OpenRouter 可以接入市面上几乎所有类型的大模型，且国内用户方便获取额度。
1.  **获取 OpenRouter API Key：**
    *   访问 [OpenRouter 官网](https://openrouter.ai/)。
    *   点击 `Get API Key` 创建并复制你的 API Key。
2.  **连接 OpenRouter：**
    *   在 OpenCode 中输入命令 `/connect` 并回车。
    *   选择 `OpenRouter`。
    *   粘贴你的 API Key 并回车。
3.  **验证：**
    *   现在你可以通过 OpenRouter 访问更多大模型。


### 4. OpenCode 核心功能与使用


#### 4.1 Session 管理
- OpenCode 强调 Session 概念，每次新的对话都是一个独立的 Session，支持后台运行和并行开发。
*   **创建新 Session：** 在当前任务执行过程中输入 `new` 命令。
*   **查看 Session：** 输入 `/sessions` 查看所有 Session 及其状态。
*   **切换 Session：** 在 `/sessions` 列表中选择要切换的 Session。
*   **并行开发：** 利用多个 Session 同时处理不同需求。


#### 4.2 代码比对界面
*   命令行版的代码比对界面非常优秀，能够清晰展示代码修改。


#### 4.3 分享与导出
*   **分享 Session：** 输入 `/share` 命令，可以将当前 Session 的对话记录分享为一个网页链接。
*   **取消分享：** 输入 `/unshare` 命令。
*   **导出记录：** 输入 `/export` 命令，可以将对话记录导出为文件。


#### 4.4 时间线 / 检查点 (Timeline)
*   **查看时间线：** 输入 `/timeline` 命令。
*   **代码回溯：** 可以选择任意一次对话记录，使用 `revert` 功能，将代码和聊天内容一起回溯到该时间点之前的状态，方便重新修改。


#### 4.5 Agent Skills (技能包)
- Agent Skills 类似于带目录的说明书，每个文件夹对应一个技能包。
1.  **迁移 Skills：**
    *   在 OpenCode 项目文件夹根目录，新建一个 `.opencode` 文件夹。
    *   在 `.opencode` 文件夹内，再新建一个 `skills` 文件夹。
    *   将 `.claude` 目录下的技能文件夹复制到 `.opencode/skills` 目录，并将其中的 `.claude` 文件名替换为 `.opencode`。
2.  **使用 Skills：**
    *   启动 OpenCode。
    *   可以询问 OpenCode “你有哪些 Skills？” 来验证。
    *   当 AI 需要时，它会自动调用这些技能进行工作。


#### 4.6 MCP  配置
- OpenCode 支持 Local (本地命令) 和 Remote (远程调用) 两种 MCP。
1.  **打开配置文件：**
    *   导航到 OpenCode 的配置文件夹：`~/.config/opencode/opencode.json`。
2.  **添加 MCP 配置：**
    *   **Local MCP 示例 (`sh.cn`)：**
        ```json
        {
          "mcp": {
            "mcp_everything": {
            "type": "local",
            "command": ["npx", "-y", "@modelcontextprotocol/server-everything"],
            },
          },
        }
        ```
    *   **Remote MCP 示例 (`context7m` Server)：**
        ```json
        {
          "mcp": {
            "context7": {
              "type": "remote",
              "url": "https://api.context7.com/mcp", // 示例 URL
              "headers": {
                "Authorization": "Bearer YOUR_CONTEXT7_API_KEY" // 替换为你的 API Key
              },
              "env": {},
              "enabled": true
            }
            // ... 其他 MCP 配置
          }
        }
        ```
    *   **注意：** 确保 JSON 格式正确，逗号等符号不要遗漏。
3.  **验证：**
    *   重启 OpenCode。
    *   输入 `/mcp` 即可看到已配置的 MCP。


### 5. Oh My OpenCode 插件：超强编程体验
- oh-my-opencode 是一个集成了工具、MCP 和编程 Agent 的插件。


#### 5.1 插件特性
*   **集成工具：** LSP (语法语义理解)、AST 工具 (代码语法树搜索)、LOC 工具 (多模态理解图片/PDF)、Delegate Task (任务分配)、Background Task (后台调度)。
*   **内置 MCP Server：** Web Search (网络搜索)、Context7 (最新文档获取)、grep app (GitHub 仓库代码搜索)。
*   **七大编程智能体：**
    *   Sisyphus (西西弗斯)：主智能体，规划和调度任务。
    *   Oracle (先知)：架构设计、代码评审。
    *   Librarian (图书管理员)：查阅文献。
    *   Explorer (探索者)：网络搜索。
    *   Frontend Engineer (前端工程师)：前端开发。
    *   Documenter (文档编写者)：文档编写。
    *   Multimodal (多模态)：处理多模态信息。
*   **模型优化：** 为每个智能体分配最适合其工作的大模型（如前端工程师使用 GEMINI-3.5 Pro，主 Agent 使用 Claude Opus-4.5）。


#### 5.2 插件安装
1.  **获取安装提示词：**
    *   访问该插件的 GitHub 首页，复制以 `install` 开头的一段提示词。
    
    ```text
        Install and configure oh-my-opencode by following the instructions here: 
        https://raw.githubusercontent.com/code-yeongyu/oh-my-opencode/refs/heads/master/docs/guide/installation.md
    ```

2.  **安装：**
    *   在 OpenCode 中粘贴该提示词并执行。
    *   插件会询问你是否有 Claude、ChatGPT、GEMINI 订阅，根据实际情况选择。
    *   选择 `Yes` 确认后，插件会自动安装。


#### 5.3 插件配置 (智能体模型选择)
1.  **打开配置文件：**
    *   导航到 `~/.config/opencode/oh-my-opencode.json`。
2.  **调节智能体模型：**
    *   该文件定义了七大智能体所用的模型。你可以根据自己的需求进行调整（例如将 Sisyphus 的模型改为 GPT-4）。


#### 5.4 插件使用方法
*   **指定智能体：** 输入 `@` 符号，然后选择一个智能体来为你工作（例如 `@frontend-ui-ux-engineer`）。
*   **Ultra Work 模式：** 输入魔法词 `ULW` 或 `ultra work`。
    *   在此模式下，OpenCode 会调用其所有潜能，将任务分配给多个智能体并行执行，并由主智能体 Sisyphus 进行调度。适用于复杂、多步骤的编程任务。
*   **Ralph Loop 循环模式：** 输入 `/ralph` 或 `/ralph-loop` 命令。
    *   该模式可以强制 AI 长时间循环对一个困难任务持续工作，直到任务完成或所有测试用例通过。适用于需要长时间迭代和优化的重构或复杂开发任务。


### 6. 其他重要功能
*   **`/init`：** 让 AI 通读整个项目文件夹，将项目知识生成 `agents.md` 文件作为系统提示词，帮助 AI 快速了解项目。
*   **`/compact`：** 压缩之前对话的上下文，提炼成简洁摘要，释放模型上下文窗口。
*   **自定义命令：**
    *   在 OpenCode 配置文件夹 (`.config/opencode/`) 中新建 `commands` 文件夹。
    *   在该文件夹内以 Markdown 文件的形式定义自定义命令（如 `run_tests.md`），指定 `build` 或 `plan` 模式，并编写描述。
    *   使用时输入 `/命令名` (如 `/run_tests`)。
*   **定制智能体：**
    *   在 OpenCode 配置文件夹 (`.config/opencode/`) 中新建 `agents` 文件夹。
    *   在该文件夹内新建 Markdown 文件定义智能体（如 `code_reviewer.md`），指定描述、类型 (`agent` 或 `primary`)、模型等。
    *   `agent` (sub-agent) 类型由主智能体调度在后台执行。
    *   `primary` (主智能体) 类型可以通过 Tab 键直接切换到该智能体进行工作。
