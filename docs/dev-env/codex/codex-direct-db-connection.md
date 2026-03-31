# 在 Codex 中让模型直接连接本机数据库（Windows / MySQL）

## 1. 目标

让 Codex 在当前这台 Windows 机器上，通过本机 `mysql` 客户端直接连接数据库并执行查询。

这里是“本机代执行”，不是云端直连你的内网数据库。

## 2. 适用场景

- 你在本机可访问目标数据库（网络、白名单、权限已开通）
- 想让 Codex 直接帮你做连通性校验、只读查询和排障
- 典型项目场景：`C:\project\gd-api` 这类内网项目

## 3. 前置条件

1. 本机已安装 MySQL 客户端，且 `mysql --version` 可用。
2. 你已准备连接参数：`host` / `port` / `database` / `user` / `password`。
3. 连接参数优先从项目配置读取，不把明文密码写进知识库文档。

## 4. 标准执行顺序

### 4.1 先校验客户端

```powershell
mysql --version
```

### 4.2 再做基础连通性

```powershell
mysql -h <host> -P <port> -u <user> -p -D <database> -e "SELECT 1;"
```

返回 `1` 说明链路和凭据基本可用。

### 4.3 再执行业务 SQL（建议只读）

```sql
SELECT DATABASE() AS db, NOW() AS now_time;
SHOW TABLES LIKE 'your_table_name';
SELECT * FROM your_table_name LIMIT 20;
SELECT COUNT(*) AS total FROM your_table_name;
```

## 5. 给 Codex 的可复制提示词

```text
请先读取我给你的项目交接文档，然后按以下顺序执行数据库检查：
1) 运行 mysql --version；
2) 使用本机 mysql 客户端执行 SELECT 1；
3) 再执行我指定的 SQL；
4) 每一步输出：命令、结果、结论、下一步。
约束：
- 默认只读；
- 未经确认，禁止 UPDATE/DELETE/TRUNCATE/DDL；
- 失败时先给原始报错，再给排查命令。
```

## 6. 失败排查

### 6.1 `mysql` 命令不存在

```powershell
where mysql
```

无结果时用绝对路径测试：

```powershell
"C:\Program Files\MySQL\MySQL Server 8.0\bin\mysql.exe" --version
```

### 6.2 超时 / 拒绝访问

1. 检查 `host:port` 连通性和白名单。
2. 检查账号库权限。
3. 确认没有连错环境（test/demo/prod）。

### 6.3 密码报错

1. 先核对密码来源（配置中心 / 环境变量 / 运维口径）。
2. 不把明文密码提交到仓库。

## 7. 安全基线

1. 默认只读查询。
2. 写操作必须二次确认。
3. 先在测试库验证，再考虑生产库。
4. SQL 执行前让 Codex复述影响范围。

