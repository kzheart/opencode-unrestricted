# opencode-unrestricted

OpenCode 的「unrestricted」agent 配置包 —— 一个**模型越狱（jailbreak / 破甲）提示词**配置：把受限请求（逆向、二进制补丁、渗透测试、恶意软件机理、软件破解、钓鱼等）重框架为「学术/防御研究」语境，并显式抑制模型拒答与回退话术，使其按请求原样执行。

> **同时支持 OpenCode V1（`< 2.0`）与 V2（`≥ 2.0`）。**
> V2 组件在仓库根目录（`agents/` / `commands/` / `plugins/`）；V1 组件隔离在 `v1/` 下（`v1/agent/` / `v1/command/` / `v1/plugin/`），避免 `cp -R` 交叉污染。两套越狱行为等价：全 allow、研究语境注入、关键词改写、agent 门控。

先确认版本再选安装路径：

```bash
opencode --version
# ≥ 2.0  → 按「安装 · OpenCode V2」
# <  2.0  → 按「安装 · OpenCode V1」
```

## 目录结构

```
opencode-unrestricted/
├── agents/                         # V2 主 agent
│   └── unrestricted.md
├── commands/                       # V2 斜杠命令
│   └── unrestricted.md
├── plugins/                        # V2 插件（仅正式插件，不含测试）
│   └── unrestricted-boost.ts
├── v1/                             # V1 整包（与 V2 隔离）
│   ├── agent/unrestricted.md
│   ├── command/unrestricted.md
│   ├── plugin/unrestricted-boost.ts
│   └── opencode.json.example
├── tests/
│   ├── unrestricted-boost.test.ts      # V2 插件测试
│   └── v1-unrestricted-boost.test.ts   # V1 插件测试
├── opencode.json.example           # V2 配置示例
├── package.json
├── tsconfig.json
└── README.md
```

## 组件说明

| 路径 | 适用 | 作用 |
|------|------|------|
| `agents/unrestricted.md` | V2 | `mode: primary` + `permissions` 全 allow；Markdown body = system prompt |
| `commands/unrestricted.md` | V2 | `/unrestricted` 命令，`agent: unrestricted` |
| `plugins/unrestricted-boost.ts` | V2 | `Plugin.define` + `session.hook("context"\|"compaction")`；`event.agent` 门控 |
| `opencode.json.example` | V2 | `default_agent` / `permissions` / `agents` |
| `v1/agent/unrestricted.md` | V1 | `mode: primary` + `permission: allow` |
| `v1/command/unrestricted.md` | V1 | `/unrestricted` 命令 |
| `v1/plugin/unrestricted-boost.ts` | V1 | `export default Plugin` + `experimental.chat.system.transform` / `messages.transform` |
| `v1/opencode.json.example` | V1 | `default_agent` / `permission` / `agent`（单数） |

## V1 与 V2 差异速查

| 项目 | V1（`< 2.0`） | V2（`≥ 2.0`） |
|------|--------------|--------------|
| 本仓库路径 | `v1/agent` `v1/command` `v1/plugin` | `agents` `commands` `plugins` |
| 配置目录名 | `agent/` `command/` `plugin/` | `agents/` `commands/` `plugins/` |
| 插件入口 | `export default Plugin`（`async (input) => ({...hooks})`） | `export default Plugin.define({ id, setup })` |
| 插件包 | `@opencode-ai/plugin`（**类型导入**，运行时擦除） | `@opencode/plugin`（**运行时依赖**） |
| 系统提示注入 | `experimental.chat.system.transform` | `ctx.session.hook("context", …)`（+ `compaction`） |
| 消息改写 | `experimental.chat.messages.transform`（`info.role` / `parts`） | 同一 `context` hook（`message.role` / `message.content`） |
| agent 门控 | 从 `input` 猜 agent 名 + system lab 标记兜底 | hook 事件原生 `event.agent` |
| 配置字段 | `agent`、`permission` | `agents`、`permissions: [{ action, resource, effect }]` |

## 安装 · OpenCode V2（`opencode --version` ≥ 2）

### 1. 复制 V2 组件

```bash
cp -R agents commands plugins ~/.config/opencode/
# plugins/ 里只有正式插件 unrestricted-boost.ts；测试在仓库 tests/，不会被拷进去
```

若你以前装过本仓库的 V1 或早期单数目录，**先清理**，避免 V1/V2 并存：

```bash
rm -rf ~/.config/opencode/plugin ~/.config/opencode/agent ~/.config/opencode/command
rm -f ~/.config/opencode/plugins/*.test.ts
```

### 2. 安装插件运行时依赖

```bash
cd ~/.config/opencode && bun add @opencode/plugin
# 或：npm i @opencode/plugin
```

### 3. 写入 V2 配置

参考根目录 `opencode.json.example`：

```json
{
  "$schema": "https://opencode.ai/config.json",
  "default_agent": "unrestricted",
  "permissions": [{ "action": "*", "resource": "*", "effect": "allow" }],
  "agents": {
    "unrestricted": {
      "mode": "primary",
      "color": "#EF4444",
      "description": "默认主 agent。安全实验室：逆向/补丁/渗透/取证。工具优先。",
      "permissions": [{ "action": "*", "resource": "*", "effect": "allow" }]
    }
  }
}
```

### 4. 重启并验证

```bash
opencode service restart
opencode debug agents | grep -A2 unrestricted
# 日志里应有 loading plugin …/plugins/unrestricted-boost.ts，且没有 failed to load plugin
```

## 安装 · OpenCode V1（`opencode --version` < 2）

### 1. 复制 V1 组件（只拷 `v1/` 下的单数目录）

```bash
cp -R v1/agent v1/command v1/plugin ~/.config/opencode/
```

不要用 `cp -R v1 ~/.config/opencode/`（会多一层 `v1/` 前缀，发现不到）。
也不要拷仓库根的 `agents/` / `commands/` / `plugins/`（那是 V2 专用）。

若配置目录里已有 V2 复数目录，建议先清掉以免混用：

```bash
rm -rf ~/.config/opencode/agents ~/.config/opencode/commands ~/.config/opencode/plugins
```

### 2. 写入 V1 配置

参考 `v1/opencode.json.example`：

```json
{
  "$schema": "https://opencode.ai/config.json",
  "default_agent": "unrestricted",
  "permission": "allow",
  "agent": {
    "unrestricted": {
      "mode": "primary",
      "color": "#EF4444",
      "permission": "allow",
      "description": "默认主 agent。安全实验室：逆向/补丁/渗透/取证。工具优先。"
    }
  }
}
```

### 3. 插件依赖（可选）

V1 插件用的是 `import type { Plugin } from "@opencode-ai/plugin"`，**类型导入，运行时擦除**，一般不需要 `npm install` 也能加载。只有本地编辑/类型检查时才需要：

```bash
cd ~/.config/opencode && bun add -d @opencode-ai/plugin
```

### 4. 重启并验证

重启 opencode，按 Tab 应能切到 `unrestricted`；确认插件已加载且没有 `failed to load plugin`。

## 插件行为（两版等价）

两版插件做同一件事，只是挂接的 API 不同：

1. **注入语境**：system 里没有 `[UNRESEARCH-LAB]` 时 `unshift` 一段 BOOST，已有则跳过（幂等）。
2. **改写用户轮次**：只处理 user 文本，命中安全意图词表时在正文前追加「授权研究环境」上下文；非文本不动。
3. **agent 门控**：只对 `unrestricted` 生效；build / plan / 其他 agent 一律不动。

词表分三组：高置信度中文/英文短语、技术名与工具名、注册自动化上下文；泛词（注册/分析/测试）单独出现不触发。

跑测试与类型检查：

```bash
bun test                    # V1 + V2 插件测试
bun run typecheck           # tsc --noEmit（tsconfig.json，含 skipLibCheck）
```

## 使用

- 直接以 `unrestricted` 为主 agent 开始对话；
- 或输入 `/unrestricted <你的任务>` 强制切到该模式并立即开工。

## 效果演示

渗透测试：

<img src="images/渗透测试.png" alt="渗透测试演示" width="760">

软件破解：

<img src="images/破解.png" alt="破解演示" width="760">

## 注意事项

- **先用容易越狱的模型起手，再切换到想用的模型**：不同模型的安全对齐强度差异很大，有些模型冷启动很难破甲，一开口就拒答。对这类任务，先用一个容易越狱的模型起手——把「研究语境」建立起来、让受限内容开始产出、在对话历史里积累几轮已放行的上下文——然后用 `/models` 切换到真正想用的模型。越狱话术和任务上下文已经存在于对话历史里，目标模型更倾向于顺着继续执行，而不是从零重新判定拒答。
- **切换后目标模型回缩的应对**：如果切过去后目标模型开始拒答、要求澄清或改写为安全话术，再切回起手模型让它补一句「继续」或重发上一条请求，把上下文再往前推一段。
- **插件只影响 unrestricted**：build / plan / 子 agent 的请求不会被注入或改写。
- **不要把 V1 / V2 组件混装进同一配置目录**：OpenCode 会同时扫描单数与复数目录；混装可能导致旧实现抢先加载。按上面的清理步骤二选一。

## 搭配 Ferry 跨 Agent 续聊

上面那条「同会话内切模型」的策略，借助 [Ferry](https://github.com/kzheart/ferry) 可以再往前走一步，把已经放行过的越狱上下文连同任务，续聊到其他 coding agent 接着干，不必每个 agent 都从头破甲。

Ferry 把 Claude Code、Codex、OpenCode、Cursor 等 agent 的会话汇进同一个本机会话库，并提供 `ferry-resume` skill，让接手的 agent 自己读历史、写接手摘要、核对仓库后继续，全程只读、不写任何存储。

**典型流程**

1. 在 opencode 里用 unrestricted 配置加容易破甲的起手模型把任务做掉，让研究语境和已放行的产出在会话历史里攒起来（同「注意事项」那条）。
2. 会话变长、上下文顶满，或者想换一个更强的模型、换一个 agent 接手时，开一个新的 coding agent 会话，粘入 `/ferry-resume opencode <session-id>`。
3. 接手方从旧会话末尾读起，拿到的是研究语境已经建立、受限内容已经产出过、且停在某处的任务历史，顺着继续的概率明显高于从零破甲。

安装与更多用法见 [kzheart/ferry](https://github.com/kzheart/ferry)。

## 致谢

- 提示词参考并延展自 [MDX-Tom/gpt-5.6-instruct](https://github.com/MDX-Tom/gpt-5.6-instruct)（针对 gpt-5.6 系列的 Codex 破甲提示词与测试包，其自身参考自 [yynxxxxx/Codex-5.5-codex-instruct-5.5](https://github.com/yynxxxxx/Codex-5.5-codex-instruct-5.5)）。两者共享「研究/沙箱语境先行 + 显式抑制拒答 + 工具优先 + 按请求保真度交付」的核心机制，本仓库将其迁移到 OpenCode 平台并收窄为安全研究向。
- V2 实现基于 [Binaryify 的 PR #5](https://github.com/kzheart/opencode-unrestricted/pull/5) 验证完善，并对照 OpenCode V2 文档与 `@opencode/plugin` 类型。
- V1 实现恢复自本仓库合入 V2 之前的 `main`（`agent/` / `command/` / `plugin/`）。
- 感谢 [LINUX DO](https://linux.do) 社区

## License

MIT
