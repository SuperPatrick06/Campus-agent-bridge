<div align="center">

<picture>
  <source media="(prefers-color-scheme: dark)" srcset=".github/assets/banner-dark.svg">
  <source media="(prefers-color-scheme: light)" srcset=".github/assets/banner-light.svg">
  <img alt="campus-agent-bridge — 架在 Codex 与网关之间的桥" src=".github/assets/banner-dark.svg" width="100%">
</picture>

### 把只有 chat 能力的 AI 网关，变成真正的 **agent**

**网关零改动 · 无云端中转 · 零 npm 依赖**

[![License: MIT](https://img.shields.io/badge/license-MIT-3b82f6?style=flat-square)](LICENSE) [![Node.js ≥ 18](https://img.shields.io/badge/node-%E2%89%A5%2018-22c55e?style=flat-square)](https://nodejs.org) [![Dependencies](https://img.shields.io/badge/dependencies-0-a855f7?style=flat-square)](package.json) [![Tests](https://img.shields.io/badge/tests-21%20passing-22c55e?style=flat-square)](test/run.js)

[English](README.en.md) · **简体中文**

[痛点](#痛点) · [工作原理](#工作原理) · [特性](#特性) · [快速开始](#快速开始) · [三个坑](#三个经典坑) · [隐私](#隐私)

</div>

---

## 痛点

学校 / 公司发给你一个 OpenAI 兼容的 **chat** API —— 问答很好用，当编码助手则完全不行。
[Codex](https://github.com/openai/codex) 能规划、执行命令、改你的代码，但前提是
API 得接受它完整的 **Responses** 方言。Codex 不停发送网关从没见过的东西，
网关则回以这样的报错：

| Codex 发送的 | 网关的回复 |
|---|---|
| `include`、`prompt_cache_key`、`client_metadata` … | ❌ `property include should not exist` |
| 工具输出里带图片 | ❌ `input must be a non-empty string…` |
| `custom_tool_call` 历史条目 | ❌ 不是 "supported message/function item" |
| `custom` / `namespace` 工具类型 | ❌ 只允许 `type: "function"` |

**campus-agent-bridge** 就跑在两者之间的 `127.0.0.1` 上：转发前清洗请求，
返回时把响应改写回 Codex 的方言。

## 工作原理

<picture>
  <source media="(prefers-color-scheme: dark)" srcset=".github/assets/diagram-dark.svg">
  <source media="(prefers-color-scheme: light)" srcset=".github/assets/diagram-light.svg">
  <img alt="Codex ↔ campus-agent-bridge（127.0.0.1:8787）↔ 网关" src=".github/assets/diagram-dark.svg" width="100%">
</picture>

**网关当大脑，工具在本地跑** —— 中间的代理只负责翻译协议方言。
完整协议走读：[docs/how-it-works.md](docs/how-it-works.md)。

## 特性

| | | |
|:---|:---|:---|
| **协议清洗**<br>未知参数、历史噪音，网关看不到的就不发 | **看图不坏**<br>无论网关多严格，截图照样送达模型 | **工具双向翻译**<br>`custom_tool_call` ⇄ `function_call`，JSON 与 SSE 全覆盖 |
| **沙箱解锁**<br>Codex 0.16x 必需配置，告别静默拒绝 | **零依赖**<br>一个 `node` 就是全部运行时 | **久经考验**<br>21 个测试，假网关执行真实世界的全部规则 |
| **隐私优先**<br>key 只存在于 gitignore 的 `./local/`，代理不持有 | **会话隔离**<br>你的 `~/.codex` 一行不动 | **自诊断**<br>`gab doctor` 一键体检 |

## 快速开始

**前置** — [Node.js](https://nodejs.org) ≥ 18 ·
[Codex CLI](https://github.com/openai/codex) ≥ 0.16x · 你的网关 URL + key

```sh
git clone https://github.com/SuperPatrick06/Campus-agent-bridge.git
cd campus-agent-bridge
./codex-campus        # Windows 用: codex-campus.cmd
```

**首次运行会引导你完成配置**（网关 URL → API key → 选模型），然后打开 Codex。
之后每天只需 `codex-campus` —— 自动拉起本地代理，直接进入隔离会话。

<img src=".github/assets/terminal.svg" alt="首次运行 —— 配置向导，然后进入 Codex" width="100%">

> **全局使用** —— 把该文件夹加入 `PATH`（Windows），或
> `ln -s "$PWD/codex-campus" /usr/local/bin/codex-campus`（macOS / Linux）。

<details>
<summary><b>手动命令</b> —— 同样的事，下一层拆开做</summary>

```sh
gab init        # 交互式配置 —— 写入 ./local（已 gitignore）
gab codex       # 与 ./codex-campus 完全相同
gab start       # 仅后台启动代理
gab doctor      # 检查配置 / 代理 / 网关连通性
npm test        # 跑测试（单元 + 端到端）
```

</details>

### `init` 写了什么

| 文件 | 用途 | 会被提交吗？ |
|:---|:---|:---:|
| `local/config.local.json` | 网关 URL、key、模型、端口 | ❌ gitignore |
| `local/codex-home/config.toml` | 隔离的 Codex 配置 → 指向代理 | ❌ gitignore |
| `local/codex-home/models.json` | 你网关模型的 catalog 条目 | ❌ gitignore |

你的主 `~/.codex` 配置**一行都不会动** —— `codex-campus` 用
`CODEX_HOME=local/codex-home` 启动，你放在那里的其他 provider 和 key
照常工作，互不干扰。

## 三个经典坑

每个玩 Codex × 严格网关的人都会踩，本项目已全部自动处理：

1. **`property include should not exist`** —— Codex 0.16x 会发送网关白名单外的参数
   （`include`、`prompt_cache_key`、`client_metadata` …）。
   → 转发前剥离；`reasoning` 只留 `effort`，`text` 只留 `format`。

2. **`approval required by policy, but sandbox_approval is false`** —— 新目录里
   Codex 会*静默*自动拒绝每一条命令和文件读取（连提示都没有），agent 像死了一样。
   → 生成的配置写入 `sandbox_mode = "danger-full-access"`
   （+ `[windows] sandbox = "elevated"`）。

3. **`input must be a non-empty string or an array of supported
   message/function items`** —— 看图后 Codex 发送的工具输出是数组（含
   `input_image`），网关只收字符串；而且 `input_image` 分片上多一个字段
   （如 `detail:"original"`）同样触发这个报错。→ 输出被字符串化，图片挪进
   user 消息并重建为只含 `type` + `image_url` 的分片，模型照样看得见。

每个报错在 [docs/troubleshooting.md](docs/troubleshooting.md) 都有完整条目
（报错 → 原因 → 修法）。

## 隐私

设计目标：**你在配置里输入的一切只存在于 `./local/`**：

- **不持钥匙** —— 代理原样转发 Codex 的 `Authorization` 头
- **零遥测** —— 唯一的网络请求发往*你的*网关
- **零泄漏** —— 源码里不含任何机构名、URL 或 key
- **零残留** —— `DUMP=1` 调试转储（`dump-*.json`）已 gitignore，用完请删除

<details>
<summary><b>发布前自查</b></summary>

```sh
grep -rniE 'yourname|yourschool|sk-[0-9a-f]{8}|[0-9]{1,3}(\.[0-9]{1,3}){3}' \
  . --exclude-dir=local --exclude-dir=.git
```

</details>

## 开发

```sh
npm test                        # 21 个测试：单元 + 假网关端到端
node --check proxy.js bin/cli.js
```

测试套件内置一个**假严格网关**（执行真实网关的全部规则：未知参数、条目白名单、
字符串工具输出、function-only 工具），并把真实代理跑在它前面。

欢迎贡献 —— 尤其是其他网关的协议适配！

## 免责声明

本项目为独立社区项目，与 OpenAI 及任何机构无关联。如果你的网关属于学校或公司，
**请确认其条款允许自动化客户端** —— 合规责任在你。agent 会话可以执行本地命令，
请像对待任何编码助手一样审阅模型的行为。全文：[DISCLAIMER.md](DISCLAIMER.md) ·
安全：[SECURITY.md](SECURITY.md)

## 许可

[MIT](LICENSE) © campus-agent-bridge contributors。
模型 catalog 数据派生自 [openai/codex](https://github.com/openai/codex)
（Apache-2.0）—— 见 [NOTICE](NOTICE)。

## Star History

<a href="https://star-history.com/#SuperPatrick06/Campus-agent-bridge&Date">
  <picture>
    <source media="(prefers-color-scheme: dark)" srcset="https://api.star-history.com/svg?repos=SuperPatrick06/Campus-agent-bridge&type=Date">
    <source media="(prefers-color-scheme: light)" srcset="https://api.star-history.com/svg?repos=SuperPatrick06/Campus-agent-bridge&type=Date">
    <img alt="Star History Chart" src="https://api.star-history.com/svg?repos=SuperPatrick06/Campus-agent-bridge&type=Date" width="100%">
  </picture>
</a>

<div align="center">
<br>
如果这个项目解锁了你的校园 AI —— 来个 ⭐ star 吧！
<br><br>
</div>
