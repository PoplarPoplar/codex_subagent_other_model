# Codex 执行提示词：全自动实现 Linux Codex CLI 全局双子 Agent

请在当前仓库中直接完成一套**可复现、可安装、可检查、可卸载，并且不会破坏用户现有 Codex 配置**的实现。

不要只给方案、步骤或伪代码。你要直接创建并完善仓库中的脚本、模板、示例和 README，运行当前环境能够执行的非破坏性验证，直到仓库达到“用户只需要填写 1 个 API Key 和 2 个接入点 ID，就能一键安装”的状态。

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

## 一、执行方式：不要在实现过程中向用户索要密钥

本任务分成两个阶段。

### 阶段 A：你现在必须自动完成

在没有真实 API Key 和接入点 ID 的情况下，直接完成：

1. 仓库结构设计；
2. 全部脚本实现；
3. 全部 Agent 模板；
4. Codex Provider 配置合并逻辑；
5. 安装、检查和卸载逻辑；
6. 本地用户配置文件模板；
7. README 和可复制测试提示词；
8. Shell、Python、TOML 模板和幂等逻辑的静态验证；
9. 不需要真实 API 的 dry-run 验证；
10. 明确记录哪些在线验证因为没有真实凭证而未执行。

**不要因为缺少以下三个值而停下来询问用户：**

```text
ARK_API_KEY
ARK_V4FLASH_ENDPOINT_ID
ARK_GLM52_ENDPOINT_ID
```

仓库模板中统一使用明确占位符。

### 阶段 B：实现完成后用户才操作

用户最后只需要：

1. 复制并编辑一个被 `.gitignore` 忽略的本地配置文件；
2. 填写 1 个火山方舟 API Key；
3. 填写 DeepSeek V4 Flash 和 GLM-5.2 的两个 `ep-...` 接入点 ID；
4. 运行一条安装命令；
5. 运行一条检查命令；
6. 重启或重新打开 Codex CLI。

完成报告中只能把这几步留给用户，不得把仓库实现工作继续甩给用户。

---

## 二、最重要的安装范围

这里的“系统级别”准确含义是：

> 安装到当前 Linux 用户的全局 Codex 配置中，使两个自定义子 Agent 能在该用户的任意项目中使用。

最终生效位置必须是：

```text
~/.codex/config.toml
~/.codex/agents/ark-v4flash-explorer.toml
~/.codex/agents/ark-glm52-worker.toml
```

允许创建当前用户私有辅助文件，例如：

```text
~/.config/codex-ark-subagents/env
~/.local/bin/codex-ark
```

严禁把最终 Agent 安装到：

```text
当前仓库/.codex/agents/
任意业务项目/.codex/agents/
当前仓库/.codex/config.toml
任意业务项目/.codex/config.toml
```

仓库中只能保存模板和安装工具，最终生效配置必须位于当前用户的 `~/.codex/`。

不要使用 `sudo`，不要修改 `/etc`，不要给其他 Linux 用户安装。

---

## 三、已经确定的技术方案

用户已经在火山方舟北京地域创建并授权：

1. DeepSeek V4 Flash 自定义推理接入点；
2. GLM-5.2 自定义推理接入点。

二者满足：

```text
同一个火山方舟账号
同一个项目
同一个北京地域
同一个 ARK_API_KEY
同一个 Responses API Base URL
不同的 ep-... 接入点 ID
```

统一 Codex Provider 的目标语义：

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
DEEPSEEK_API_KEY
本地 Responses 转换代理
localhost:8787
Python 协议转换服务
systemd 代理服务
Chat Completions 转 Responses
```

火山方舟已经提供原生 Responses API，不需要本地协议代理。

---

## 四、开始前必须核对最新官方资料

实现前查阅执行时最新资料：

- Codex 配置参考：
  - https://developers.openai.com/codex/config-reference
  - https://developers.openai.com/codex/config-advanced
- Codex 自定义子 Agent：
  - https://developers.openai.com/codex/agent-configuration/subagents
- 火山方舟 Responses API：
  - https://www.volcengine.com/docs/82379/1795150

必须核对当前 Codex CLI 版本及配置 schema，确认：

1. 自定义 Provider 的字段名称；
2. `wire_api = "responses"` 的当前要求；
3. 自定义 Agent 文件格式；
4. 沙箱和审批字段当前有效取值；
5. 用户级 Agent 的发现目录；
6. 多 Agent 并发配置字段。

示例字段若已变化，应按当前官方文档和本机 schema 修正，但不得改变本提示词规定的架构、全局安装范围和权限边界。

---

## 五、仓库最终结构

至少创建：

```text
README.md
PROMPT.md
.gitignore
user-config.env.example
scripts/install.sh
scripts/check.sh
scripts/uninstall.sh
scripts/render_agents.py
scripts/merge_codex_config.py
templates/config.fragment.toml
templates/ark-v4flash-explorer.toml
templates/ark-glm52-worker.toml
examples/test-readonly.md
examples/test-development.md
```

允许增加少量必要测试或辅助文件，但不要引入：

```text
Docker
数据库
Web 服务
MCP Server
本地模型代理
常驻后台服务
```

### 用户私有配置文件

仓库必须提供：

```text
user-config.env.example
```

内容至少为：

```bash
ARK_API_KEY=""
ARK_V4FLASH_ENDPOINT_ID=""
ARK_GLM52_ENDPOINT_ID=""
```

README 要求用户复制为：

```text
user-config.env
```

并填写真实值。`.gitignore` 必须忽略：

```text
user-config.env
*.local.env
.env
.env.*
```

但不能忽略 `user-config.env.example`。

安装脚本默认从仓库根目录的 `user-config.env` 读取，也应允许使用：

```bash
./scripts/install.sh --config /path/to/private.env
```

---

## 六、输入校验和密钥安全

安装时读取：

```text
ARK_API_KEY
ARK_V4FLASH_ENDPOINT_ID
ARK_GLM52_ENDPOINT_ID
```

必须做到：

1. 不把真实 API Key 写入 Git；
2. 不把真实 API Key 写进 `~/.codex/config.toml`；
3. Provider 只引用环境变量名 `ARK_API_KEY`；
4. 不在终端输出完整 Key；
5. 接入点 ID 必须以 `ep-` 开头；
6. 两个 Agent 不允许使用同一个接入点 ID；
7. 空值、占位符、带换行或明显格式错误时立即失败；
8. 私有环境文件复制到 `~/.config/codex-ark-subagents/env` 时权限必须为 `600`；
9. 生成的 Agent 文件中可以保存接入点 ID，但不能保存 API Key；
10. 检查脚本只显示 Key 是否存在，不能显示具体值。

为了让用户不必每次手动 `export`，安装脚本应创建：

```text
~/.local/bin/codex-ark
```

该包装器只能：

1. 安全加载 `~/.config/codex-ark-subagents/env`；
2. 导出 `ARK_API_KEY`；
3. 使用 `exec codex "$@"` 启动用户现有 Codex；
4. 不修改 Codex 主模型；
5. 不打印 Key。

README 同时说明：

- 使用 `codex-ark` 最省事；
- 用户也可以手动导出 `ARK_API_KEY` 后继续使用原来的 `codex` 命令；
- 如果 `~/.local/bin` 不在 PATH，如何临时或永久加入 PATH。

不得自动修改 `.bashrc`、`.zshrc` 或其他 shell 启动文件。

---

## 七、全局 Provider 配置合并

安装脚本必须安全、幂等地修改：

```text
~/.codex/config.toml
```

目标 Provider：

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

并确保多 Agent 配置至少满足当前主线程加两个子线程，例如当前 schema 对应语义：

```toml
[agents]
max_threads = 3
max_depth = 1
```

配置合并规则：

1. 修改前创建带时间戳备份；
2. 不覆盖整个 `config.toml`；
3. 不修改用户顶层 `model`；
4. 不修改用户顶层 `model_provider`；
5. 不修改 ChatGPT 登录；
6. 不删除其他 Provider、MCP、通知、沙箱或项目配置；
7. `[agents]` 已存在时保留其他字段；
8. 已有更大的并发值时保留用户值；
9. `max_depth` 保持不允许子 Agent 继续无限嵌套的安全值；
10. 同名 Provider 内容相同则不重复写入；
11. 同名 Provider 内容冲突时先备份，再明确提示并安全更新；
12. 重复安装必须幂等；
13. 合并后必须通过 TOML 解析；
14. 不能用简单字符串无脑追加造成重复 section。

优先使用一个小型、可审计的 Python 合并工具。可以使用 Python 标准库读取 TOML；如确实需要第三方 TOML 写入库，应安装到本仓库或用户目录的隔离虚拟环境，并在 README 中说明，不得污染系统 Python。

---

## 八、DeepSeek V4 Flash 全局探索 Agent

模板：

```text
templates/ark-v4flash-explorer.toml
```

安装位置：

```text
~/.codex/agents/ark-v4flash-explorer.toml
```

目标语义：

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

即使当前 Codex schema 字段名称有变化，该 Agent 也必须保持只读。

---

## 九、GLM-5.2 全局开发 Agent

模板：

```text
templates/ark-glm52-worker.toml
```

安装位置：

```text
~/.codex/agents/ark-glm52-worker.toml
```

目标语义：

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
2. 必须遵守当前项目的 AGENTS.md 和更具体规则；
3. 只修改父 Agent 明确允许的文件；
4. 不扩展任务范围；
5. 不擅自改变业务需求、整体架构、鉴权、事务、幂等和安全策略；
6. 不执行 git add、git commit、git push；
7. 不删除或覆盖不属于本任务的修改；
8. 运行父 Agent 指定的验证；
9. 未运行或失败的测试必须如实说明；
10. 不读取或输出 API Key、Token、凭证和真实客户数据；
11. 最终 diff、架构决策、最终测试和提交由 Codex GPT-5.6 主 Agent 负责。

完成后返回：
- 修改的文件
- 实现摘要
- 执行的命令
- 测试结果
- 风险和未确认事项
"""
```

如果当前 Codex schema 不支持示例审批字段，按官方 schema 使用最接近且安全的值。不得取消“禁止自行提交”和“只能修改指定文件”的边界。

---

## 十、安装脚本

`scripts/install.sh` 必须：

1. 使用 `set -euo pipefail`；
2. 支持 `--config` 和 `--dry-run`；
3. 默认读取 `./user-config.env`；
4. 检查 Linux、`codex`、Python 和必要依赖；
5. 不使用 sudo；
6. 验证三个用户输入；
7. 创建所有目标目录；
8. 修改前备份 `~/.codex/config.toml`；
9. 安全合并 Provider 和 Agent 并发配置；
10. 渲染两个 Agent 模板中的接入点 ID；
11. 安装到 `~/.codex/agents/`；
12. 私有环境文件权限设为 `600`；
13. 创建 `~/.local/bin/codex-ark`；
14. 不修改主 Agent 模型和登录；
15. 安装结束后自动运行本地静态检查；
16. 有真实凭证时允许执行在线连通性检查；
17. 重复执行保持幂等；
18. 失败时尽可能不留下半安装状态，并给出备份位置和恢复方法。

`--dry-run` 必须不修改用户目录，只显示将执行的动作，并且不能显示完整 API Key。

---

## 十一、检查脚本

`scripts/check.sh` 至少支持：

```bash
./scripts/check.sh --config user-config.env
./scripts/check.sh --offline
```

离线检查：

1. `codex --version`；
2. 仓库文件是否完整；
3. Shell 语法；
4. Python 脚本语法；
5. TOML 模板可解析；
6. 占位符是否正确；
7. `~/.codex/config.toml` 可解析；
8. Provider 是否存在且没有改写主 Provider；
9. 两个全局 Agent 是否存在；
10. 两个 Agent 的 Endpoint ID 不同；
11. V4 Flash Agent 保持只读；
12. GLM Agent 不允许自行提交；
13. 环境文件权限是否为 `600`；
14. `codex-ark` 包装器是否存在且不泄露 Key。

在线检查：

1. `ARK_API_KEY` 是否可加载，但不显示值；
2. V4 Flash 接入点最小非流式 Responses 请求；
3. GLM-5.2 接入点最小非流式 Responses 请求；
4. 两个接入点的 SSE 流式请求；
5. 至少一次接近 Codex 工具调用格式的 Function Calling 验证；
6. 清晰区分普通文本、SSE、Function Calling 和 Codex 原生子 Agent 尚未验证的状态；
7. 返回正确退出码。

不得把“普通文本返回成功”描述成“Codex 子 Agent 完整兼容已验证”。

---

## 十二、卸载脚本

`scripts/uninstall.sh` 必须：

1. 使用 `set -euo pipefail`；
2. 支持 `--dry-run`；
3. 修改前备份 `~/.codex/config.toml`；
4. 只删除本方案安装的两个 Agent；
5. 只删除 `ark_reward` Provider；
6. 只删除本方案创建的 `~/.config/codex-ark-subagents/`；
7. 只删除本方案创建的 `~/.local/bin/codex-ark`；
8. 不删除整个 `~/.codex`；
9. 不影响 GPT-5.6 主 Agent；
10. 不影响其他 Provider、Agent、MCP 和项目配置；
11. 重复卸载保持幂等；
12. 卸载后再次验证 TOML 可解析。

---

## 十三、README 要求

README 使用中文，至少包括：

1. 架构图；
2. 为什么一个 Provider 可以服务两个接入点；
3. 为什么不需要本地代理；
4. 系统级/用户级安装范围；
5. 明确说明不是项目级 Agent；
6. 前置条件；
7. 用户最后只需执行的最少步骤；
8. 如何复制并编辑 `user-config.env`；
9. 如何安装、检查、启动 Codex；
10. `codex-ark` 和原始 `codex` 的区别；
11. 如何确认两个子 Agent 分别走了正确接入点；
12. 如何查看火山方舟用量和 Reward Plan 统计；
13. 如何调用只读探索 Agent；
14. 如何让 GLM 开发 Agent 修改代码；
15. 为什么最终仍需 GPT-5.6 审查 diff 和重跑测试；
16. 如何卸载和回滚；
17. 常见错误及处理：401/403、Endpoint 不存在、Responses 404、SSE 失败、Function Calling 不兼容、Agent 未被发现、环境变量缺失、`~/.local/bin` 不在 PATH；
18. 数据授权和敏感信息风险；
19. 已验证版本矩阵；
20. 已知限制。

README 开头提供最短路径：

```bash
cp user-config.env.example user-config.env
nano user-config.env
chmod 600 user-config.env
./scripts/install.sh --config user-config.env
./scripts/check.sh --config user-config.env
codex-ark
```

---

## 十四、示例测试提示词

`examples/test-readonly.md` 提供可直接粘贴到 Codex 的测试：

```text
保持当前 GPT-5.6 主 Agent，不要切换主模型。

调用 ark_v4flash_explorer 子 Agent，只读检查当前仓库：
1. 列出顶层目录和用途；
2. 找出 README、TODO 或任务入口；
3. 指出一个需要主 Agent 继续确认的风险；
4. 不修改任何文件。

等待子 Agent 完成后，由 GPT-5.6 主 Agent 复核文件路径和结论，
并说明子任务是否由 ark_reward Provider 执行。
```

`examples/test-development.md` 提供可直接粘贴的开发测试：

```text
保持当前 GPT-5.6 主 Agent，负责目标理解、拆分、审查和最终测试。

先调用 ark_v4flash_explorer，只读定位当前最小待办任务、相关文件和风险。
然后调用 ark_glm52_worker，只完成主 Agent 明确允许的最小修改，禁止提交代码。

子 Agent 完成后，主 Agent必须：
1. 检查完整 git diff；
2. 检查是否超出任务范围；
3. 重新运行真实测试；
4. 修正残留问题；
5. 最终汇报两个子 Agent 分别完成了什么。
```

---

## 十五、你现在必须执行的验证

在没有真实凭证时，至少实际执行：

1. `bash -n` 检查所有 Shell 脚本；
2. Python 编译检查；
3. TOML 模板解析；
4. 使用临时 HOME 的安装 dry-run；
5. 使用临时 HOME 和虚构但格式有效的 Endpoint ID 做离线安装；
6. 重复安装幂等验证；
7. 卸载与重复卸载验证；
8. 验证不会创建项目级 `.codex/agents/`；
9. 验证不会覆盖主模型配置；
10. 验证 `.gitignore` 可以拦截 `user-config.env`；
11. 验证日志不会输出完整测试 Key。

不得向真实火山 API 发送虚构凭证。没有真实凭证时，把在线检查标记为“待用户填写后执行”。

---

## 十六、安全边界

必须遵守：

1. 不提交真实 API Key；
2. 不读取或打印 Codex 登录令牌；
3. 不覆盖整个 `~/.codex/config.toml`；
4. 不修改主 Agent 默认 Provider；
5. 不自动修改 shell rc 文件；
6. 不创建公网服务；
7. 不记录完整 API 请求正文；
8. 不自动执行 git push；
9. 不声称未完成的在线验证已经成功；
10. Reward Plan 接入点不得用于密钥、真实客户数据、未脱敏日志或不允许外传的代码。

---

## 十七、完成标准和最终报告

只有满足以下条件才算完成：

1. 仓库文件已经实际创建；
2. 安装脚本能够把配置安装到当前用户全局 `~/.codex/`；
3. 仓库中没有项目级生效 Agent；
4. 用户只需填写一个本地配置文件；
5. 用户只需运行安装、检查和启动命令；
6. 不需要用户手工编写 TOML；
7. 不需要用户手工合并 `~/.codex/config.toml`；
8. 不需要用户自己实现脚本；
9. 静态和离线测试已经真实运行；
10. 未执行的在线验证已明确标记。

完成后只向用户汇报：

1. 创建和修改的文件；
2. 实际运行的验证及结果；
3. 未验证内容；
4. 用户最后需要执行的最少命令；
5. 如何确认两个子 Agent 确实走火山方舟接入点；
6. 如何安全回滚。

不要在完成前停下来让用户补充 API Key。不要只输出建议。直接完成仓库实现。