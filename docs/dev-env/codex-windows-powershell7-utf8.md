# Windows 下 Codex 中文乱码：PowerShell 7 + UTF-8 傻瓜式处理手册

## 1. 问题现象

在 Windows 环境中使用 Codex 时，终端输出中文出现乱码，表现为：

- 中文字符显示异常
- 命令输出中的汉字不可读
- 在排障、查看日志或读取文件时影响判断

## 2. 适用环境

适用于以下场景：

- 操作系统：Windows
- 使用工具：Codex
- 默认终端或终端编码设置导致中文显示异常

## 3. 一次性处理步骤

下面按顺序执行即可，尽量不要跳步骤。

### 步骤 0：先确认当前是不是旧版 PowerShell

在当前终端执行：

```powershell
$PSVersionTable.PSVersion
$PSVersionTable.PSEdition
Get-Command powershell -ErrorAction SilentlyContinue
Get-Command pwsh -ErrorAction SilentlyContinue
```

如何判断：

- 如果主版本是 `5`，基本就是旧版 Windows PowerShell
- 如果主版本是 `7`，说明已经安装了 PowerShell 7
- 如果 `Get-Command pwsh` 能找到路径，说明机器里已经有 `pwsh`

推荐目标：

```text
PowerShell 7.x
pwsh 可执行
```

### 步骤 1：升级到 PowerShell 7

优先使用 PowerShell 7，不建议继续依赖旧版 Windows PowerShell。

原因：

- PowerShell 7 对现代终端和 UTF-8 的支持更稳定
- 与跨平台工具链的兼容性更好
- 对中文输出场景更友好

#### 方式 A：使用 `winget` 安装或升级

先检查是否有 `winget`：

```powershell
where.exe winget
```

如果能找到，再执行：

```powershell
winget install --id Microsoft.PowerShell --source winget
```

如果已经装过 PowerShell 7，可执行：

```powershell
winget upgrade --id Microsoft.PowerShell --source winget
```

#### 方式 B：没有 `winget` 时

如果执行 `where.exe winget` 提示找不到命令，说明当前环境没有 `winget`。

这时直接换一种判断方式：

```powershell
where.exe pwsh
```

如果能输出类似下面的路径，说明 PowerShell 7 已经安装：

```text
C:\Program Files\PowerShell\7\pwsh.exe
```

如果 `pwsh` 也找不到，就先手动安装 PowerShell 7，安装完成后重新打开终端，再执行：

```powershell
where.exe pwsh
pwsh -v
```

### 步骤 2：让终端使用 `pwsh`

确保 Codex 使用的终端是 `pwsh`，而不是旧版 `powershell`。

先验证 `pwsh` 能不能直接运行：

```powershell
pwsh
```

进入后再执行：

```powershell
$PSVersionTable.PSVersion
```

如果显示主版本为 `7`，说明你当前已经进入正确终端。

如果你的 Codex 终端配置里可以设置 shell，目标就是让它使用：

```text
pwsh
```

### 步骤 3：设置终端为 UTF-8 / 65001

确保当前终端会话使用 UTF-8 编码页。

在 `pwsh` 中执行：

```powershell
chcp 65001
[Console]::InputEncoding = [System.Text.UTF8Encoding]::new()
[Console]::OutputEncoding = [System.Text.UTF8Encoding]::new()
$OutputEncoding = [System.Text.UTF8Encoding]::new()
```

执行完后立刻检查：

```powershell
chcp
[Console]::InputEncoding.WebName
[Console]::OutputEncoding.WebName
```

期望结果：

- `chcp` 显示活动代码页为 `65001`
- 输入编码和输出编码都显示 `utf-8`

#### 可选：写入 PowerShell 7 配置文件

如果你不想每次都手动执行，可以先查看 PowerShell 7 配置文件路径：

```powershell
$PROFILE
```

如果文件不存在，创建它：

```powershell
New-Item -ItemType File -Force -Path $PROFILE
```

然后把下面内容追加进去：

```powershell
Add-Content -Path $PROFILE -Value 'chcp 65001 > $null'
Add-Content -Path $PROFILE -Value '[Console]::InputEncoding = [System.Text.UTF8Encoding]::new()'
Add-Content -Path $PROFILE -Value '[Console]::OutputEncoding = [System.Text.UTF8Encoding]::new()'
Add-Content -Path $PROFILE -Value '$OutputEncoding = [System.Text.UTF8Encoding]::new()'
```

追加完成后可查看：

```powershell
Get-Content $PROFILE
```

### 步骤 4：重启 Codex

完成以上设置后，重启 Codex，让新的终端配置重新加载。

建议动作：

1. 完全关闭 Codex
2. 重新打开 Codex
3. 打开一个新终端
4. 再次确认当前 shell 是 `pwsh`

### 步骤 5：重新验证中文显示

重启后再次查看中文输出是否正常。

## 4. 验证命令

下面这组命令可以直接整套执行：

```powershell
$PSVersionTable.PSVersion
$PSVersionTable.PSEdition
$PSHOME
Get-Command pwsh -ErrorAction SilentlyContinue
$env:TERM
chcp
[Console]::InputEncoding.WebName
[Console]::OutputEncoding.WebName
Write-Output "中文输出验证"
Write-Output "你好，Codex"
```

验证重点：

- PowerShell 主版本应为 7
- `PSEdition` 通常应为 `Core`
- 编码页应显示为 `65001`
- 输入和输出编码应为 `utf-8`
- 中文输出应可正常阅读

## 5. 注意事项

- 只切换编码页、不升级 PowerShell，效果可能不稳定
- 只升级 PowerShell、不确保 Codex 终端实际使用 `pwsh`，也可能仍然乱码
- 修改完成后如果不重启 Codex，旧会话可能继续沿用旧配置
- 某些历史终端窗口或外部工具自身也可能带来额外编码问题，需要分开判断
- 如果 `$PROFILE` 里重复追加了多次编码设置，虽然通常不影响使用，但会显得冗余，可以后续手动整理

## 6. 最短执行版

如果你已经确认机器里装好了 `pwsh`，可以直接按下面这套最短流程处理：

```powershell
pwsh
chcp 65001
[Console]::InputEncoding = [System.Text.UTF8Encoding]::new()
[Console]::OutputEncoding = [System.Text.UTF8Encoding]::new()
$OutputEncoding = [System.Text.UTF8Encoding]::new()
$PSVersionTable.PSVersion
chcp
Write-Output "你好，Codex"
```

然后：

1. 重启 Codex
2. 打开新终端
3. 再次执行 `Write-Output "你好，Codex"` 看中文是否正常

## 7. 结论

这类问题在 Windows 下通常不是单点故障，而是以下几项需要同时对齐：

- PowerShell 7
- `pwsh` 终端
- UTF-8 / 65001 编码
- 重启 Codex 后再次验证

如果以上几项都已到位，Codex 中的中文乱码问题通常可以解决。
