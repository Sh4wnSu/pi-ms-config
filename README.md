# pi-ms-config

Pi 多模型主从代理配置与实施方案（2026-11-19）。

**目标**：让 Claude Opus 5.5 负责需求分析、架构规划、决策和最终验收；让 Command Code GOAT 的 DeepSeek V4.1 Flash 子代理承担代码检索、修改、编译、测试和初步复核；BTW 侧会话固定使用 DeepSeek，不回退到高价主模型。

## 文档

- [完整实施方案：Pi 多模型主从代理架构](docs/PI_MULTI_MODEL_PLAN.md)

## 配置模板

| 文件 | 作用 |
| --- | --- |
| [examples/project-settings.json](examples/project-settings.json) | 目标工程的 `.pi/settings.json`：分配子代理模型、工具与模型权限 |
| [examples/subagent-config.json](examples/subagent-config.json) | `~/.pi/agent/extensions/subagent/config.json`：即时启用委派工具 |
| [examples/btw.json](examples/btw.json) | `~/.pi/agent/btw.json`：将 BTW 固定为 Command Code DeepSeek |
| [examples/cto-instructions.md](examples/cto-instructions.md) | 仅追加到主会话的 CTO 行为契约 |

## 快速开始

~~~bash
pi install npm:@commandcode/pi-commandcode-provider
pi install npm:pi-subagents
pi install npm:@lanlance/pi-btw
pi --list-models command-code
~~~

在 Pi 中运行 `/login command-code`，从 Command Code Studio 获取 API Key。将示例配置**合并**到实际运行环境（不要覆盖已有设置），用 `pi --list-models opus` 确认独立的 Opus 登录及实际模型名称。

在目标代码仓库中启动受限主会话（将 Opus 模型 ID 替换为本机实际值）：

~~~bash
pi --model anthropic/claude-opus-5-5 --tools subagent,bg_wait --append-system-prompt .pi/cto-instructions.md
~~~

然后运行 `/subagents-models` 和 `/subagents-doctor` 确认 Worker 的实际模型映射，并先在测试工作树验证权限、委派和 BTW 故障不回退行为。

> **注意**：该仓库只保存方案和示例，未在你的本地机器完成实测。Command Code GOAT 不包含 Opus 主模型的费用；不要上传任何 API Key、生产源码、内部日志或凭据。实际账单需在 Command Code Studio 查，因为官方 Pi Provider 在缺失定价元数据时可能将费用显示为 $0。
