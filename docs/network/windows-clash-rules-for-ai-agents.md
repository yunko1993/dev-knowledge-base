# Windows 下 Clash 规则设计：给 AI 工具稳定翻墙的可复用模板

## 1. 为什么单独写这篇

这篇是“通用规则文档”，目标不是只服务 Codex，而是让所有 AI 工具共用一套稳定策略，例如：

- Codex / Claude Code / Cursor 等桌面或 CLI 工具
- 依赖 `curl`、`git`、`npm`、`pip`、`uv` 的开发链路
- 需要通过 Clash 做分流与验证的 Windows 环境

如果你已经看过 Codex 专篇，可把本文当成“规则基线 + 验证剧本”。

## 2. 适用范围

- 操作系统：Windows
- 代理工具：Clash Verge（其他 Clash 内核也可参考）
- 代理模式：`rule`
- 目标：让 AI 相关流量稳定走代理，国内常见站点尽量直连

## 3. 设计原则（先定策略，再写规则）

1. AI 相关域名必须显式规则，不依赖模糊关键字
2. 规则按“从具体到宽泛”排列，避免被 `MATCH` 提前吞掉
3. 排障阶段不要在 AI 策略组放 `DIRECT`，先保证可用性
4. 验证时同时看命令结果和 Clash 日志，不只看“能打开网页”

## 4. 最小可用模板（可直接改名复用）

> 下面模板只保留与 AI 稳定性最相关的最小集合，适合先跑通，再逐步扩展。

```yaml
mixed-port: 7897
allow-lan: true
mode: rule
log-level: info
ipv6: false

proxy-groups:
  - name: 🚀 节点选择
    type: select
    proxies:
      - 手动选择你的可用节点A
      - 手动选择你的可用节点B

  - name: 🤖 AI
    type: select
    proxies:
      - 🚀 节点选择

rules:
  # OpenAI / ChatGPT
  - DOMAIN-SUFFIX,openai.com,🤖 AI
  - DOMAIN-SUFFIX,oaistatic.com,🤖 AI
  - DOMAIN-SUFFIX,oaiusercontent.com,🤖 AI
  - DOMAIN,chatgpt.com,🤖 AI
  - DOMAIN-SUFFIX,chatgpt.com,🤖 AI
  - DOMAIN-KEYWORD,chatgpt,🤖 AI

  # GitHub（按需）
  - DOMAIN-SUFFIX,github.com,🚀 节点选择
  - DOMAIN-SUFFIX,githubusercontent.com,🚀 节点选择

  # 兜底
  - MATCH,🚀 节点选择
```

## 5. 建议补充的 AI 相关规则

按你实际使用的工具补，不要一次塞太多：

- OpenAI：`openai.com`、`chatgpt.com`、`oaistatic.com`、`oaiusercontent.com`
- Anthropic：`anthropic.com`、`claude.ai`
- Google AI：`ai.google.dev`、`generativelanguage.googleapis.com`
- 开发依赖链路：`github.com`、`githubusercontent.com`

写法建议优先 `DOMAIN` / `DOMAIN-SUFFIX`，最后才用 `DOMAIN-KEYWORD`。

## 6. Windows 侧代理注入（可选但推荐）

部分 CLI 或桌面程序不会完全跟随系统代理，建议补用户环境变量：

```powershell
setx HTTP_PROXY "http://127.0.0.1:7897"
setx HTTPS_PROXY "http://127.0.0.1:7897"
setx ALL_PROXY "http://127.0.0.1:7897"
```

然后彻底重启目标程序（必要时重启 Clash）。

## 7. 验证清单（必须做）

### 7.1 端口监听

```powershell
netstat -ano | findstr 7897
```

期望出现 `LISTENING`。

### 7.2 环境变量是否写入

```powershell
[System.Environment]::GetEnvironmentVariable("HTTP_PROXY","User")
[System.Environment]::GetEnvironmentVariable("HTTPS_PROXY","User")
[System.Environment]::GetEnvironmentVariable("ALL_PROXY","User")
```

期望值一致，且都为 `http://127.0.0.1:7897`。

### 7.3 基础链路探活（走本地代理）

```powershell
curl.exe -I -x http://127.0.0.1:7897 https://api.openai.com
```

能返回 HTTP 头，说明“本地端口 + 节点链路”基础可用。

### 7.4 日志命中验证（最关键）

1. 打开 Clash 日志
2. 触发一次真实 AI 请求
3. 确认出现相关域名并命中 `🤖 AI` 策略组

如果没命中，优先修规则顺序或域名覆盖，而不是先换工具。

## 8. 高频故障与处理

### 故障 A：网页能开，CLI 超时

常见原因：

- CLI 未读系统代理
- 环境变量未生效（程序未重启）
- 端口写错（7890/7897 混用）

处理：

1. 补 `HTTP_PROXY/HTTPS_PROXY/ALL_PROXY`
2. 重启 CLI 或 IDE
3. 再跑 `curl -x` 测试

### 故障 B：日志有请求，但工具仍频繁重连

常见原因：

- 节点不稳定
- 节点对长连接/流式响应支持差

处理：

1. 在 `🤖 AI` 组固定到单一节点测试
2. 不要先用自动策略
3. 稳定后再回到自动选择

### 故障 C：规则看起来对，但命中 `DIRECT`

常见原因：

- 规则顺序被覆盖
- 写了泛规则，没写显式域名

处理：

1. 把 AI 显式规则前置
2. 排障阶段移除 `🤖 AI` 组中的 `DIRECT`
3. 复测日志命中

## 9. 给 AI Agent 的执行提示词（可复制）

```text
请按这篇文档执行代理排障，顺序必须是：
1) 先校验 Clash 监听端口；
2) 再校验用户环境变量；
3) 再用 curl -x 测试 OpenAI 链路；
4) 最后根据 Clash 日志判断是否命中 🤖 AI 规则组。
每一步都输出“命令、实际结果、结论、下一步”。
```

## 10. 与 Codex 专篇关系

- 通用规则与排障基线：本文
- Codex 桌面版专项步骤与现象：`docs/dev-env/codex/windows-clash-verge-proxy.md`

建议维护方式：

1. 规则基线变更先改本文
2. 工具特定差异再回填到各自专篇

