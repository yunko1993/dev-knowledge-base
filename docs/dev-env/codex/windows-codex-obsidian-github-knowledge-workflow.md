# Windows 下用 Codex + Obsidian + GitHub 管理知识库

## 1. 这篇文档解决什么问题

这篇文档用于固定一套适合个人长期积累、也适合 AI agent 参与整理的知识库工作流：

- Codex 负责整理、改写、抽象、补全文档
- Obsidian 负责本地浏览、搜索、轻量编辑
- GitHub 负责同步、版本管理和跨机器恢复

目标不是单纯“记笔记”，而是把知识库整理成：

- 人可以快速阅读
- AI 可以直接读取
- AI 读取后可以继续执行、排查、补充和维护

## 2. 整体架构

### 2.1 角色分工

- `Codex`：负责批量整理、抽象经验、统一文档风格、补充命令和排查步骤
- `Obsidian`：负责本地阅读、导航、搜索和轻量编辑 Markdown
- `GitHub`：负责版本管理、同步、历史追踪和远程备份

### 2.2 为什么这样分工

- Codex 擅长改写和结构化整理，不适合承担同步职责
- Obsidian 擅长本地阅读体验，但不是必须承担同步职责
- GitHub 最适合做版本管理和跨设备同步，边界最清晰

一句话总结：

> Codex 管内容，Obsidian 管阅读体验，GitHub 管同步和版本。

## 3. 当前目录组织

### 3.1 Git 仓库位置

知识库仓库实际位于：

```text
C:\github\ai-knowledge-base
```

这里是知识的真实存储位置，也是 Git 操作的根目录。

### 3.2 Obsidian 总入口 Vault

Obsidian 使用单独的总入口目录：

```text
C:\obsidian-vault
```

这个目录本身不是主知识仓库，而是一个统一入口，用来浏览多个本地知识库或文档仓库。

当前结构类似：

```text
C:\obsidian-vault
  README.md
  知识库导航.md
  repos\
    ai-knowledge-base\   -> 目录链接，实际指向 C:\github\ai-knowledge-base
```

这样做的好处：

- Git 仓库继续放在统一的 `C:\github`
- Obsidian 有独立入口，后续接更多仓库也不乱
- 不需要把所有仓库都真的搬进 Obsidian 目录

## 4. 日常使用方式

### 4.1 在 Codex 中做重整理

适合交给 Codex 的事情：

- 把零散经验整理成标准文档
- 给旧文档补“验证步骤 / 排障动作 / 安全边界”
- 把项目内经验抽象成跨项目通用知识
- 批量修正文档链接、标题和结构
- 把稳定方法升级成 skill

### 4.2 在 Obsidian 中做阅读和轻编辑

适合在 Obsidian 中做的事情：

- 快速浏览文档
- 用左侧目录和全局搜索找内容
- 临时补几句结论或待办
- 检查文档间跳转是否顺畅

Obsidian 在这套工作流里的定位是：

> 本地知识库阅读器 + 轻编辑器。

### 4.3 用 GitHub 做同步

同步方式保持简单：

- 本地真实文件由 Git 管理
- 修改完成后使用 `git add / commit / push`
- 远端以 GitHub 仓库为准

这意味着：

- 不依赖 Obsidian Sync
- 不要求登录 Obsidian 账号
- 换机器时只需要拉取 Git 仓库，再重新打开 Obsidian vault

## 5. 链接规范

这部分很关键，因为它直接决定文档在 GitHub 和 Obsidian 里能不能同时好用。

### 5.1 仓库内文档优先使用相对路径链接

推荐写法：

```md
[Windows 下 Codex 中文乱码：PowerShell 7 + UTF-8 解决记录](windows-powershell7-utf8.md)
```

或：

```md
[通用 Clash 规则文档](../../network/windows-clash-rules-for-ai-agents.md)
```

这样有两个好处：

- GitHub 能按仓库目录正确跳转
- Obsidian 也能按本地文件结构正确跳转

### 5.2 尽量不要在仓库正文里写 `C:\...` 绝对路径链接

例如这种写法：

```text
C:\github\ai-knowledge-base\README.md
```

更适合在终端说明或人工指引里出现，不适合作为仓库 Markdown 的主链接形式。

原因：

- GitHub 不会把它当成仓库内正常跳转
- Obsidian 往往会把它当成外部文件链接，可能交给系统其他编辑器打开

### 5.3 `[[Wiki Link]]` 不是默认首选

Obsidian 的 `[[文档名]]` 很方便，但 GitHub 不原生支持。

如果文档需要同时兼容：

- GitHub 预览
- Obsidian 本地阅读

那就优先使用标准 Markdown 相对链接。

## 6. Skill 文件和普通文档的区别

### 6.1 `SKILL.md` 首先是给 Codex 读的

Skill 文件的第一职责不是“在 Obsidian 里最好看”，而是：

- 让 Codex 能稳定解析
- 让 skill 的触发条件和执行规则足够明确

所以 `SKILL.md` 顶部通常会有 frontmatter，例如：

```yaml
---
name: code-comment-style
description: ...
---
```

### 6.2 Obsidian 会把 frontmatter 识别成属性

这不是文件损坏，而是 Obsidian 的正常行为。

如果在 Obsidian 中看到：

- `name`
- `description`

被单独渲染成“属性区”，说明 frontmatter 被正常识别了。

如果某个 skill 需要更适合人阅读的说明，可以额外补一份说明文档，而不是强行把 `SKILL.md` 改成只追求展示效果。

## 7. 这套工作流里 AI 应该承担什么

AI 在这套工作流里最适合做三件事：

### 7.1 抽象

把某个项目里的经验改写成跨项目可复用的知识，而不是把公司专有信息直接原样沉淀。

### 7.2 结构化

补齐文档的关键组成部分，例如：

- 前置条件
- 执行命令
- 成功判断
- 失败排查
- 安全边界

### 7.3 维护一致性

例如：

- 批量统一标题风格
- 批量修正相对链接
- 让 README 和具体文档入口保持一致
- 让文档和 skill 的定位不要互相混淆

## 8. 已踩过的坑

### 8.1 绝对路径链接会破坏 Obsidian 内部跳转体验

仓库里的绝对路径链接虽然适合某些桌面工具点击打开，但在 Obsidian 中容易被当成外部文件链接处理。

结论：

- 仓库文档内部，优先相对路径
- 需要给 Codex/终端明确路径时，再单独写绝对路径说明

### 8.2 Microsoft Store 和 Clash 的 `DIRECT` 规则不是一回事

即便规则命中 `DIRECT`，Microsoft Store 只要先经过系统代理入口，也可能打不开。

当前更稳的经验是：

- Codex 继续用 `HTTP_PROXY/HTTPS_PROXY`
- Microsoft Store 需要时临时关闭 Clash 系统代理

### 8.3 Obsidian 不负责版本管理

Obsidian 很适合看和写，但版本管理边界不如 Git 清晰。

所以这套工作流里应明确：

- Obsidian 不替代 Git
- GitHub 才是同步和历史追踪主线

## 9. 推荐维护方式

### 9.1 文档优先

一个问题第一次稳定解决后，先写文档。

### 9.2 高频流程再升级为 skill

当某个动作开始频繁重复，而且适合固定步骤执行时，再把它升级为 skill。

### 9.3 定期回头清理链接和目录

知识库一旦变大，最容易乱的是：

- 入口过多
- 链接失效
- 同类文档重复

所以需要定期让 Codex 帮忙做一次：

- 链接检查
- 入口梳理
- 标题统一
- 文档去重

## 10. 当前结论

当前这套工作流的推荐形态是：

- `Codex`：负责整理和维护知识内容
- `Obsidian`：负责本地阅读、搜索和轻编辑
- `GitHub`：负责同步和版本管理

这套组合的优点是：

- 工具边界清晰
- 迁移成本低
- 不依赖单一平台的闭源同步机制
- 既适合人读，也适合 AI 继续接手处理
