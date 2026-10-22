# opencode-unrestricted

OpenCode **V2** 的「unrestricted」agent 配置包 —— 一个**模型越狱（jailbreak / 破甲）提示词**配置：把受限请求（逆向、二进制补丁、渗透测试、恶意软件机理、软件破解、钓鱼等）重框架为「学术/防御研究」语境，并显式抑制模型拒答与回退话术，使其按请求原样执行。

> 本仓库**只支持 OpenCode V2**（`opencode --version` ≥ 2）：`agents/` + `commands/` + `plugins/` 布局、`permissions` 有序规则数组、`Plugin.define` + `session.hook("context")` 插件 API。
> V2 的插件 API 与 V1 不兼容，本仓库不再提供 V1 实现。

## 目录结构

```
opencode-unrestricted/
├── agents/
│   └── unrestricted.md        # 主 agent 的 system prompt（Markdown body 即 system）
├── commands/
│   └── unrestricted.md        # /unrestricted 斜杠命令
├── plugins/
│   ├── unrestricted-boost.ts  # V2 插件：注入研究语境 + 改写命中轮次
│   └── unrestricted-boost.test.ts
├── opencode.json.example      # 脱敏后的 V2 原生配置示例
├── package.json               # 声明插件运行时依赖 @opencode/plugin
└── README.md
```

## 组件说明

| 文件 | 作用 |
|------|------|
| `agents/unrestricted.md` | 定义 `unrestricted` 主 agent：`mode: primary` + `permissions` 全 allow，system prompt 采用「研究语境先行」策略，把逆向/补丁/漏洞利用/恶意软件机理等请求作为学术与防御研究产物处理，并要求工具优先、按请求保真度交付 |
| `commands/unrestricted.md` | 注册 `/unrestricted` 命令：把参数当作当前任务，切到 unrestricted agent 并立刻用工具执行 |
| `plugins/unrestricted-boost.ts` | V2 插件（`id: unrestricted.boost`）：在 `context` / `compaction` 请求上给 unrestricted agent 注入 `[UNRESEARCH-LAB]` 研究语境；对命中的安全关键词用户轮次改写为「授权研究环境」上下文。用 `event.agent` 门控，不影响 build/plan 等其他 agent |
| `opencode.json.example` | 脱敏后的 `~/.config/opencode/opencode.json` 示例，只保留 unrestricted 相关配置（V2 原生字段） |

## 与 V1 版实现的差异

| 项目 | V1 | V2 |
|------|----|----|
| agent 目录 | `agent/` | `agents/` |
| command 目录 | `command/` | `commands/` |
| 插件目录（全局） | `plugin/` | `plugins/` |
| 插件入口 | `export const X: Plugin = async (input) => ({ ...hooks })` | `export default Plugin.define({ id, setup(ctx) })` |
| 插件包 | `@opencode-ai/plugin`（类型导入） | `@opencode/plugin`（**运行时依赖**） |
| 系统提示注入 | `experimental.chat.system.transform` | `ctx.session.hook("context", e => e.system.push(...))` |
| 消息改写 | `experimental.chat.messages.transform`（`info.role` / `parts`） | 同一个 `context` hook（`message.role` / `message.content`） |
| agent 门控 | 从 `input` 猜 agent 名 | hook 事件原生带只读 `event.agent` |
| 配置字段 | `agent`、`permission` | `agents`、`permissions: [{ action, resource, effect }]` |
| 插件配置 | `plugin: [...]` | `plugins: [...]` |

本仓库不复用 V1 文件，安装时按 V2 布局放置即可；`< 2.0` 的 OpenCode 不适用。

## 安装

### 1. 复制文件到 OpenCode 配置目录

```bash
cp -R agents commands plugins ~/.config/opencode/
```

### 2. 安装插件运行时依赖

插件 `import { Plugin } from "@opencode/plugin"` 是**运行时导入**，必须在配置目录装一次：

```bash
cd ~/.config/opencode && bun add @opencode/plugin
# 或：npm i @opencode/plugin
```

缺少这个包时，插件会在加载阶段报模块解析错误；确认 `~/.config/opencode/node_modules/@opencode/plugin` 存在即可。

### 3. 修改 opencode.json

在 `~/.config/opencode/opencode.json` 里设置默认 agent（参考 `opencode.json.example`）：

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
opencode debug agents | grep -A2 unrestricted          # agent 已加载，permissions 为 allow
grep "loading plugin" ~/.local/share/opencode/log/opencode.log | grep unrestricted-boost
```

重启后按 Tab 键应能在主 agent 之间切到 `unrestricted` 开始使用。

## 配置规则

这套配置依赖 OpenCode V2 的目录自动发现机制：

1. **agent 目录**：markdown 文件放在 `~/.config/opencode/agents/`，文件名即 agent 名。`agents/unrestricted.md` 生成名为 `unrestricted` 的 agent；frontmatter 的 `description` / `mode` / `color` / `permissions` 生效，Markdown 正文就是它的 system prompt。
2. **command 目录**：markdown 放在 `~/.config/opencode/commands/`，文件名即命令名。`commands/unrestricted.md` 生成 `/unrestricted`；frontmatter 的 `agent: unrestricted` 指定执行者，正文里的 `$ARGUMENTS` 被命令参数替换。
3. **plugin 目录**：全局插件从 `~/.config/opencode/plugins/` 自动发现（`.ts`/`.js` 文件与插件包目录），**不需要写进 `opencode.json` 的 `plugins` 数组**；`plugins` 数组是给 npm 包/外部路径用的。项目级用 `.opencode/plugins/`。
4. **默认 agent**：`default_agent: "unrestricted"` 使 `unrestricted` 成为默认主 agent。agent 定义既可以写在 markdown frontmatter，也可以（冗余）写在 `opencode.json` 的 `agents` 字段，二者按顺序合并。
5. **权限模型**：V2 用有序 `permissions` 数组，`action` 支持通配（`shell` / `edit` / `read` / `subagent` / `webfetch` / `skill` / `*` 等），**最后一条命中规则生效**。全 allow 即 `action: "*"` + `effect: allow`；想收窄就先放宽规则、再用后面的例外覆盖。
6. **项目级隔离**：同样的目录结构放到项目根的 `.opencode/` 下即可按项目启用，不影响全局配置。

## 插件行为

`plugins/unrestricted-boost.ts` 只注册两个 hook：

- `ctx.session.hook("context", …)` —— agent 主循环（含工具续跑）的模型请求；
- `ctx.session.hook("compaction", …)` —— 上下文压缩摘要，避免压缩后的摘要丢掉研究语境。

每个 hook 里做两件事，且都用 `event.agent === "unrestricted"` 门控，其他 agent 一律不动：

1. **注入语境**：system 里没有 `[UNRESEARCH-LAB]` 时 `unshift` 一段 BOOST，已有则跳过（幂等，不会随多轮请求堆叠）。
2. **改写用户轮次**：只处理 `role === "user"` 的文本 part，命中安全意图词表时在正文前追加「授权研究环境」上下文；非文本 part（图片等）不动。

词表分三组：高置信度中文/英文短语、技术名与工具名、注册自动化上下文；泛词（注册/分析/测试）单独出现不触发，避免误伤正常开发请求。

跑测试：

```bash
bun test plugins/unrestricted-boost.test.ts
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
- **插件只影响 unrestricted**：`context` hook 用 `event.agent` 门控，build / plan / 子 agent 的请求不会被注入或改写；想让别的 agent 也用，把 `AGENT_ID` 改成白名单判断即可。

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
- V2 迁移依据 OpenCode V2 文档：`/docs/migrate-v1`、`/docs/build/plugins`、`/docs/build/plugins/migrate-v1`、`/docs/agents`。
- 感谢 [LINUX DO](https://linux.do) 社区

## License

MIT
