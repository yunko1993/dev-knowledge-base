# Windows 下 Codex 桌面版通过 Clash Verge 连接代理的配置与校验

> 先读通用规则文档（推荐）：  
> [Windows 下 Clash 规则设计：给 AI 工具稳定翻墙的可复用模板](../../network/windows-clash-rules-for-ai-agents.md)  
> 本文只补充 Codex 桌面版的专项现象与操作细节。

## 1. 问题现象

Windows 下使用 Codex 桌面版时，如果网络不能直接访问 OpenAI，常见表现是：

- 应用频繁显示 `Reconnecting...`
- 请求发出后长时间无响应
- 规则已经写了，但不确定 Codex 到底有没有真的走代理

这类问题通常不是代码问题，而是桌面应用联网链路问题。

## 2. 适用环境

适用于以下场景：

- 操作系统：Windows
- 应用：Codex 桌面版
- 代理工具：Clash Verge
- 代理模式：`rule`
- 需要让 Codex 请求先进入 Clash，再由 Clash 规则决定直连还是代理

## 3. 核心结论

这次排障最终确认有效的思路是：

1. Clash 监听端口、系统代理和环境变量三边保持一致
2. Codex 的请求先打到本机 Clash 端口
3. OpenAI / ChatGPT 相关域名命中专门的 AI 策略组
4. 如果代理已经生效但仍频繁重连，优先怀疑节点质量，不要先怀疑规则

## 4. 推荐配置方式

### 步骤 1：确认 Clash Verge 的实际代理端口

先确认 Clash Verge 当前真正监听的是哪个端口，不要只凭印象。

如果配置文件里写的是：

```yaml
mixed-port: 7897
```

那后续所有配置都统一使用：

```text
http://127.0.0.1:7897
```

重点：

- 端口必须统一
- 不要出现 Clash 实际监听 `7890`，但环境变量写成 `7897` 这种不一致

### 步骤 2：打开 Clash 的系统代理

在 Clash Verge 中保持系统代理开启。

这一步是基础项，但对某些桌面程序来说还不一定够，所以建议再补环境变量。

### 步骤 3：给 Windows 用户环境变量写入代理

在 PowerShell 中执行：

```powershell
setx HTTP_PROXY "http://127.0.0.1:7897"
setx HTTPS_PROXY "http://127.0.0.1:7897"
setx ALL_PROXY "http://127.0.0.1:7897"
```

执行后注意：

- 新环境变量不会自动注入当前已打开的程序
- 需要彻底退出 Codex 后再打开
- 必要时连 Clash 一起重开

### 步骤 4：按顺序重启

推荐顺序：

1. 关闭当前 PowerShell 窗口
2. 彻底退出 Codex 桌面版
3. 重启 Clash Verge
4. 重新打开 Codex 桌面版

如果仍然不生效，可以再进一步：

1. 注销 Windows 账号
2. 重新登录
3. 再打开 Clash Verge 和 Codex

## 5. Clash 规则配置要点

核心原则不是“让所有流量都走代理”，而是：

```text
Codex -> 127.0.0.1:7897 -> Clash -> 按规则决定直连或代理
```

也就是说：

- `HTTP_PROXY` / `HTTPS_PROXY` 只是把出口交给 Clash
- 具体走不走代理，仍然由 Clash 规则决定
- 你原来写的 `DIRECT` 规则依然有效

### 推荐的 AI 策略组写法

排障阶段建议不要在 AI 组里保留 `DIRECT` 选项，避免误切成直连。

```yaml
- name: 🤖 ChatGPT/AI
  type: select
  use:
    - jms
    - flybird
  proxies:
    - 🚀 节点选择
```

### 推荐补充的 OpenAI / ChatGPT 规则

建议显式写明，不要只依赖关键字模糊匹配：

```yaml
- DOMAIN-SUFFIX,openai.com,🤖 ChatGPT/AI
- DOMAIN-SUFFIX,oaistatic.com,🤖 ChatGPT/AI
- DOMAIN-SUFFIX,oaiusercontent.com,🤖 ChatGPT/AI
- DOMAIN,chatgpt.com,🤖 ChatGPT/AI
- DOMAIN-SUFFIX,chatgpt.com,🤖 ChatGPT/AI
- DOMAIN-KEYWORD,chatgpt,🤖 ChatGPT/AI
```

完整配置模板请看通用文档：  
[Windows 下 Clash 规则设计：给 AI 工具稳定翻墙的可复用模板](../../network/windows-clash-rules-for-ai-agents.md)

## 6. 怎么校验配置有没有生效

### 校验 1：环境变量是否写入成功

在 PowerShell 中执行：

```powershell
[System.Environment]::GetEnvironmentVariable("HTTP_PROXY","User")
[System.Environment]::GetEnvironmentVariable("HTTPS_PROXY","User")
[System.Environment]::GetEnvironmentVariable("ALL_PROXY","User")
```

期望输出：

```text
http://127.0.0.1:7897
```

如果结果为空，说明 `setx` 没有写进去。

### 校验 2：Clash 端口是否真的在监听

执行：

```powershell
netstat -ano | findstr 7897
```

正常应看到类似：

```text
127.0.0.1:7897 LISTENING
```

如果没有监听，说明配置文件、界面显示和实际端口状态不一致。

### 校验 3：本地代理到 OpenAI 的链路是否可用

执行：

```powershell
curl.exe -I -x http://127.0.0.1:7897 https://api.openai.com
```

这个命令不代表 Codex 一定没问题，但它能确认两件事：

- 本地代理端口可用
- 通过 Clash 访问 OpenAI 的基础链路是通的

如果这里都超时，优先排查 Clash、规则或节点，不要先怪 Codex。

### 校验 4：看 Clash 日志是否出现 Codex 请求

这是最关键的一步。

操作顺序：

1. 打开 Clash Verge 日志
2. 清空日志或记住当前内容
3. 打开 Codex，让它发起一次真实请求
4. 回到日志查看是否出现相关域名

重点关注：

- `openai.com`
- `api.openai.com`
- `chatgpt.com`
- `ab.chatgpt.com`
- `oaistatic.com`
- `oaiusercontent.com`

正确表现：

- 日志里能看到这些请求
- 这些请求命中了 `🤖 ChatGPT/AI`

如果日志里根本没有这些请求，通常说明：

- Codex 没读到代理配置
- 端口写错
- Clash 没真的监听这个端口

如果日志里出现了请求，但走的是 `DIRECT`，说明规则仍有问题。

## 7. 如何判断问题到底出在哪

### 情况 1：已经进了 Clash，但 Codex 仍频繁 `Reconnecting...`

这时更应该怀疑：

- 当前 `🤖 ChatGPT/AI` 选中的节点不稳定
- 节点对长连接或流式连接支持不好
- 机场线路对 OpenAI 连接质量一般

优先动作：

1. 在 Clash Verge 里切换 `🤖 ChatGPT/AI` 的节点
2. 不要先用自动，先固定到一个具体节点测试
3. 切换后彻底退出 Codex 再重新打开

### 情况 2：日志显示已经命中 AI 策略组，而且响应很快

这说明：

- 代理配置已经生效
- Clash 分流规则已经生效
- 当前节点基本可用

如果此时 Codex 已经“秒回复”，就不要再继续折腾配置，先稳定使用。

## 8. 一句话结论

Windows 下 Codex 桌面版接入 Clash Verge 时，真正关键的不是“有没有开代理”这一句，而是以下几件事同时成立：

- Clash 端口统一
- 系统代理开启
- 环境变量写入成功
- Clash 日志里能看到 Codex / OpenAI 请求
- 这些请求命中 `🤖 ChatGPT/AI`
- 当前节点本身稳定

这几项都满足后，Codex 基本就能稳定工作。
