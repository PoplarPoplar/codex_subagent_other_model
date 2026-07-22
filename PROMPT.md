# Codex 执行提示词：直接完成 Linux Codex CLI 全局双子 Agent 配置

请在当前仓库中直接完成实现，并在用户填写凭证后，**由你自己把最终配置写入当前 Linux 用户的全局 Codex 目录**。

不要只给方案，不要把“复制模板、运行安装脚本、合并配置”等工作继续交给用户。

最终架构：

```text
Codex GPT-5.6 / ChatGPT 会员模型作为主 Agent
        ↓ 规划、拆分、调度、复核、最终测试
DeepSeek V4 Flash 全局子 Agent
        ↓ 代码搜索、日志分析、依赖定位、只读检查
GLM-5.2 全局子 Agent
        ↓ 边界明确的代码实现、测试、配置和局部修复
Codex GPT-5.6 主 Agent
        ↓ 审查 diff、重新测试、决定是否接受修改
```

---

## 一、最终目标

完成后，当前 Linux 用户应直接拥有以下全局配置：

```text
~/.codex/config.toml
~/.codex/agents/ark-v4flash-explorer.toml
~/.codex/agents/ark-glm52-worker.toml
~/.config/codex-ark-subagents/env
~/.local/bin/codex-ark
```

两个 Agent 必须能在当前用户的任意项目中使用。

严禁把最终生效配置放到：

```text
当前仓库/.codex/
任意业务项目/.codex/
```

这里的“全局”是指当前 Linux 用户级别，不是修改 `/etc`，不要使用 `sudo`，不要给其他用户安装。

---

## 二、你必须直接执行，不需要用户再运行安装脚本

本任务分为两个连续阶段。

### 阶段 A：立即自动完成

在没有真实凭证的情况下，直接完成：

1. 检查当前 Codex CLI 版本；
2. 查阅当前官方 Codex 配置和自定义子 Agent 文档；
3. 核对当前版本实际支持的字段、Agent 文件格式和 Provider 配置；
4. 在仓库中创建 README、模板、检查脚本、卸载脚本和测试提示词；
5. 实现可靠的 TOML 合并或修改逻辑；
6. 使用临时 HOME 做非破坏性 dry-run；
7. 验证重复执行不会产生重复配置；
8. 创建本地凭证文件：

```text
~/.config/codex-ark-subagents/env
```

9. 将其权限设为 `600`；
10. 文件中写入以下空占位符：

```bash
ARK_API_KEY=
ARK_V4FLASH_ENDPOINT_ID=
ARK_GLM52_ENDPOINT_ID=
```

11. 完成以上工作后，只暂停一次，告诉用户编辑该文件并回复：

```text
已填写
```

不要在阶段 A 中向用户索要 API Key，也不要要求用户运行 `install.sh`。

### 阶段 B：用户回复“已填写”后继续自动完成

用户回复“已填写”后，你必须自己：

1. 读取 `~/.config/codex-ark-subagents/env`；
2. 不得在终端或回复中打印完整 API Key；
3. 检查两个接入点 ID 都以 `ep-` 开头；
4. 检查两个接入点 ID 不相同；
5. 使用火山方舟 `/api/v3/responses` 分别测试两个接入点；
6. 测试普通非流式请求；
7. 测试 SSE 流式请求；
8. 在修改 `~/.codex/config.toml` 前创建带时间戳备份；
9. 安全合并统一 Provider；
10. 直接生成并写入两个全局 Agent 文件；
11. 创建 `~/.local/bin/codex-ark` 包装命令；
12. 让该包装命令读取私有 env 文件后再执行真实 `codex`；
13. 检查最终 TOML 是否可以解析；
14. 检查原有主模型、ChatGPT 登录和其他 Provider/MCP 配置没有被破坏；
15. 使用 `codex-ark` 做一次最小启动或非交互检查；
16. 能验证的内容直接验证，不能验证的内容明确标记为未验证；
17. 完成后只向用户汇报最终结果和以后如何使用。

**不要让用户再手动复制文件，不要让用户再运行安装命令。**

如果写入 `~/.codex/`、`~/.config/` 或 `~/.local/bin/` 需要沙箱批准，请主动发起一次必要的权限请求，然后继续执行；不要因为需要批准就退化成只给说明。

---

## 三、已确定的火山方舟方案

用户已经在火山方舟北京地域创建并授权两个自定义推理接入点：

1. DeepSeek V4 Flash；
2. GLM-5.2。

二者满足：

```text
同一个火山方舟账号
同一个项目
同一个北京地域
同一个 ARK_API_KEY
同一个 Responses API Base URL
不同的 ep-... 接入点 ID
```

统一 Provider 目标语义：

```toml
[model_providers.ark_reward]
name = "Volcengine Ark Reward Endpoints"
base_url = "https://ark.cn-beijing.volces.com/api/v3"
env_key = "ARK_API_KEY"
wire_api = "responses"
request_max_retries = 2
stream_max_retries = 2
stream_idle_timeout_ms = 300000
```

禁止实现以下旧方案：

```text
DeepSeek 官方 API
本地 Responses 转换代理
localhost:8787
Chat Completions 转 Responses
DEEPSEEK_API_KEY
Docker/MCP/Web 服务代理
```

---

## 四、必须先核对当前 Codex 格式

开始实现前，查阅执行时最新官方资料：

- https://developers.openai.com/codex/config-reference
- https://developers.openai.com/codex/config-advanced
- https://developers.openai.com/codex/agent-configuration/subagents

必须核对：

1. 用户级配置位置；
2. 用户级自定义 Agent 目录；
3. 自定义 Provider 字段；
4. `wire_api = "responses"` 的当前要求；
5. Agent 文件允许的字段；
6. `sandbox_mode` 和审批策略的当前合法值；
7. 当前 Codex CLI 是否自动发现 `~/.codex/agents/*.toml`；
8. 如果不是自动发现，应按官方当前格式在 `~/.codex/config.toml` 中注册 Agent，但最终仍必须是用户全局 Agent。

如果示例字段与当前版本不一致，以当前官方文档和本机 schema 为准修正。不要照抄无效字段。

---

## 五、仓库中需要创建的内容

至少创建：

```text
README.md
.gitignore
templates/config.fragment.toml
templates/ark-v4flash-explorer.toml
templates/ark-glm52-worker.toml
scripts/check.sh
scripts/uninstall.sh
scripts/validate_config.py
examples/test-readonly.md
examples/test-development.md
```

这里的脚本用于以后检查、卸载和复现，**不是要求用户在本次配置中再运行安装脚本**。

本次全局配置由当前 Codex 会话自己完成。

`.gitignore` 必须忽略：

```text
*.env
.env
user-config.env
*.log
*.bak
```

仓库中不得出现真实 API Key 或真实接入点 ID。

---

## 六、安全合并 `~/.codex/config.toml`

修改规则：

1. 文件不存在时创建；
2. 文件存在时先备份为：

```text
~/.codex/config.toml.bak-YYYYMMDD-HHMMSS
```

3. 不得覆盖整个文件；
4. 不得修改用户现有顶层 `model`；
5. 不得修改用户现有顶层 `model_provider`；
6. 不得修改 ChatGPT 登录配置；
7. 不得删除或破坏其他 Provider、Agent、MCP、通知、沙箱或项目配置；
8. 如果已有 `[model_providers.ark_reward]`，比较后幂等更新；
9. 不得产生重复 TOML section；
10. 修改后必须使用 Python 3.11+ `tomllib` 或可靠 TOML 库解析验证；
11. 失败时自动恢复备份；
12. `[agents]` 至少允许主线程和两个子线程，目标为：

```toml
[agents]
max_threads = 3
max_depth = 1
```

13. 如果用户已有更大的 `max_threads`，保留原值；
14. 保留 `[agents]` 中其他未知字段；
15. 不要凭字符串追加实现复杂 TOML 合并。

---

## 七、DeepSeek V4 Flash 全局探索 Agent

最终文件：

```text
~/.codex/agents/ark-v4flash-explorer.toml
```

目标语义如下，字段格式按当前 Codex 版本校正：

```toml
name = "ark_v4flash_explorer"

description = """
使用火山方舟 DeepSeek V4 Flash 接入点完成高频、边界明确的只读任务：
代码搜索、目录分析、日志分析、依赖定位、测试失败归纳和只读代码审查。
不负责最终架构、业务规则、安全决策或代码提交。
"""

nickname_candidates = [
  "Flash-Scout",
  "Flash-Tracer",
  "Flash-Mapper",
  "Flash-Reviewer"
]

model_provider = "ark_reward"
model = "用户填写的 ARK_V4FLASH_ENDPOINT_ID"

sandbox_mode = "read-only"
approval_policy = "untrusted"

developer_instructions = """
你是由 Codex 主 Agent 调度的只读探索子 Agent。

规则：
1. 每次只完成父 Agent 明确分配的一个边界清晰任务；
2. 优先精确搜索和定向读取，不做无边界扫描；
3. 返回真实文件路径、符号、代码位置和证据；
4. 不修改任何文件；
5. 不执行 git add、git commit 或 git push；
6. 不决定业务范围、整体架构、鉴权、事务、幂等和安全策略；
7. 不读取或输出 API Key、Token、凭证、真实客户数据和未脱敏文件；
8. 未运行的测试不得声称通过；
9. 工具失败时如实报告；
10. 最终结论由 GPT-5.6 主 Agent 复核。

返回：
- 结论摘要
- 检查过的文件和位置
- 证据
- 风险或不确定项
- 建议主 Agent 下一步验证
"""
```

该 Agent 必须保持只读。

---

## 八、GLM-5.2 全局开发 Agent

最终文件：

```text
~/.codex/agents/ark-glm52-worker.toml
```

目标语义如下，字段格式按当前 Codex 版本校正：

```toml
name = "ark_glm52_worker"

description = """
使用火山方舟 GLM-5.2 接入点完成边界明确的代码实现、测试、配置和局部故障修复。
适合 Java 后端、多文件修改、单元测试和集成测试候选实现。
最终架构决策、diff 审查、最终测试和提交由 Codex 主 Agent 负责。
"""

nickname_candidates = [
  "GLM-Builder",
  "GLM-Tester",
  "GLM-Fixer",
  "GLM-Implementer"
]

model_provider = "ark_reward"
model = "用户填写的 ARK_GLM52_ENDPOINT_ID"

sandbox_mode = "workspace-write"
approval_policy = "on-request"

developer_instructions = """
你是由 Codex 主 Agent 调度的开发执行子 Agent。

规则：
1. 每次只处理一个边界明确的任务；
2. 必须遵守当前项目的 AGENTS.md 和更具体规则；
3. 只修改父 Agent 明确允许的文件；
4. 不扩展任务范围；
5. 不擅自改变业务规则、架构、鉴权、事务、幂等和安全策略；
6. 不执行 git add、git commit 或 git push；
7. 不删除或覆盖其他人的未提交修改；
8. 运行父 Agent 指定的验证命令；
9. 未运行或失败的测试必须如实说明；
10. 不读取或输出密钥、Token 和真实客户数据；
11. 返回修改文件、实现说明、测试结果、风险和未确认事项；
12. 最终 diff、最终测试和提交由 GPT-5.6 主 Agent 负责。
"""
```

如果当前 Codex 版本或火山 Responses 工具调用尚未通过验证，第一轮可先将 GLM Agent 保持 `read-only`，完成只读验证后再由当前 Codex 会话自动切换到 `workspace-write`，不要让用户手动编辑。

---

## 九、私有凭证文件和启动包装命令

创建：

```text
~/.config/codex-ark-subagents/env
```

权限必须是：

```bash
chmod 600 ~/.config/codex-ark-subagents/env
```

文件只允许包含：

```bash
ARK_API_KEY='...'
ARK_V4FLASH_ENDPOINT_ID='ep-...'
ARK_GLM52_ENDPOINT_ID='ep-...'
```

不得把 Key 写入 `~/.codex/config.toml` 或 Agent TOML。

创建：

```text
~/.local/bin/codex-ark
```

功能：

1. 使用安全方式加载 env 文件；
2. 检查 `ARK_API_KEY` 已设置；
3. 不输出 Key；
4. 最后执行真实 `codex "$@"`；
5. 使用 `exec` 保留退出码和信号；
6. 权限设为可执行。

不要默认修改 `.bashrc`、`.zshrc` 或其他 shell 启动文件。

如果 `~/.local/bin` 不在 PATH，只在最终报告中给出一次 PATH 修复命令，不要擅自覆盖 shell 配置。

---

## 十、在线验证

用户回复“已填写”后，使用同一个 API Key 分别验证两个接入点。

Base URL：

```text
https://ark.cn-beijing.volces.com/api/v3
```

至少验证：

1. DeepSeek V4 Flash 非流式 Responses；
2. GLM-5.2 非流式 Responses；
3. DeepSeek V4 Flash SSE；
4. GLM-5.2 SSE；
5. 尽可能验证 Function Calling 或 Codex 需要的工具调用格式；
6. 检查 API 返回的模型/接入点信息与预期一致；
7. 不在日志中写请求源码、完整响应正文或 API Key；
8. 失败时输出状态码、错误类型和安全截断后的错误信息。

在线验证失败时：

- 不得声称配置成功；
- 不得删除用户原有 Codex 配置；
- 保留备份；
- 明确指出是 Key、Endpoint、Responses、SSE、Function Calling 还是 Codex Agent 格式问题；
- 尽可能自动修复后重试一次。

---

## 十一、完成后的 Codex 调用验证

最终至少提供并尽可能实际验证以下调用。

只读验证：

```text
保持当前 GPT-5.6 主 Agent。
调用 ark_v4flash_explorer 子 Agent，只读检查当前仓库的目录结构、任务入口和一个潜在风险。
不修改任何文件。子 Agent 完成后由主 Agent 复核路径和结论。
```

开发验证：

```text
保持当前 GPT-5.6 主 Agent。
先调用 ark_v4flash_explorer 搜索与当前任务相关的文件和风险。
再调用 ark_glm52_worker 完成一个边界明确的最小修改，不得提交代码。
最后由 GPT-5.6 主 Agent 检查 git diff、重新运行测试并决定是否接受修改。
```

验证成功标准：

1. 主线程仍使用原有 ChatGPT/Codex 登录；
2. V4 Flash 子 Agent 使用 V4 Flash 的 `ep-...`；
3. GLM 子 Agent 使用 GLM-5.2 的 `ep-...`；
4. 两个 Agent 都是当前用户全局可用；
5. V4 Flash 无写权限；
6. GLM 不执行提交；
7. 主 Agent 可以复核结果；
8. 火山方舟控制台能看到两个接入点各自用量。

---

## 十二、仓库文档要求

README 使用中文，至少说明：

1. 架构和职责分工；
2. 为什么使用一个 Provider、一个 Key、两个 Endpoint；
3. 最终全局文件位置；
4. 用户只需要编辑哪个凭证文件；
5. 如何使用 `codex-ark`；
6. 如何检查配置；
7. 如何安全卸载和恢复备份；
8. 如何确认两个子 Agent 走了正确模型；
9. Reward Plan 数据授权提示；
10. 不要发送密钥、真实客户数据和未脱敏内容；
11. 已验证和未验证项目；
12. 常见错误和排查方式。

`scripts/uninstall.sh` 必须只删除本方案创建的文件和 Provider/Agent 配置，不得删除整个 `~/.codex`，修改前必须备份。

---

## 十三、最终完成报告

完成后只汇报：

1. 仓库中创建或修改的文件；
2. 写入当前用户全局目录的文件；
3. 创建的备份；
4. 实际运行的离线和在线验证；
5. 两个接入点是否均成功；
6. Codex 两个全局子 Agent 是否被识别；
7. 哪些项目仍未验证；
8. 用户以后启动时使用的唯一命令，例如：

```bash
codex-ark
```

不要再让用户运行安装脚本，也不要让用户手动复制模板到 `~/.codex/`。
