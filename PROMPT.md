# Codex 执行提示词：在 Linux Codex CLI 中安装火山方舟全局双子 Agent

请在当前仓库中实现一套**可复现、可安装、可检查、可卸载，并且不会破坏用户现有 Codex 配置**的方案。

最终目标是：

```text
Codex GPT-5.6 / ChatGPT 会员模型作为主 Agent
        ↓ 规划、拆分、调度、复核、最终测试
DeepSeek V4 Flash 子 Agent
        ↓ 代码搜索、日志分析、依赖定位、只读检查
GLM-5.2 子 Agent
        ↓ 边界明确的代码实现、测试、配置和局部修复
Codex GPT-5.6 主 Agent
        ↓ 审查 diff、重新测试、决定是否接受修改
```

不要只输出说明或伪代码。请在当前仓库中实际创建脚本、模板和文档，并运行当前环境允许执行的非破坏性验证。

---

## 一、最重要的安装范围

这里所说的“系统级别”是指：

> 安装到当前 Linux 用户的全局 Codex 配置中，使两个子 Agent 能在该用户的任意项目中使用。

安装目标必须是：

```text
~/.codex/config.toml
~/.codex/agents/ark-v4flash-explorer.toml
~/.codex/agents/ark-glm52-worker.toml
```

严禁把最终 Agent 安装到：

```text
当前仓库/.codex/agents/
任意业务项目/.codex/agents/
当前仓库/.codex/config.toml
任意业务项目/.codex/config.toml
```

仓库中可以保存模板和安装脚本，但最终生效位置只能是当前用户的 `~/.codex/`。

不要使用 `sudo`，不要修改 `/etc`，不要尝试给其他 Linux 用户安装。

---

## 二、已确定的技术方案

用户已经在火山方舟北京地域创建并授权了两个自定义推理接入点：

1. DeepSeek V4 Flash 接入点；
2. GLM-5.2 接入点。

二者满足以下关系：

```text
同一个火山方舟账号
同一个项目
同一个北京地域
同一个 ARK_API_KEY
同一个 Responses API Base URL
不同的 ep-... 接入点 ID
```

统一 Provider：

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
Python 协议转换服务
DEEPSEEK_API_KEY
systemd 代理服务
Chat Completions 转 Responses
```

火山方舟已经提供原生 Responses API，因此不需要任何本地代理。

---

## 三、开始前必须核对

开始实现前，请查阅执行时最新的官方资料：

- Codex 配置参考：
  - https://developers.openai.com/codex/config-reference
  - https://developers.openai.com/codex/config-advanced
- Codex 自定义子 Agent：
  - https://developers.openai.com/codex/agent-configuration/subagents
- 火山方舟 Responses API：
  - https://www.volcengine.com/docs/82379/1795150

必须确认当前 Codex 版本实际支持的字段名称和取值。

如果下面示例中的某个字段已变化，应以当前官方文档和本机 Codex 配置 schema 为准进行修正，但不得改变本提示词规定的总体架构和权限边界。

---

## 四、输入参数和密钥

安装方案需要使用三个用户输入：

```bash
ARK_API_KEY
ARK_V4FLASH_ENDPOINT_ID
ARK_GLM52_ENDPOINT_ID
```

示例：

```bash
export ARK_API_KEY="用户自己的火山方舟 API Key"
export ARK_V4FLASH_ENDPOINT_ID="ep-xxxxxxxxxxxxxxxx"
export ARK_GLM52_ENDPOINT_ID="ep-yyyyyyyyyyyyyyyy"
```

要求：

1. 不得把真实 API Key 写入仓库；
2. 不得把真实 API Key 写进 `~/.codex/config.toml`；
3. Provider 只能引用环境变量名 `ARK_API_KEY`；
4. 安装脚本不得打印完整 API Key；
5. 接入点 ID 可以作为安装参数写入 Agent 文件，但仓库模板中必须使用占位符；
6. 安装脚本必须检查两个接入点 ID 都以 `ep-` 开头；
7. 安装脚本必须防止把同一个接入点 ID 同时配置给两个 Agent；
8. README 必须说明如何安全地长期提供 `ARK_API_KEY`，但不得默认修改用户的 shell 配置文件。

---

## 五、仓库最终结构

至少创建：

```text
README.md
.gitignore
scripts/install.sh
scripts/check.sh
scripts/uninstall.sh
templates/config.fragment.toml
templates/ark-v4flash-explorer.toml
templates/ark-glm52-worker.toml
examples/test-readonly.md
examples/test-development.md
```

可以增加少量必要辅助脚本，但不要引入 Docker、数据库、Web 服务、MCP Server 或本地代理。

---

## 六、全局 Provider 配置要求

安装脚本需要把以下 Provider 以安全、幂等方式合并到：

```text
~/.codex/config.toml
```

目标语义：

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

同时需要保证 `[agents]` 至少允许主 Agent 加两个子 Agent：

```toml
[agents]
max_threads = 3
max_depth = 1
```

配置合并规则：

1. 修改前创建带时间戳的备份；
2. 不得覆盖整个 `~/.codex/config.toml`；
3. 不得修改用户现有顶层 `model`；
4. 不得修改用户现有顶层 `model_provider`；
5. 不得修改 ChatGPT 登录配置；
6. 不得删除其他 Provider、MCP、项目、通知或沙箱配置；
7. 如果 `[agents]` 已存在，保留其他字段；
8. 如果用户已有更大的 `max_threads`，保留现值；
9. 如果用户已有更严格的 `max_depth = 1`，保持不变；
10. 如果已存在同名 `ark_reward` Provider，先比较内容；
11. 内容相同则不重复写入；
12. 内容不同则备份并明确提示用户，再安全更新；
13. 重复执行安装脚本必须保持幂等。

不要使用简单字符串追加导致重复 TOML section。应使用可靠的 TOML 修改方式，并验证修改后的文件能被解析。

---

## 七、DeepSeek V4 Flash 全局探索 Agent

仓库模板：

```text
templates/ark-v4flash-explorer.toml
```

安装位置：

```text
~/.codex/agents/ark-v4flash-explorer.toml
```

至少实现以下语义：

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
model = "__ARK_V4FLASH_ENDPOINT_ID__"

sandbox_mode = "read-only"
approval_policy = "untrusted"

developer_instructions = """
你是由 Codex 主 Agent 调度的只读探索子 Agent。

规则：
1. 每次只完成父 Agent 明确分配的一个边界清晰任务；
2. 父 Agent 的任务说明就是你的主要交接上下文；
3. 优先精确搜索和定向读取，不进行无边界全仓库扫描；
4. 返回真实文件路径、符号、代码位置和证据；
5. 不修改文件；
6. 不执行 git add、git commit、git push；
7. 不决定业务范围、整体架构、鉴权、事务、幂等和安全策略；
8. 不读取或输出 API Key、Token、凭证、真实客户数据和未脱敏文件；
9. 不声称未运行的测试已经通过；
10. 工具调用失败时如实报告，不得编造结果；
11. 最终结论由 Codex GPT-5.6 主 Agent 复核。

返回格式：
- 结论摘要
- 检查过的文件和位置
- 证据
- 风险或不确定项
- 建议主 Agent 下一步验证
"""
```

如果当前 Codex 版本不支持示例中的某个字段，应按官方 schema 修正，但必须保持该 Agent 为 `read-only`。

---

## 八、GLM-5.2 全局开发 Agent

仓库模板：

```text
templates/ark-glm52-worker.toml
```

安装位置：

```text
~/.codex/agents/ark-glm52-worker.toml
```

至少实现以下语义：

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
model = "__ARK_GLM52_ENDPOINT_ID__"

sandbox_mode = "workspace-write"
approval_policy = "on-request"

developer_instructions = """
你是由 Codex 主 Agent 调度的开发执行子 Agent。

规则：
1. 每次只处理一个边界明确的任务；
2. 必须遵守当前项目中的 AGENTS.md 和更具体的项目规则；
3. 只修改父 Agent 明确允许的文件；
4. 不扩展任务范围；
5. 不擅自改变业务需求、系统架构、鉴权、租户隔离、事务、幂等和安全策略；
6. 不执行 git add、git commit、git push；
7. 不删除或覆盖其他人的未提交修改；
8. 必须运行父 Agent 指定的验证命令；
9. 未运行或失败的测试必须明确说明；
10. 不读取或输出 API Key、Token、凭证、真实客户数据和未脱敏文件；
11. 完成后返回修改文件、实现摘要、测试结果、风险和待确认事项；
12. 最终 diff 审查、修正、提交由 Codex GPT-5.6 主 Agent 负责。
"""
```

如果当前 Codex 版本不支持示例中的某个字段，应按官方 schema 修正。

不要把 GLM Agent 改成可以自动提交代码的角色。

---

## 九、安装脚本要求

`scripts/install.sh` 必须：

1. 使用 `set -euo pipefail`；
2. 检查当前系统是 Linux；
3. 检查 `codex` 命令存在；
4. 检查 `ARK_API_KEY` 已设置，但不输出值；
5. 检查两个 Endpoint ID 已提供；
6. 支持从环境变量读取 Endpoint ID；
7. 可以额外支持命令行参数，但不得要求用户修改模板；
8. 创建 `~/.codex/agents/`；
9. 修改配置前备份 `~/.codex/config.toml`；
10. 安全合并 Provider 和 `[agents]`；
11. 用真实 Endpoint ID 替换两个模板中的占位符；
12. 原子写入两个全局 Agent 文件；
13. 设置合理文件权限；
14. 安装后验证 TOML 文件可解析；
15. 不调用 `sudo`；
16. 不创建项目级 `.codex`；
17. 不修改主模型；
18. 不自动执行 `git push`；
19. 输出下一步检查命令；
20. 重复安装不产生重复配置。

建议支持：

```bash
ARK_API_KEY=... \
ARK_V4FLASH_ENDPOINT_ID=ep-... \
ARK_GLM52_ENDPOINT_ID=ep-... \
./scripts/install.sh
```

---

## 十、检查脚本要求

`scripts/check.sh` 至少检查：

1. `codex --version`；
2. `ARK_API_KEY` 是否存在，但不得输出值；
3. 两个 Endpoint ID 是否可获得；
4. `~/.codex/config.toml` 是否可解析；
5. `ark_reward` Provider 是否存在且 Base URL 正确；
6. 主 Codex 顶层 Provider 没有被改成 `ark_reward`；
7. 两个全局 Agent 文件是否存在；
8. 两个 Agent 是否引用不同的 Endpoint ID；
9. V4 Flash Agent 是否为 `read-only`；
10. GLM Agent 是否为 `workspace-write`；
11. 对 V4 Flash Endpoint 执行最小非流式 Responses 请求；
12. 对 GLM-5.2 Endpoint 执行最小非流式 Responses 请求；
13. 分别执行 SSE 流式 Responses 请求；
14. 尽可能验证一次简单 Function Calling；
15. 失败时输出明确原因和修复建议；
16. 返回正确的 shell exit code。

检查结果必须区分：

```text
静态配置通过
V4 Flash 普通 Responses 通过/失败
V4 Flash SSE 通过/失败
GLM-5.2 普通 Responses 通过/失败
GLM-5.2 SSE 通过/失败
Function Calling 通过/失败/未验证
Codex 原生子 Agent 调度尚需人工验证
```

不要把一次普通文本成功当作完整 Codex 兼容性验证。

---

## 十一、卸载脚本要求

`scripts/uninstall.sh` 必须：

1. 使用 `set -euo pipefail`；
2. 修改前备份 `~/.codex/config.toml`；
3. 只删除：
   - `~/.codex/agents/ark-v4flash-explorer.toml`
   - `~/.codex/agents/ark-glm52-worker.toml`
   - 本方案添加的 `model_providers.ark_reward`
4. 不删除整个 `~/.codex`；
5. 不删除其他 Agent；
6. 不删除其他 Provider；
7. 不修改用户主模型；
8. 不删除 `ARK_API_KEY`；
9. 对文件不存在和重复卸载保持幂等；
10. 卸载后验证剩余 TOML 仍可解析。

对于 `[agents]` 中的 `max_threads` 和 `max_depth`：

- 如果无法可靠判断是否由本方案首次添加，不要擅自删除；
- README 中说明这两个通用字段可能保留。

---

## 十二、README 要求

README 使用中文，至少说明：

1. 这套方案解决什么问题；
2. 架构图；
3. 为什么只需要一个 Provider 和一个 API Key；
4. 为什么需要两个不同 Endpoint ID；
5. 全局安装和项目级安装的区别；
6. 最终文件安装到哪里；
7. 前置条件；
8. 如何设置三个环境变量；
9. 一键安装；
10. 一键检查；
11. 如何卸载；
12. 如何在任意项目调用两个全局 Agent；
13. 如何确认主 Agent 仍使用 GPT-5.6；
14. 如何在火山控制台确认两个子 Agent 的调用量；
15. Reward Plan 数据授权的注意事项；
16. 不要向授权接入点发送密钥、真实客户数据、未脱敏日志和受限私有代码；
17. Responses、SSE、Function Calling 的验证状态；
18. 已知限制；
19. 常见错误：
   - 401/403；
   - Endpoint ID 不存在；
   - Endpoint 未授权；
   - `/responses` 请求失败；
   - SSE 中断；
   - 工具调用格式不兼容；
   - Codex 找不到自定义 Agent；
   - 子 Agent 意外继承主 Provider；
   - GLM Agent 无法写入工作区；
20. 如何安全回滚。

必须明确：

```text
GPT-5.6 主 Agent 仍然会消耗 Codex/ChatGPT 额度用于规划、调度、复核和最终回答。
V4 Flash 与 GLM-5.2 的调用走火山方舟 API。
并行子 Agent 越多，火山 Token 消耗越高。
弱模型返工可能抵消节省，因此只下放边界明确的任务。
```

---

## 十三、示例提示词

### `examples/test-readonly.md`

提供一个可以直接粘贴到 Codex CLI 的测试提示词：

```text
保持当前 GPT-5.6 主 Agent，不要切换主模型。

调用 ark_v4flash_explorer 子 Agent，只读检查当前仓库：
1. 列出顶层目录及用途；
2. 找出 README、TODO、AGENTS.md 或主要任务入口；
3. 指出一个需要主 Agent 进一步确认的风险；
4. 不修改任何文件。

等待子 Agent 完成后，由主 Agent核对其文件路径和结论是否真实，
并说明本次子任务是否使用了 ark_reward Provider。
```

### `examples/test-development.md`

提供一个安全的双 Agent 测试提示词：

```text
保持当前 GPT-5.6 主 Agent，不要切换主模型。

先调用 ark_v4flash_explorer 子 Agent，只读分析当前仓库中一个边界明确的小任务，
返回相关文件、现状、风险和建议验证方式。

主 Agent 审核探索结果后，再调用 ark_glm52_worker 子 Agent：
1. 只完成主 Agent 明确指定的小任务；
2. 只修改允许的文件；
3. 不执行 git add、commit、push；
4. 运行指定测试；
5. 如实报告结果。

两个子 Agent 完成后，由 GPT-5.6 主 Agent：
1. 检查完整 git diff；
2. 检查是否超出范围；
3. 重新运行真实测试；
4. 修复残留问题；
5. 决定是否接受修改。
```

---

## 十四、实际验证要求

实现完成后按顺序验证：

### 静态验证

- Shell 脚本语法；
- TOML 模板格式；
- 配置合并逻辑；
- 重复安装；
- 卸载；
- 重复卸载；
- `.gitignore` 不遗漏密钥和临时文件。

### API 验证

在环境变量可用时验证：

- V4 Flash 非流式 Responses；
- V4 Flash SSE；
- GLM-5.2 非流式 Responses；
- GLM-5.2 SSE；
- 简单 Function Calling。

### Codex CLI 人工验证

提供明确步骤，让用户确认：

1. Codex 显示创建了 `ark_v4flash_explorer`；
2. Codex 显示创建了 `ark_glm52_worker`；
3. 主线程仍为原有 GPT-5.6 / ChatGPT 登录；
4. 火山控制台两个接入点分别出现调用量；
5. V4 Flash Agent 没有修改文件；
6. GLM Agent 能在受控范围内修改文件；
7. 主 Agent 可以审查结果。

未实际执行的验证必须明确写“未验证”，不得声称成功。

---

## 十五、安全要求

必须遵守：

1. 不提交真实 API Key；
2. 不读取或打印 Codex 登录令牌；
3. 不覆盖整个 `~/.codex/config.toml`；
4. 不改变主 Agent 默认 Provider；
5. 不创建本地代理；
6. 不开放任何监听端口；
7. 不自动修改 shell 启动文件；
8. 不自动执行 `git push`；
9. 不把 Endpoint ID 当成秘密，但不要在模板中写死用户真实 ID；
10. 不声称未完成的验证已经成功；
11. 对 Reward Plan 已授权接入点，只发送用户明确允许的数据；
12. 安装脚本发现疑似错误配置时应停止，而不是强行覆盖。

---

## 十六、完成报告

完成实现后，请输出：

1. 创建和修改的文件；
2. 全局安装目标；
3. Provider 配置方式；
4. 两个 Agent 的职责和权限；
5. 安装命令；
6. 检查命令；
7. 卸载命令；
8. 已运行的验证及真实结果；
9. 未验证内容；
10. 已知限制；
11. 用户接下来最少需要执行的命令；
12. 如何确认两个子 Agent 确实分别调用了对应火山接入点；
13. 如何确认 GPT-5.6 主 Agent 没有被替换；
14. 如何安全回滚。

不要只给建议。请把当前仓库实现成用户克隆后能够按照 README 安装的工具。