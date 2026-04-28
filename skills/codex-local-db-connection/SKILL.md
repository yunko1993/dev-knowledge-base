---
name: codex-local-db-connection
description: >-
  Use when Codex on Windows needs to connect to a locally reachable MySQL or PostgreSQL database.
  Apply for client detection, PATH repair, connectivity validation, read-only SQL execution, and
  socket 10106 troubleshooting that may require bypassing the shell network stack with Node REPL MCP.
---

# Codex Local DB Connection

目标：让 Codex 在 Windows 上遇到本机可达的 MySQL / PostgreSQL 数据库时，能按固定流程完成客户端探测、PATH 修复、连通性验证、只读查询，以及 `10106` socket 问题绕行。

## 适用场景

1. 需要在 Codex 中连接本机可访问的 MySQL / PostgreSQL。
2. `mysql` / `psql` 命令找不到，怀疑客户端没进 PATH。
3. `mysql` / `psql` / JDBC 在 Codex shell 中报 Windows socket `10106`。
4. 需要让 Codex 继续做只读排查，不要因为 shell 网络栈异常而中断。

## 默认安全边界

1. 默认只读。
2. 未经明确确认，不执行 `INSERT / UPDATE / DELETE / TRUNCATE / ALTER / DROP / CREATE`。
3. 先验证测试库，再碰生产库。
4. 不擅自解码看起来像 Base64 的数据库密码，先按配置原文尝试连接。

## 必须遵守的执行顺序

### 1. 先识别数据库类型与客户端

如果用户明确说是 MySQL，就优先检查：

```powershell
mysql --version
```

如果用户明确说是 PostgreSQL，就优先检查：

```powershell
psql --version
```

如果数据库类型未明确，先问清类型；或根据用户给的连接参数、端口和项目上下文合理判断。

### 2. 如果命令不可用，先修 PATH

#### MySQL / MariaDB

先查常见路径：

```powershell
$candidates = @(
  "C:\Program Files\MySQL\MySQL Server 8.0\bin\mysql.exe",
  "C:\Program Files\MySQL\MySQL Server 5.7\bin\mysql.exe",
  "C:\Program Files\MariaDB 10.11\bin\mysql.exe"
)
$dbExe = $candidates | Where-Object { Test-Path $_ } | Select-Object -First 1
$dbExe
```

找不到再定向搜索：

```powershell
$dbExe = Get-ChildItem C:\ -Recurse -Filter mysql.exe -ErrorAction SilentlyContinue |
  Where-Object { $_.FullName -match "\\bin\\mysql\.exe$" } |
  Select-Object -ExpandProperty FullName -First 1
$dbExe
```

#### PostgreSQL

先查常见路径：

```powershell
$candidates = @(
  "C:\Program Files\PostgreSQL\17\bin\psql.exe",
  "C:\Program Files\PostgreSQL\16\bin\psql.exe",
  "C:\Program Files\PostgreSQL\15\bin\psql.exe",
  "C:\Program Files\PostgreSQL\14\bin\psql.exe"
)
$dbExe = $candidates | Where-Object { Test-Path $_ } | Select-Object -First 1
$dbExe
```

找不到再定向搜索：

```powershell
$dbExe = Get-ChildItem C:\ -Recurse -Filter psql.exe -ErrorAction SilentlyContinue |
  Where-Object { $_.FullName -match "\\bin\\psql\.exe$" } |
  Select-Object -ExpandProperty FullName -First 1
$dbExe
```

#### 写入用户 PATH 并让当前会话立刻生效

```powershell
$dbBin = Split-Path $dbExe -Parent
$userPath = [Environment]::GetEnvironmentVariable("Path", "User")
if (-not (($userPath -split ';') -contains $dbBin)) {
  $newUserPath = if ([string]::IsNullOrWhiteSpace($userPath)) { $dbBin } else { "$userPath;$dbBin" }
  [Environment]::SetEnvironmentVariable("Path", $newUserPath, "User")
}

if (-not (($env:Path -split ';') -contains $dbBin)) {
  $env:Path = "$env:Path;$dbBin"
}
```

然后立刻重新验证：

```powershell
mysql --version
```

或：

```powershell
psql --version
```

### 3. 再做基础连通性验证

#### MySQL

```powershell
mysql -h <host> -P <port> -u <user> -p -D <database> -e "SELECT 1;"
```

#### PostgreSQL

```powershell
psql -h <host> -p <port> -U <user> -d <database> -c "SELECT 1;"
```

### 4. 如果出现 socket 10106，不要误判数据库挂了

以下错误都应优先怀疑 Codex shell / PowerShell 子进程链路的 Windows socket provider，而不是直接怀疑数据库服务端：

#### MySQL

```text
ERROR 2004 (HY000): Can't create TCP/IP socket (10106)
```

#### PostgreSQL

```text
psql: error: connection to server at "127.0.0.1", port 5432 failed:
could not create socket: The requested service provider could not be loaded or initialized. (0x0000277A/10106)
```

#### Java / Python / shell 子进程

```text
无法加载或初始化请求的服务提供程序
Unrecognized Windows Sockets error: 10106: socket
```

### 5. 遇到 10106 时，改用 Node REPL MCP 验证 TCP

使用 Node REPL MCP 跑：

```javascript
const net = await import("node:net");
const result = await new Promise((resolve) => {
  const socket = net.createConnection({ host: "<host>", port: <port>, timeout: 5000 });
  socket.on("connect", () => { socket.destroy(); resolve("NODE_TCP_CONNECTED"); });
  socket.on("timeout", () => { socket.destroy(); resolve("NODE_TCP_TIMEOUT"); });
  socket.on("error", (err) => resolve(`NODE_TCP_ERROR: ${err.code} ${err.message}`));
});
nodeRepl.write(result);
```

如果返回 `NODE_TCP_CONNECTED`，结论应写成：

1. 数据库网络是通的。
2. 问题在当前 Codex shell / PowerShell 子进程网络栈。
3. 不是数据库服务挂了，也不是端口没开。

### 6. 只读排查时的处理策略

如果 shell 客户端命中 `10106`，但 Node REPL MCP 已证明 TCP 可达：

1. 继续使用可用通道做只读验证。
2. 不要把问题扩大判断成“数据库不可用”。
3. 如果只是本机 PostgreSQL 联调，可以继续用 Node REPL MCP 做临时直连通道完成查表、迁移测试、只读查询。

## 密码处理规则

1. 用户给什么密码，就先按原文尝试连接。
2. 如果配置值“看起来像 Base64”，不要自动解码。
3. 只有用户明确说明项目有配置解密逻辑，或代码里能确认存在对应解密组件时，才按那套规则处理。

## 输出格式要求

每一步都输出：

1. 命令
2. 实际结果
3. 当前结论
4. 下一步

## 典型只读 SQL

### MySQL

```sql
SELECT DATABASE() AS db, NOW() AS now_time;
SHOW TABLES LIKE 'your_table_name';
SELECT * FROM your_table_name LIMIT 20;
SELECT COUNT(*) AS total FROM your_table_name;
```

### PostgreSQL

```sql
SELECT current_database() AS db, NOW() AS now_time;
\dt
SELECT * FROM your_table_name LIMIT 20;
SELECT COUNT(*) AS total FROM your_table_name;
```
