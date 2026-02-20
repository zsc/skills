---
name: openclaw-ollama-agent
description: Add and troubleshoot a local Ollama-backed agent in OpenClaw (e.g. qwen3:1.7b), keep existing cloud agents (e.g. kimi-coding) intact, and verify end-to-end via `openclaw gateway` + `openclaw tui`. Use when you see errors like `Unknown model: ollama/...`, when Ollama models are not discovered/allowlisted, or when the TUI can’t connect (`gateway closed (1006)`).
---

# OpenClaw + Ollama 本地模型智能体：经验教训与可复用流程

> 说明：本文件按“技能（skill）”写法组织，未来如果要做成 Codex skill，请将其放入一个技能目录并改名为 `SKILL.md`（仅需要 `name/description` 的 YAML frontmatter + 本体内容）。

## 核心心智模型（最重要）

把“能跑起来”拆成两个独立门槛：

1) **模型被允许（allowlist）**：`agents.defaults.models` 里必须包含你要用的 `provider/model`（例如 `ollama/qwen3:1.7b`）。否则会报：
   - `Unknown model: ollama/qwen3:1.7b`

2) **Provider 被启用且能发现模型（provider discovery / models.json）**：OpenClaw 会把可用 provider 写入每个 agent 的 `models.json`（在 `~/.openclaw/agents/<agentId>/agent/models.json`）。
   - **坑：Ollama 虽然本地不需要鉴权，但 OpenClaw 的隐式 provider 逻辑依赖 “是否有 key/profile” 来决定启用 provider**。没有 `OLLAMA_API_KEY` 或 `auth-profiles.json` 里的 `ollama` profile 时，Ollama provider 可能根本不会进入模型注册表，进而继续触发 `Unknown model`。

结论：**要稳定启用本地 Ollama，推荐给目标 agent 写一个“占位”ollama auth profile（key 可随便，比如 `ollama`）。**

## 前置检查（先把环境确定下来）

1) 确认 Ollama 在本机运行且能访问默认端口：

```bash
ollama list
curl -sS http://127.0.0.1:11434/api/tags | head -c 200 && echo
```

2) 确认目标模型已拉取（示例：`qwen3:1.7b`）：

```bash
ollama pull qwen3:1.7b
```

## 标准流程：新增一个 Ollama 智能体（保留原有 kimi-coding 智能体）

以下以新增 `ollama-qwen3` 为例；原 `main`（kimi-coding）不动。

### 1) 新建 agent（仅创建隔离目录 + 写入模型引用）

```bash
openclaw agents add ollama-qwen3 \
  --model ollama/qwen3:1.7b \
  --workspace ~/.openclaw/workspace

openclaw agents list --json
```

预期：`agents list` 里出现 `ollama-qwen3`，并且 `main` 仍然是默认 agent。

### 2) 把模型加入 allowlist（建议用 alias 命令，避免 JSON path 引号坑）

推荐（更不容易写错 key）：

```bash
openclaw models aliases add "Ollama Qwen3 1.7B" ollama/qwen3:1.7b
```

等价做法（容易踩坑，但可用）：

```bash
openclaw config set 'agents.defaults.models[ollama/qwen3:1.7b]' '{"alias":"Ollama Qwen3 1.7B"}' --json
```

**经验教训：不要把 key 写成 `["ollama/qwen3:1.7b"]` 或 `['ollama/qwen3:1.7b']`**  
这会在配置里创建一个“包含引号字符”的模型 key，导致 `models list`/allowlist 看起来很怪。

自检：

```bash
openclaw config get agents.defaults.models --json
```

如果已经写错，清理（示例）：

```bash
openclaw config unset 'agents.defaults.models["ollama/qwen3:1.7b"]'
openclaw config unset "agents.defaults.models['ollama/qwen3:1.7b']"
```

### 3) 启用 Ollama provider（关键步骤：写占位 auth profile）

编辑目标 agent 的 auth store：

- 路径：`~/.openclaw/agents/ollama-qwen3/agent/auth-profiles.json`
- 增加一个 profile（示例）：

```json
{
  "profiles": {
    "ollama:local": {
      "type": "api_key",
      "provider": "ollama",
      "key": "ollama"
    }
  }
}
```

说明：
- `key` 是占位字符串；OpenClaw 可能会把它作为 `Authorization: Bearer ...` 发给 Ollama，但本地默认不会用到。
- 不要把任何真实云端 API key 写进技能文档或提交到版本库。

### 4) 验证：先用 embedded（不依赖 Gateway）

```bash
openclaw agent --local --agent ollama-qwen3 -m "你好，简单自我介绍一句。" --json
```

预期：
- `meta.agentMeta.provider` 为 `ollama`
- `meta.agentMeta.model` 为 `qwen3:1.7b`

如果仍然报 `Unknown model: ollama/qwen3:1.7b`：
- 回到上面的“两道门槛”，逐个排查：allowlist 是否有、ollama provider 是否启用。
- 用状态检查快速定位（尤其看 `missingProvidersInUse`）：

```bash
openclaw models status --agent ollama-qwen3 --json
```

## TUI 调通：走 Gateway（与真实 TUI 路径一致）

### 1) 先确保 Gateway 在跑

常见误区：`openclaw health`/`openclaw tui` 报 `gateway closed (1006)`，原因通常就是本机 gateway 没启动。

前台启动（最直接）：

```bash
openclaw gateway run --ws-log compact
```

另开一个终端做验证：

```bash
openclaw agent --agent ollama-qwen3 -m "用一句话介绍你自己。" --json
```

### 2) 进入 TUI 并切换 agent

```bash
openclaw tui
```

TUI 内常用命令：
- `/agents`：列出可选 agent
- `/agent ollama-qwen3`：切换到该 agent
- `/status`：看连接/会话/模型状态

## 常见报错与修复清单（经验教训）

### A) `gateway closed (1006 abnormal closure ...)`

含义：CLI/TUI 想连 `ws://127.0.0.1:<port>`，但网关没在监听（或端口/URL 错）。

做法：

```bash
openclaw gateway status --json
openclaw gateway run
```

### B) `Unknown model: ollama/qwen3:1.7b`

含义：模型引用在运行时解析不到，通常是以下之一：

1) **没进 allowlist**：`agents.defaults.models` 缺少 `ollama/qwen3:1.7b`  
   - 修复：`openclaw models aliases add ...` 或 `openclaw config set ...`
2) **Ollama provider 未启用/未发现**：没有 `OLLAMA_API_KEY` 且 auth profiles 里没有 `ollama` profile  
   - 修复：在目标 agent 的 `auth-profiles.json` 写 `ollama:local` 占位 profile
3) **Ollama 模型本身没拉取**：`ollama list` 没有 `qwen3:1.7b`  
   - 修复：`ollama pull qwen3:1.7b`

### C) 配置被写“奇怪的 key”（比如带引号的模型 key）

原因：`openclaw config set` 的 bracket path 写法把引号当成 key 字符写进去了。

做法：
- 用 `openclaw config get agents.defaults.models --json` 检查
- 用 `openclaw config unset ...` 清理错误 key
- 优先改用 `openclaw models aliases add` 来避免这类错误

## 最小成功标准（Definition of Done）

- `openclaw agents list` 同时包含 `main`（kimi-coding）与 `ollama-qwen3`
- `openclaw agent --local --agent ollama-qwen3 ...` 成功返回，provider=ollama
- 启动 `openclaw gateway run` 后，`openclaw agent --agent ollama-qwen3 ...` 成功返回
- `openclaw tui` 内 `/agent ollama-qwen3` 后可正常对话

