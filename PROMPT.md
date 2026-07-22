# Codex 执行提示词：在 Linux Codex CLI 中接入 DeepSeek 全局子 Agent

请在当前仓库中实现一套**可复现、可安装、可卸载、不会破坏现有 Codex 配置**的方案，使 Linux 用户能够：

```text
GPT-5.6 / ChatGPT 会员模型作为 Codex 主 Agent
        ↓ 规划、拆分、复核
DeepSeek V4 Flash 作为低成本全局子 Agent
        ↓ 代码搜索、日志分析、局部检查、低风险候选修改
GPT-5.6 主 Agent 最终审查和验证
```

不要只给使用说明，请在当前仓库中实际创建所需脚本、模板和文档，并运行能够在当前环境运行的非破坏性验证。

---

## 一、必须先确认的事实

开始实现前，阅读并核对以下官方资料，以执行时的最新内容为准：

- Codex 自定义 Provider 配置：
  - https://developers.openai.com/codex/config-reference
  - https://developers.openai.com/codex/config-advanced
- Codex 自定义子 Agent：
  - https://developers.openai.com/codex/agent-configuration/subagents
- DeepSeek API：
  - https://api-docs.deepseek.com/zh-cn/
  - https://api-docs.deepseek.com/zh-cn/api/create-chat-completion/

当前已知基线如下，但必须重新验证：

1. Codex 自定义 Provider 目前要求 `wire_api = "responses"`；
2. DeepSeek 官方 API 当前主要提供 `/chat/completions` 和 Anthropic 兼容接口，没有原生 `/v1/responses`；
3. 因此不能简单地把 `https://api.deepseek.com` 直接写成 Codex Responses Provider；
4. 需要一个运行在本机、只监听 `127.0.0.1` 的 Responses → DeepSeek Chat Completions 兼容代理；
5. DeepSeek 当前推荐模型为：
   - 默认低成本子 Agent：`deepseek-v4-flash`；
   - 可选增强模型：`deepseek-v4-pro`；
6. 不要再默认使用即将或已经弃用的 `deepseek-chat`、`deepseek-reasoner`。

如果执行时 DeepSeek 已经正式支持完整的 Responses API，并且经过实际请求验证，那么可以改为直接接入；否则继续使用本地代理。

---

## 二、实现目标

在当前仓库中产出一套针对 Linux + Codex CLI 的安装方案，至少包含：

```text
README.md
scripts/install.sh
scripts/uninstall.sh
scripts/check.sh
templates/deepseek-flash-worker.toml
examples/test-prompt.md
.gitignore
```

可以按需要增加其他文件，但不要引入与目标无关的复杂平台。

### 最终安装效果

安装完成后，用户目录至少应存在：

```text
~/.codex/config.toml
~/.codex/agents/deepseek-flash-worker.toml
```

Codex 主线程仍然使用用户原有的 ChatGPT 登录和 GPT-5.6，不能被切换为 DeepSeek。

用户应能够在任意项目中对 Codex 主 Agent 说：

```text
请使用 deepseek_flash_worker 子 Agent，只读分析当前仓库的目录结构、待完成任务和潜在风险。
主 Agent 负责复核结论，不要让子 Agent 修改文件。
```

---

## 三、代理选择

优先采用一个**已经针对 Codex Responses API 与 DeepSeek V4 做过适配、可审计源代码、支持 Linux**的轻量本地代理，而不是自行从零实现协议转换。

候选方案可包括：

- `deepseek-proxy`；
- `holo-q/deepseek-responses-proxy`；
- 其他在执行时更成熟且明确支持 Codex CLI 的方案。

选择前必须检查：

1. 项目来源、许可证和维护状态；
2. 是否支持 `/v1/responses`；
3. 是否支持 SSE；
4. 是否能够处理 Codex 的工具调用格式；
5. 是否正确保留 DeepSeek 思考模式工具调用所需的上下文；
6. 是否只监听本机地址；
7. 是否会记录请求正文或 API Key；
8. 是否可以固定版本，避免未来升级突然破坏兼容性。

在 README 中说明最终选择及原因，同时明确它属于第三方兼容层，不得将其描述成 DeepSeek 官方组件。

如果候选代理无法可靠支持 Codex 工具调用，第一版必须把 DeepSeek 子 Agent 限制为只读，并如实记录限制，不得伪造“完全兼容”。

---

## 四、安装脚本要求

### `scripts/install.sh`

必须满足：

1. 使用 `set -euo pipefail`；
2. 检查 Linux、`codex`、Python/uv/pip 等必要依赖；
3. 不使用 `sudo` 修改系统文件；
4. 优先安装到用户目录或隔离虚拟环境；
5. 固定代理版本或 Git commit；
6. 代理只绑定 `127.0.0.1`，默认端口可使用 `8787`；
7. 不把真实 API Key 写入仓库；
8. 不在终端日志中打印完整 API Key；
9. 修改 `~/.codex/config.toml` 前必须创建带时间戳备份；
10. 必须以幂等方式合并配置，不能粗暴覆盖整个文件；
11. 如果配置中已经存在同名 Provider 或 Agent，先检测差异并给出安全处理；
12. 不修改用户现有的顶层：
    - `model`
    - `model_provider`
    - ChatGPT 登录配置
13. 仅增加或更新 DeepSeek Provider 和必要的 `[agents]` 配置；
14. 将全局 Agent 安装到 `~/.codex/agents/`；
15. 安装结束后输出清晰的下一步命令。

建议写入的 Provider 结构如下，具体字段以当前 Codex 版本验证结果为准：

```toml
[model_providers.deepseek_local]
name = "DeepSeek through local Responses proxy"
base_url = "http://127.0.0.1:8787/v1"
env_key = "DEEPSEEK_API_KEY"
wire_api = "responses"
```

不得在 Provider 配置中写入真实 Key。

对 `[agents]` 的修改必须保留用户已有字段。第一版建议：

```toml
[agents]
max_threads = 4
max_depth = 1
```

如果用户已经设置这些值，不得无提示覆盖；仅在缺失时添加，或保留更严格的现有配置。

### API Key

安装脚本不得把 API Key 提交到 Git，也不得默认追加到公开 shell 配置。

支持以下安全方式之一，并在 README 中说明：

```bash
export DEEPSEEK_API_KEY="用户自己的密钥"
```

也可以提供可选的 `systemd --user`/环境文件方案，但环境文件必须：

- 位于用户配置目录；
- 权限为 `chmod 600`；
- 被明确标记为包含敏感信息；
- 不被提交到 Git。

### `scripts/uninstall.sh`

必须：

1. 只删除本仓库安装的 Agent、代理环境和服务；
2. 不删除整个 `~/.codex`；
3. 从 `config.toml` 中仅移除本方案添加的 Provider 配置；
4. 修改前再次备份；
5. 不影响 GPT-5.6 主 Agent 和其他 Provider；
6. 对文件不存在、重复卸载保持幂等。

---

## 五、全局 DeepSeek 子 Agent 模板

创建：

```text
templates/deepseek-flash-worker.toml
```

安装后复制到：

```text
~/.codex/agents/deepseek-flash-worker.toml
```

至少包含以下语义，可以根据 Codex 当前格式修正字段：

```toml
name = "deepseek_flash_worker"

description = """
使用 DeepSeek V4 Flash 完成低成本、边界明确的代码搜索、日志分析、
依赖检查、测试结果归纳和只读代码审查。
不负责最终架构、业务规则、安全决策或高风险修改。
"""

nickname_candidates = [
  "DeepSeek-Scout",
  "DeepSeek-Tracer",
  "DeepSeek-Mapper",
  "DeepSeek-Reviewer"
]

model_provider = "deepseek_local"
model = "deepseek-v4-flash"

sandbox_mode = "read-only"
approval_policy = "untrusted"

developer_instructions = """
你是由 Codex 主 Agent 调度的低成本只读子 Agent。

工作边界：
1. 只完成父 Agent 明确分配的一个边界清晰任务；
2. 你不会继承父 Agent 的完整对话，父 Agent 的任务说明就是你的完整交接；
3. 优先进行精确搜索和定向读取，不做无边界全仓库扫描；
4. 返回明确文件路径、符号、证据和仍未确认的事项；
5. 不修改文件，不执行 git add、commit、push；
6. 不改变需求、架构、业务规则、鉴权、事务、幂等和安全策略；
7. 不读取或输出密钥、Token、真实客户数据和未脱敏文件；
8. 不声称未运行的测试已经通过；
9. 工具或代理行为异常时直接报告，不得编造结果；
10. 最终结论由 GPT-5.6 主 Agent 复核。

返回格式：
- 结论摘要
- 检查过的文件和位置
- 证据
- 风险或不确定项
- 建议主 Agent 下一步验证
"""
```

第一版保持 `read-only`。只有实际验证搜索、shell 工具、补丁和测试流程都可靠后，才能另行新增写入型 Agent；不要直接把本 Agent 改成 `workspace-write`。

---

## 六、检查脚本

创建 `scripts/check.sh`，至少检查：

1. `codex --version`；
2. `DEEPSEEK_API_KEY` 是否存在，但不得输出值；
3. 本地代理进程是否可访问；
4. `POST http://127.0.0.1:8787/v1/responses` 的最小非流式请求；
5. 流式 SSE 请求；
6. DeepSeek Provider 是否已经合并到 `~/.codex/config.toml`；
7. 全局 Agent TOML 是否存在并能通过基本 TOML 解析；
8. 主 Codex 顶层 Provider 是否未被安装脚本改成 DeepSeek；
9. 失败时输出具体原因和修复建议；
10. 返回正确的 shell exit code。

不要把一次简单文本返回当成完整成功。至少区分：

```text
代理连通性通过
Responses 基础转换通过
SSE 通过/失败
Codex 子 Agent 工具调用尚未验证
```

---

## 七、真实验证流程

实现后按顺序验证，并在 README 中记录实际执行结果；未执行的项目必须明确标记为“未验证”。

### 1. 静态验证

- Shell 脚本语法；
- TOML 格式；
- `.gitignore` 不会遗漏密钥、环境文件和临时日志；
- 安装、重复安装、卸载、重复卸载的幂等逻辑。

### 2. 代理验证

使用 `deepseek-v4-flash` 完成：

- 非流式普通文本；
- 流式普通文本；
- 至少一个工具调用或与 Codex 工具格式接近的验证。

### 3. Codex CLI 验证

不能把主 Agent 切换到 DeepSeek。应保持主线程走用户原有 GPT-5.6，然后让主 Agent 生成自定义子 Agent。

在 `examples/test-prompt.md` 中提供可以直接粘贴到 Codex CLI 的测试提示词，例如：

```text
请保持当前 GPT-5.6 主 Agent，不要切换主模型。

生成一个 deepseek_flash_worker 子 Agent，只读检查当前仓库：
1. 列出顶层目录和用途；
2. 找出 README、TODO 或任务入口；
3. 指出一个需要主 Agent 进一步确认的风险；
4. 不修改任何文件。

等待子 Agent 完成后，由主 Agent 检查其文件路径和结论是否真实，
并明确说明本次子任务是否由 DeepSeek Provider 执行。
```

验证成功标准：

1. Codex 显示创建了 `deepseek_flash_worker`；
2. DeepSeek 控制台出现对应 API 用量；
3. 主 Agent 仍使用原有 GPT-5.6/ChatGPT 登录；
4. 子 Agent 能读取当前项目并返回真实文件路径；
5. 子 Agent 没有修改任何文件；
6. 主 Agent 能复核结果。

---

## 八、README 必须说明

README 使用中文，至少包含：

1. 架构图；
2. 为什么 DeepSeek 官方地址不能直接作为 Codex Responses Provider；
3. 前置依赖；
4. DeepSeek API Key 获取后如何通过环境变量提供；
5. 一键安装；
6. 启动、停止和查看代理状态；
7. 检查命令；
8. 如何在任意项目调用全局 Agent；
9. 如何确认主 Agent 与子 Agent 分别走了哪个 Provider；
10. 如何卸载；
11. 常见错误：
    - `/v1/responses` 404；
    - 401/403；
    - 模型 ID 不存在；
    - SSE 中断；
    - 子 Agent 回退到 OpenAI Provider；
    - 工具调用格式不兼容；
    - 代理未启动；
12. 安全说明；
13. 已验证版本矩阵；
14. 已知限制。

明确说明：

- 这套方案节省的是 Codex 主模型承担执行型工作的额度；
- GPT-5.6 主 Agent 仍会消耗额度用于拆分、复核和最终回答；
- 子 Agent 上下文不是免费的，大范围并行可能增加 DeepSeek API 成本；
- 弱模型返工可能抵消节省，因此只下放边界明确的任务。

---

## 九、安全与范围限制

必须遵守：

1. 不提交任何真实 API Key；
2. 不读取或打印现有 Codex 登录令牌；
3. 不覆盖整个 `~/.codex/config.toml`；
4. 不修改主 Agent 的默认 Provider；
5. 不把本地代理绑定到 `0.0.0.0`；
6. 不开放公网端口；
7. 默认不记录完整请求正文；
8. 如果开启调试日志，必须提醒日志可能包含源码和提示词；
9. 不自动执行 `git push`；
10. 不声称未完成的验证已经成功。

---

## 十、完成报告

完成后请输出：

1. 创建和修改的文件；
2. 选择的代理及固定版本/commit；
3. Codex Provider 配置方式；
4. 全局 Agent 配置方式；
5. 已运行的验证命令与真实结果；
6. 未验证内容；
7. 已知限制；
8. 用户接下来只需要执行的最少命令；
9. 如何确认 DeepSeek 子 Agent 确实被调用；
10. 如何安全回滚。

不要只给方案或伪代码。请完成仓库实现，做到用户克隆仓库后可以按 README 操作。