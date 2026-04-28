# 在 Codex 中让模型直接连接本机数据库（Windows / MySQL / PostgreSQL）

## 1. 目标（面向 AI 自治）

这篇文档的目标不是“告诉人怎么点界面”，而是让 AI 在拿到文档后可自行完成：

1. 判断 `mysql` 是否可用
2. 自动定位 `mysql.exe`
3. 自动把 MySQL `bin` 目录写入用户 `PATH`
4. 验证连接并执行只读 SQL

这里是“本机代执行”，不是云端直连内网数据库。

## 2. 使用前提

1. 当前机器本身可访问目标数据库。
2. 你提供连接参数：`host` / `port` / `database` / `user` / `password`。
3. 默认只读，不执行写操作。

## 3. AI 标准排查流程（必须按顺序）

### 步骤 1：检查 `mysql` 是否已在 PATH

```powershell
mysql --version
```

如果成功，直接跳到“步骤 4”。

### 步骤 2：自动查找 `mysql.exe`

先查常见安装路径：

```powershell
$candidates = @(
  "C:\Program Files\MySQL\MySQL Server 8.0\bin\mysql.exe",
  "C:\Program Files\MySQL\MySQL Server 5.7\bin\mysql.exe",
  "C:\Program Files\MariaDB 10.11\bin\mysql.exe"
)
$mysqlExe = $candidates | Where-Object { Test-Path $_ } | Select-Object -First 1
$mysqlExe
```

如果还没找到，再全盘定向搜索：

```powershell
$mysqlExe = Get-ChildItem C:\ -Recurse -Filter mysql.exe -ErrorAction SilentlyContinue |
  Where-Object { $_.FullName -match "\\bin\\mysql\.exe$" } |
  Select-Object -ExpandProperty FullName -First 1
$mysqlExe
```

### 步骤 3：把 `mysql.exe` 所在目录写入用户 PATH

```powershell
$mysqlBin = Split-Path $mysqlExe -Parent
$userPath = [Environment]::GetEnvironmentVariable("Path", "User")
if (-not (($userPath -split ';') -contains $mysqlBin)) {
  $newUserPath = if ([string]::IsNullOrWhiteSpace($userPath)) { $mysqlBin } else { "$userPath;$mysqlBin" }
  [Environment]::SetEnvironmentVariable("Path", $newUserPath, "User")
}

# 让当前会话立即生效
if (-not (($env:Path -split ';') -contains $mysqlBin)) {
  $env:Path = "$env:Path;$mysqlBin"
}

mysql --version
```

### 步骤 4：验证数据库连通性

```powershell
mysql -h <host> -P <port> -u <user> -p -D <database> -e "SELECT 1;"
```

出现 `1` 代表基础链路可用。

注意：如果项目配置文件里的密码看起来像 Base64，不要默认解码。先按应用实际配置值原样连接；只有确认项目启动时有专门的解密逻辑，才按对应规则处理。

### 步骤 5：执行只读 SQL

```sql
SELECT DATABASE() AS db, NOW() AS now_time;
SHOW TABLES LIKE 'your_table_name';
SELECT * FROM your_table_name LIMIT 20;
SELECT COUNT(*) AS total FROM your_table_name;
```

## 4. 给 Codex 的可复制提示词

```text
按这篇文档执行数据库接入排查，必须严格按顺序：
1) 先检查 mysql --version；
2) 若失败，自动查找 mysql.exe；
3) 将 mysql.exe 所在 bin 目录写入用户 PATH，并让当前会话立即生效；
4) 再次验证 mysql --version；
5) 执行 SELECT 1 验证连通性；
6) 执行我给的只读 SQL。
每一步都输出：命令、结果、结论、下一步。
禁止未经确认执行 UPDATE/DELETE/TRUNCATE/DDL。
```

## 5. 常见失败与处理

### 5.1 找不到 `mysql.exe`

1. 扩展搜索范围到其他盘符（如 `D:\`）。
2. 确认是否仅安装了 MySQL Server 但未安装客户端组件。

### 5.2 PATH 写入后仍不生效

1. 当前会话需要补 `env:Path`（文档步骤 3 已包含）。
2. 新开终端后再次执行 `mysql --version`。

### 5.3 连接失败（超时/拒绝/认证失败）

1. 检查网络与白名单。
2. 检查账号权限与库名。
3. 检查是否连错环境。

### 5.4 Codex PowerShell 报 Windows socket 10106

现象：

MySQL 可能表现为：

```text
ERROR 2004 (HY000): Can't create TCP/IP socket (10106)
```

PostgreSQL 可能表现为：

```text
psql: error: connection to server at "127.0.0.1", port 5432 failed:
could not create socket: The requested service provider could not be loaded or initialized. (0x0000277A/10106)
```

Java / PostgreSQL JDBC 可能表现为：

```text
org.postgresql.util.PSQLException: 尝试连接已失败
Caused by: java.net.SocketException: Unrecognized Windows Sockets error: 10106: create
```

或 PowerShell / Java / Python 等在 Codex shell 子进程内创建 socket 时报类似错误：

```text
无法加载或初始化请求的服务提供程序
Unrecognized Windows Sockets error: 10106: socket
```

判断方式：

1. `mysql --version` 或 `psql --version` 正常，说明客户端已安装。
2. `mysql -h ... -P ...` 或 `psql -h ... -p ...` 仍在创建 TCP socket 阶段失败。
3. 失败信息不是账号密码类错误，例如 MySQL 的 `Access denied` 或 PostgreSQL 的 `password authentication failed`。
4. 用 Codex 的 Node REPL MCP 测试 TCP：

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

如果 Node REPL 返回 `NODE_TCP_CONNECTED`，说明数据库网络是通的，问题在当前 Codex shell / PowerShell 子进程链路的 Windows socket provider，不是 MySQL / PostgreSQL 服务端，也不是端口没开。

处理建议：

1. 对只读排查，优先换一条可用通道验证，例如 Node REPL MCP 或本机正常终端。
2. 不要在这种情况下误判为库挂了、端口不通、账号错误。
3. 如果 Node REPL 能到服务端但认证失败，MySQL 错误会变成类似：

```text
Access denied for user '<user>'@'<client_ip>' (using password: YES)
```

PostgreSQL 错误会变成类似：

```text
FATAL: password authentication failed for user "<user>"
FATAL: database "<database>" does not exist
```

这时才继续查账号、密码、来源 IP 授权。

补充：如果只是本机 PostgreSQL 联调，且 Codex shell 的 `psql` 触发 10106，可以继续用 Node REPL MCP 作为临时直连通道完成建表、查数、迁移测试；不要把这个问题扩大判断成所有 PostgreSQL 客户端都不可用。

### 5.5 MySQL 配置密码不要擅自 Base64 解码

示例：某项目配置中密码为：

```yaml
password: 8mK2pQv9RzL4xTn7
```

排查结论：该值虽然像 Base64，但数据库认证要求使用配置原文作为密码；解码一层或两层都会导致：

```text
Access denied for user '<user>'@'<client_ip>' (using password: YES)
```

所以标准流程是：

1. 先用配置原文连接。
2. 原文失败，再确认项目是否接入了配置解密组件。
3. 不要仅凭“像 Base64”就自动解码。

## 6. 安全基线

1. 默认只读。
2. 写操作必须二次确认。
3. 先测测试库，再碰生产库。
