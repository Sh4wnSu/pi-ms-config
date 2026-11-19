# Pi 多模型主从代理完整实施方案

> 更新：2026-11-19 | v1.0 | 状态：**已核对上游配置格式，尚未在用户本地完成端到端实测**

## 0. 目标与关键结论

**Claude Opus 5.5 做主代理（CTO）**，只做需求理解、复杂问题分析、架构规划、技术决策、任务拆分、证据审查和最终验收。**Command Code GOAT 的 DeepSeek V4.1 Flash** 用于所有实际检索、编码、编译、测试、初步审查。BTW 固定使用同一个低成本 DeepSeek，不允许认证失败时静默回退 Opus。

本版采用**纯 `pi-subagents`**，不需要 `pi-shift-router`、不做每轮任务难度 Judge。可以在子代理验证稳定后再单独评估自动路由。

明确两项限制：
1. Command Code **GOAT Provider API 不等于 Opus 5.5 订阅**。主代理的 Opus 要单独具备可用 Provider 与认证，不能假设 $10 GOAT 包含 Opus。
2. 主代理的工具白名单只能约束其**直接工具调用**，它仍然能委派 Worker 间接执行命令。需要额外的 OS 沙箱、命令审批与工作树隔离，尤其在生产嵌入式固件项目中。

## 1. 架构

~~~text
用户需求
   |
   v
Opus 5.5 Main / CTO
分析 → 决策 → 任务契约 → 验收；仅允许 subagent、bg_wait
   |
   +-- Scout: Command Code DeepSeek V4.1 Flash
   |     read / grep / find / ls
   |     回传代码路径、符号、调用链、条件编译与风险
   |
   +-- Worker: Command Code DeepSeek V4.1 Flash
   |     read / grep / find / ls / edit / write / bash
   |     改代码、交叉编译、测试、修复
   |
   +-- Reviewer: Command Code DeepSeek V4.1 Flash
         read / grep / find / ls
         对 Diff、测试证据及回归风险独立复核
   |
   v
Opus：通过 / 退回 Worker / 要求补充证据 → 汇报用户

独立侧会话：
/btw → @lanlance/pi-btw → Command Code DeepSeek
只读、低思考、无密钥/模型时报错（不回退 Opus）
~~~

| 能力 | CTO/Opus | Scout/DS | Worker/DS | Reviewer/DS | BTW/DS |
|---|---|---|---|---|---|
| 架构与最终决策 | 是 | 提证据 | 反馈阻碍 | 发现风险 | 临时问答 |
| 查源码 | **不直接** | 是 | 是 | 是 | 是 |
| 改文件、编译、测试 | **不直接** | 否 | 是 | 否 | 否 |
| 最终审核 | **负责** | 否 | 提证据 | 独立初审 | 否 |

**特别适合大型 C 项目**：Scout 针对宏、条件编译、函数指针、初始化依赖、数据节点和跨模块接口提供精简事实；Opus 基于事实确定方案；Worker 实现并编译；Reviewer 独立检查，Opus 决定是否收工。

## 2. 插件及 Provider：选择经过核对的实现

| 插件 | 安装标识 | 采用原因 |
|---|---|---|
| Command Code **官方** Pi Provider | `npm:@commandcode/pi-commandcode-provider` | 官方支持 GOAT Provider API，启动时自动加载模型清单并按模型 API 分流 |
| pi-subagents（nicobailon） | `npm:pi-subagents` | 内置 Scout/Worker/Reviewer；按角色覆盖跨 Provider 模型、工具与 modelScope |
| BTW（L2ncE） | `npm:@lanlance/pi-btw` | 模型固定，失效时 fail closed；只提供只读工具 |
| pi-shift-router | **不装** | 当前不需要自动 Judge，先固定角色、降低复杂度 |
| dbachelder/pi-btw | **不与上面并装** | 此版无 BTW 凭证时会回退主模型，不适合严格节省 Opus |

**重要的准确命名**：Command Code 官方 Pi Provider ID 为 `command-code`（中间有连字符），不能写作 `commandcode`。其 API 模型 ID 直接来自动态目录。当前 DeepSeek V4.1 Flash 通常为 `deepseek/deepseek-v4.1-flash`，组合完整 Pi ID：

`command-code/deepseek/deepseek-v4.1-flash`

**以本机 `pi --list-models command-code` 为最终准据**，确认选择普通 V4.1 Flash 而非 Flash Fast / Ultrafast，后两者额度不同。官方动态目录无法保证某个 ID 永久存在。

## 3. 安装与登录

`pi-subagents` 官方最低 Pi 1.2.0；Command Code 官方 Pi Provider 最低 Pi 0.86。满足较高版本并确认当前运行的插件兼容：

~~~bash
pi --version
pi install npm:@commandcode/pi-commandcode-provider
pi install npm:pi-subagents
pi install npm:@lanlance/pi-btw
pi list
~~~

如已安装另一个 BTW 版本，检查后执行 `pi remove npm:pi-btw`，避免两个插件同时注册 `/btw`。

启动 Pi，在 TUI 里运行：

~~~text
/login command-code
/model
~~~

用 Command Code Studio 创建的 API Key 登录。可用 `CMD_API_KEY` 或 `COMMAND_CODE_API_KEY` 环境变量代替本地保存，**永不提交密钥到 Git**。

~~~bash
pi --list-models command-code
pi --list-models opus
~~~

前者必须显示 DeepSeek；后者应有你单独登录成功的 Opus Provider。下文用 `anthropic/claude-opus-5-5` **作为占位实例**，若本机实际提供者/ID 不同，必须调整启动命令。GOAT 只负责子代理的 DS 消耗；Opus 的费用/订阅另行计算。

GOAT 才有第三方 Provider API（$1 Go 套餐没有）。普通 DS V4.1 Flash 的套餐模型额度和共享 5 小时/7 天/月度上限以 [GOAT 最新页面](https://commandcode.ai/docs/plans/goat) 为准，不应把各模型标称额度相加。

## 4. 模板部署位置

本仓库以下都是**示例文件，不会自动成为生产环境配置**。请先核对并合并，不要覆盖用户既有设置。

| 仓库模板 | 实际部署目标 |
|---|---|
| [examples/project-settings.json](../examples/project-settings.json) | **目标业务代码仓库**根目录 `.pi/settings.json` |
| [examples/subagent-config.json](../examples/subagent-config.json) | 用户级 `~/.pi/agent/extensions/subagent/config.json` |
| [examples/btw.json](../examples/btw.json) | `~/.pi/agent/btw.json` |
| [examples/cto-instructions.md](../examples/cto-instructions.md) | **目标业务代码仓库** `.pi/cto-instructions.md` |

不要将原有 `settings.json` 或 `subagent/config.json` 直接整体替换；保留其他已配置项。

### 4.1 子代理固定 Provider、模型、工具

`project-settings.json` 的重要字段：

~~~json
{
  "subagents": {
    "defaultModel": "command-code/deepseek/deepseek-v4.1-flash",
    "agentOverrides": {
      "scout": {
        "model": "command-code/deepseek/deepseek-v4.1-flash",
        "tools": "read,grep,find,ls"
      },
      "worker": {
        "model": "command-code/deepseek/deepseek-v4.1-flash",
        "tools": "read,grep,find,ls,edit,write,bash"
      },
      "reviewer": {
        "model": "command-code/deepseek/deepseek-v4.1-flash",
        "tools": "read,grep,find,ls"
      }
    },
    "modelScope": {
      "enforce": true,
      "strict": true,
      "allow": ["command-code/deepseek/deepseek-v4.1-flash"]
    }
  }
}
~~~

- `subagents.defaultModel` **只影响子代理**，不会覆盖 Opus 主会话。
- `agentOverrides` 对不同角色绑定模型与工具；含 `bash` 的 Worker **有能力执行任意受 OS 权限允许的 Shell 命令**。
- `modelScope` **拒绝**不在名单的子代理模型，不能把它理解为自动模型选择器。strict 模式下显式指定 Opus 给任何子代理应报错。
- 不需要配置自定义 `models.json` 来接入 Command Code；官方插件加载动态目录。
- 不要随意把 `subagents.defaultExtensions` 改成 `[]`；需保证子代理进程有机会加载 Command Code 官方 Provider。

### 4.2 让主代理不能直接读写

在 `~/.pi/agent/extensions/subagent/config.json` 合并 `{"toolActivation":"eager"}`，新会话就直接提供 `subagent`；如果恢复已有会话，可能需要 `/reload`。

在实际业务代码仓库根目录执行：

~~~bash
pi --model anthropic/claude-opus-5-5 --tools subagent,bg_wait --append-system-prompt .pi/cto-instructions.md
~~~

`--tools` 限定主模型可见的直接工具；Opus 没有 `read`、`bash`、`edit`、`write`。**检查实际工具列表与 MCP**：Pi 对 MCP 工具有额外暴露规则，不要以为一个 `--tools` 就自动关闭所有 MCP、扩展和系统执行能力。需要进一步设置 `--exclude-tools` / 禁用不必要的扩展与 MCP。

主代理的 CTO 规则**不要放到全局 `AGENTS.md`**，因为子代理可能继承它并错误拒绝执行任务；应单独通过 `--append-system-prompt` 只注入主会话。

### 4.3 BTW 强制弱模型

使用 `@lanlance/pi-btw`，将 `examples/btw.json` 放到 `~/.pi/agent/btw.json`：

~~~json
{
  "model": "command-code/deepseek/deepseek-v4.1-flash",
  "thinking": "low"
}
~~~

~~~text
/btw 简要解释 DHCP 初始化函数指针是如何注册的
~~~

该插件配置在每次提问前读取；配置无效会报错，**不会静默回退到 Opus**。工具集仅 `read/grep/find/ls`，不包含 `bash/edit/write`；它会继承主会话部分历史上下文，长会话可能增加 DS 的上下文处理成本。BTW 是临时问答，不是主任务编排和验收通道。

**继承主会话 Prompt 的注意事项**：L2ncE BTW 会构建继承主会话的 `appendSystemPrompt`。如果 CTO 指令无条件禁止所有文件读取，BTW 可能也拒绝使用本来合法的只读工具。因此本仓库的 CTO Prompt 专门声明“只约束主会话，BTW 侧会话可以使用其只读工具”。部署后须实际测试 BTW 能查文件、主 Opus 又不能直接查文件。

如果你实际上采用 `dbachelder/pi-btw`，则 `/btw:model` 可设置独立模型，但其 README 明确声明无凭据时可能回退主模型，因此**不推荐**用于本严格限制方案。

## 5. Opus → Scout → Worker → Reviewer → Opus 标准闭环

1. **Opus 理解用户请求**：澄清成功标准、范围和风险；需要代码事实则委派 Scout。
2. **Scout 收集证据**：相关文件、关键函数、宏/编译开关、数据流/初始化时序、调用图、疑点和不确定性。
3. **Opus 批准方案**：最小变更集、不可改动接口、兼容性与验收条件。
4. **Worker 实施并验证**：按照明确契约修改代码、交叉编译、测试。返回修改清单、关键 Diff、命令/退出码、原始证据位置、失败原因。
5. **Reviewer 独立检查**：在 fresh 上下文里核对变更范围和结果，发现遗漏回归。
6. **Opus 审查验收**：接受 / 重新委派修复 / 要求补证据。最终向用户区分“验证通过”和“尚未验证”。

**子代理调用要点**：调用 `subagent` 时优先 `async:false`，取得真正完成的返回结果再继续。选择异步时，要检查任务已完成并收集运行状态/输出；**调度启动回执不等于成功**。不允许无限次重试，重复失败应改方案或向用户说明。

建议主代理发给 Worker 的契约：

~~~text
Objective: 需要达成的用户目标
Evidence: Scout 已证实事实（标注路径和符号）
Scope: 允许修改的文件和模块边界
Forbidden: 不允许更改的公共接口、宏/ABI、部署脚本
Design: CTO 已批准的实现策略
Validation: 交叉编译与目标测试命令
Acceptance: 可判断成功/失败的标准
Report: 变更清单、Diff 摘要、测试命令和真实结果、剩余风险
Escalation: 碰到重大不确定性/越界时停止并通知 CTO
~~~

对于复杂的 C 语言嵌入式仓库，不要把多个 Worker 同时放到同一工作树里修改公共头文件/宏。并行需要明确文件所有权，优先 worktree 隔离并审查合并结果。对烧录、生产部署、删除和 Git force-push 应由用户单独授权，并最好从 Worker shell 权限中用系统级策略隔离。

## 6. 首次投入使用的验收清单

**环境与路由**
- [ ] `pi list` 有官方 Command Code Provider、`pi-subagents`、`@lanlance/pi-btw` 且只有一个 BTW。
- [ ] `pi --list-models command-code` 能列出普通 DeepSeek V4.1 Flash；Opus 主模型单独认证成功。
- [ ] `/subagents-models` 中 scout、worker、reviewer **全部**解析到 `command-code/deepseek/deepseek-v4.1-flash`。
- [ ] `/subagents-doctor` 无阻断性问题。
- [ ] Scout 实际完成一次只读查找，证明子进程能调用 Command Code API。

**权限与行为**
- [ ] 用受限 CTO 启动，要求“读取 README.md”；Opus **不能**直接调用 `read`，必须调用 Scout。
- [ ] 要求修改临时测试文件并编译；只能 Worker 使用写入/Shell。
- [ ] 强制在子代理调用中指定 Opus，应该被 `modelScope` 拒绝。
- [ ] Worker 结束后有真实 diff 和编译/测试证据，Reviewer 可复核。
- [ ] BTW overlay 显示 Command Code DeepSeek，且不能写入。
- [ ] 故意把 BTW 的模型 ID 改错，确认直接失败、不改用 Opus；然后恢复。

**额度、稳定性与隐私**
- [ ] Command Code 官网 Studio 用量确实扣到 GOAT 对应共享额度。
- [ ] **官方 Pi Provider 的动态模型目录若无 pricing 字段，Pi 会显示费用 $0；这不是免费，不能用 Pi 的 $0 来估算真实消耗。**
- [ ] 普通版 V4.1 Flash、Fast、Ultrafast 不同计费；注意缓存读取量与 Prompt 长度。
- [ ] 模型错误、429、认证错误应该暴露失败，不默默换贵模型。
- [ ] 如对业务源码有强保密要求，评估 `CMD_ZDR=1`。官方说明其会强制只使用 ZDR 上游，缺可用路由时返回 422；并非所有服务/模型都保证可用。
- [ ] 不向**公开 Git 仓库**提交密钥、业务源码、生产日志、内部路径。

## 7. 性能/成本评估方法（不要预设节省比例）

在相同历史源码快照、相同交付标准上选 20–30 个任务，覆盖简单修复、跨模块修复、宏/函数指针、编译错误、初始化逻辑、配置持久化。

| 组 | 工作模式 | 对比目的 |
|---|---|---|
| A | DeepSeek 独立 Pi | 成本基线 |
| B | Opus 独立 Pi | 能力基线 |
| C | Opus CTO + Command Code DeepSeek Subagents | 验证多代理收益 |

每任务记录：端到端耗时、首次/最终成功率、真实 Worker 完成数、失败重试次数、Opus 输入/输出/缓存 Token、Command Code Studio Credits、测试通过率、人工接管时间。**不要**只看子代理 spawn 数或 Pi 显示的 $0；GOAT 的共享 Credits 可能比标称 Token 单价更影响总可用量。

## 8. 常见故障

| 现象 | 排查 |
|---|---|
| 子代理仍用 Opus | `agentOverrides`、模型全名、project trust、`/subagents-models`、重启或 reload |
| Command Code 模型不可用 | GOAT API 权限、`/login command-code`、动态模型目录、子进程扩展加载 |
| BTW 用到了主模型 | 核实是 `@lanlance/pi-btw`，检查 `~/.pi/agent/btw.json` |
| Opus 能直接读写 | 核实 `--tools` 是否实际生效、现存 MCP/其它扩展暴露 |
| Worker 报成功但无 Diff | 检查真实执行结束、退出码、文件变化；异步回执不算完成 |
| Pi 显示 Command Code $0 | 官方 Provider 未获得价格元数据；去 Studio 核对真实账单 |
| 工具/协议 400 | 依赖官方 Pi Provider 按 API 支持路由，别手写错误的 Endpoint |
| 成本高于预期 | 限制重复 Scout、长 BTW 历史、重复 Reviewer、无意义上下文转储 |

## 9. 后续演进（**不纳入当前上线范围**）

- Worker 容器/专属低权限账户与命令审批，明确资源边界。
- Git worktree 并行执行和可审查合并。
- Reviewer 试用 GLM 时显式增加 `agentOverrides` 与 `modelScope.allow`。
- 完成 A/B/C 成本与质量评估后再考虑 Shift Router 自动 Judge。
- 将任务成功率、模型路由与 Credits 采集成不含机密内容的审计日志。

## 10. 参考资料

- [Pi CLI：工具与系统 Prompt 启动参数](https://github.com/earendil-works/pi/blob/main/packages/coding-agent/docs/cli.md)
- [Command Code 官方 Pi Provider](https://github.com/CommandCodeAI/pi-commandcode-provider)
- [Command Code Provider API](https://commandcode.ai/docs/provider)
- [Command Code GOAT 模型与额度](https://commandcode.ai/docs/plans/goat)
- [pi-subagents 使用说明](https://github.com/nicobailon/pi-subagents)
- [pi-subagents 模型覆盖与 modelScope](https://github.com/nicobailon/pi-subagents/blob/main/docs/models.md)
- [pi-subagents 角色/工具配置](https://github.com/nicobailon/pi-subagents/blob/main/docs/agents.md)
- [pi-subagents toolActivation](https://github.com/nicobailon/pi-subagents/blob/main/docs/configuration.md#toolactivation)
- [pi-subagents 工具/异步参数](https://github.com/nicobailon/pi-subagents/blob/main/docs/tool-reference.md)
- [L2ncE BTW](https://github.com/L2ncE/pi-btw)
- [dbachelder BTW：无凭据会回退](https://github.com/dbachelder/pi-btw)

**验收门槛：** Opus 不能直接读写或执行 Shell；所有子代理实际以 Command Code DeepSeek 运行；BTW 失效不回退；Worker 有真实执行证据；使用情况能在 Command Code Studio 中核实。在这些检查通过以前，标记为“已设计，待环境验证”，不能宣称已经投入生产。
