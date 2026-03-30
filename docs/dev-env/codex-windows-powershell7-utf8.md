# Windows 下 Codex 中文乱码：PowerShell 7 + UTF-8 解决记录

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

## 3. 解决步骤

### 步骤 1：升级到 PowerShell 7

优先使用 PowerShell 7，不建议继续依赖旧版 Windows PowerShell。

原因：

- PowerShell 7 对现代终端和 UTF-8 的支持更稳定
- 与跨平台工具链的兼容性更好
- 对中文输出场景更友好

### 步骤 2：让终端使用 `pwsh`

确保 Codex 使用的终端是 `pwsh`，而不是旧版 `powershell`。

目标状态：

```powershell
pwsh
```

### 步骤 3：设置终端为 UTF-8 / 65001

确保当前终端会话使用 UTF-8 编码页。

可执行：

```powershell
chcp 65001
[Console]::InputEncoding = [System.Text.UTF8Encoding]::new()
[Console]::OutputEncoding = [System.Text.UTF8Encoding]::new()
$OutputEncoding = [System.Text.UTF8Encoding]::new()
```

如果后续仍会频繁使用，可以考虑把相关设置放入 PowerShell 7 的配置文件中统一生效。

### 步骤 4：重启 Codex

完成以上设置后，重启 Codex，让新的终端配置重新加载。

### 步骤 5：重新验证中文显示

重启后再次查看中文输出是否正常。

## 4. 验证命令

可用以下命令进行验证：

```powershell
$PSVersionTable.PSVersion
$env:TERM
chcp
Write-Output "中文输出验证"
```

验证重点：

- PowerShell 主版本应为 7
- 编码页应显示为 `65001`
- 中文输出应可正常阅读

## 5. 注意事项

- 只切换编码页、不升级 PowerShell，效果可能不稳定
- 只升级 PowerShell、不确保 Codex 终端实际使用 `pwsh`，也可能仍然乱码
- 修改完成后如果不重启 Codex，旧会话可能继续沿用旧配置
- 某些历史终端窗口或外部工具自身也可能带来额外编码问题，需要分开判断

## 6. 结论

这类问题在 Windows 下通常不是单点故障，而是以下几项需要同时对齐：

- PowerShell 7
- `pwsh` 终端
- UTF-8 / 65001 编码
- 重启 Codex 后再次验证

如果以上几项都已到位，Codex 中的中文乱码问题通常可以解决。
