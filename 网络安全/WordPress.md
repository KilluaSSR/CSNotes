## 默认文件结构

在Linux上安装WordPressu前，需要完整配置LAMP。其中包括Linux操作系统、Apache HTTP服务器、MySQL数据库以及PHP语言环境。安装完成后，所有 WordPress 支持文件和目录都将在位于 的 Web 根目录中可访问`/var/www/html`。

以下是默认 WordPress 安装的目录结构：


```shell
tree -L 1 /var/www/html
.
├── index.php //是WordPress的主页
├── license.txt //包含有用的信息，例如安装的 WordPress 版本
├── readme.html
├── wp-activate.php //用于设置新的 WordPress 网站时的电子邮件激活过程
├── wp-admin //包含管理员访问的登录页面和后端仪表板
├── wp-blog-header.php
├── wp-comments-post.php
├── wp-config.php //包含 WordPress 连接数据库所需的信息，例如数据库名称、数据库主机、用户名和密码、身份验证密钥和盐以及数据库表前缀
├── wp-config-sample.php
├── wp-content //存储插件和主题的主目录。子目录`uploads/`通常存储上传到平台的所有文件。这些目录和文件应仔细枚举，因为它们可能包含敏感数据，从而可能导致远程代码执行、其他漏洞或错误配置的利用。
├── wp-cron.php
├── wp-includes //包含除管理组件和网站主题之外的所有内容。这是存储核心文件的目录，例如证书、字体、JavaScript 文件和小部件。
├── wp-links-opml.php
├── wp-load.php
├── wp-login.php
├── wp-mail.php
├── wp-settings.php
├── wp-signup.php
├── wp-trackback.php
└── xmlrpc.php
```

## 用户角色

标准 WordPress 安装中有五种类型的用户。

| 角色  |                  描述                   |
| :-: | :-----------------------------------: |
| 管理员 | 此用户可以访问网站内的管理功能，包括添加和删除用户和帖子，以及编辑源代码。 |
| 编辑  |        编辑者可以发布和管理帖子，包括其他用户的帖子。        |
| 作者  |            作者可以发布和管理自己的帖子。            |
| 贡献者 |        这些用户可以撰写和管理自己的帖子，但不能发布。        |
| 订阅者 |        这些是普通用户，可以浏览帖子并编辑其个人资料。        |

通常需要获得管理员权限才能在服务器上执行代码。然而，编辑者和作者可能有权访问某些普通用户无法访问的易受攻击的插件。


## WordPress版本确定

不同的版本有不同的漏洞、可能存在不一样的错误配置（如默认密码）等，因此，了解应用程序的版本是必要的。


- 网页源代码

	在网页源代码的`meta generator`中，可能会有版本信息。
	
	```html
	...SNIP...
	<link rel='https://api.w.org/' href='http://demo.com/index.php/wp-json/' />
	<link rel="EditURI" type="application/rsd+xml" title="RSD" href="http://demo.com/xmlrpc.php?rsd" />
	<link rel="wlwmanifest" type="application/wlwmanifest+xml" href="http://demo.com/wp-includes/wlwmanifest.xml" /> 
	<meta name="generator" content="WordPress 5.3.3" />
	...SNIP...
	```
	
	```shell
	curl -s -X GET http://demo.com | grep '<meta name="generator"'
	
	<meta name="generator" content="WordPress 5.3.3" />
	```

- CSS
	 CSS也可能会透露版本信息
	```html
	...SNIP...
	<link rel='stylesheet' id='bootstrap-css'  href='http://demo.com/wp-content/themes/ben_theme/css/bootstrap.css?ver=5.3.3' type='text/css' media='all' />
	<link rel='stylesheet' id='transportex-style-css'  href='http://demo.com/wp-content/themes/ben_theme/style.css?ver=5.3.3' type='text/css' media='all' />
	<link rel='stylesheet' id='transportex_color-css'  href='http://demo.com/wp-content/themes/ben_theme/css/colors/default.css?ver=5.3.3' type='text/css' media='all' />
	<link rel='stylesheet' id='smartmenus-css'  href='http://demo.com/wp-content/themes/ben_theme/css/jquery.smartmenus.bootstrap.css?ver=5.3.3' type='text/css' media='all' />
	...SNIP...
	```

- JS
		JS也可能会透露版本信息
	```html
	...SNIP...
	<script type='text/javascript' src='http://demo.com/wp-includes/js/jquery/jquery.js?ver=1.12.4-wp'></script>
	<script type='text/javascript' src='http://demo.com/wp-includes/js/jquery/jquery-migrate.min.js?ver=1.4.1'></script>
	<script type='text/javascript' src='http://demo.com/wp-content/plugins/mail-masta/lib/subscriber.js?ver=5.3.3'></script>
	<script type='text/javascript' src='http://demo.com/wp-content/plugins/mail-masta/lib/jquery.validationEngine-en.js?ver=5.3.3'></script>
	<script type='text/javascript' src='http://demo.com/wp-content/plugins/mail-masta/lib/jquery.validationEngine.js?ver=5.3.3'></script>
	...SNIP...
	```

## 插件与主题的枚举

> 当然我们可以使用自动化程序`WPScan`，不过这个放到最后讨论。

```shell
curl -s -X GET https://www.blogtyrant.com  | sed 's/href=/\n/g' | sed 's/src=/\n/g' | grep 'wp-content/plugins/*' | cut -d"'" -f2

// grep 'wp-content/themes/*' 也可以

"https://www.blogtyrant.com/wp-content/plugins/google-analytics-premium/assets/js/frontend-gtag.min.js?ver=8.22.0" id="monsterinsights-frontend-script-js"></script>
"https://www.blogtyrant.com/wp-content/plugins/optinmonster/assets/dist/js/helper.min.js?ver=2.16.4" id="optinmonster-wp-helper-js"></script>

```


- 遗憾的是，并非所有已安装的插件和主题都能被动发现。在这种情况下，我们必须主动向服务器发送请求来枚举它们。我们可以通过发送指向服务器上**可能存在的目录或文件**的 GET 请求来实现。

- 如果该目录或文件确实存在，我们要么获得对该目录或文件的访问权限，要么会收到来自 Web 服务器的重定向响应，表明内容确实存在。但是，我们无法直接访问它。


```shell
curl -I -X GET http://demo.com/wp-content/plugins/mail-masta

HTTP/1.1 301 Moved Permanently
Date: Wed, 13 May 2020 20:08:23 GMT
Server: Apache/2.4.29 (Ubuntu)
Location: http://demo.com/wp-content/plugins/mail-masta/
Content-Length: 356
Content-Type: text/html; charset=iso-8859-1
```

如果内容不存在，我们将收到`404 Not Found error`。


```shell
curl -I -X GET http://demo.com/wp-content/plugins/someplugin

HTTP/1.1 404 Not Found
Date: Wed, 13 May 2020 20:08:18 GMT
Server: Apache/2.4.29 (Ubuntu)
Expires: Wed, 11 Jan 1984 05:00:00 GMT
Cache-Control: no-cache, must-revalidate, max-age=0
Link: <http://demo.com/index.php/wp-json/>; rel="https://api.w.org/"
Transfer-Encoding: chunked
Content-Type: text/html; charset=UTF-8
```

## 用户枚举

## WPScan

> [WPScan](https://github.com/wpscanteam/wpscan)是一款自动化的 WordPress 扫描和枚举工具。它可以检测 WordPress 网站使用的各种主题和插件是否已过时或存在漏洞。

> 提示：
> WPScan 可以从外部来源提取漏洞信息，以增强扫描功能。我们可以从[WPVulnDB](https://wpvulndb.com/)获取 API 令牌，WPScan 会使用该令牌扫描漏洞。免费计划每天最多允许 50 个请求。要使用 WPVulnDB 数据库，只需创建一个帐户并从用户页面复制 API 令牌即可。然后，可以使用`--api-token`参数将此令牌提供给 WPScan。


- `--enumerate`标志用于枚举 WordPress 应用程序的各个组件，例如插件、主题和用户。默认情况下，WPScan 会枚举易受攻击的插件、主题、用户、媒体和备份。但是，可以提供特定参数将枚举限制在特定组件上。例如，可以使用参数枚举所有插件`--enumerate ap`。

- 让我们尝试一下一次真实的扫描吧。注意：默认使用的线程数为 5，但可以使用“-t”标志更改此值。

```shell
└─$ wpscan --url=https://www.blogtyrant.com/
_______________________________________________________________
         __          _______   _____
         \ \        / /  __ \ / ____|
          \ \  /\  / /| |__) | (___   ___  __ _ _ __ ®
           \ \/  \/ / |  ___/ \___ \ / __|/ _` | '_ \
            \  /\  /  | |     ____) | (__| (_| | | | |
             \/  \/   |_|    |_____/ \___|\__,_|_| |_|

         WordPress Security Scanner by the WPScan Team
                         Version 3.8.28
                               
       @_WPScan_, @ethicalhack3r, @erwan_lr, @firefart
_______________________________________________________________

[i] Updating the Database ...
[i] Update completed.

[+] URL: https://www.blogtyrant.com/ [198.18.1.79]
[+] Started: Tue Apr 29 17:30:41 2025

Interesting Finding(s):

[+] Headers
 | Interesting Entries:
 |  - server: cloudflare
 |  - via: 1.1 google
 |  - alt-svc: h3=":443"; ma=86400
 |  - cf-cache-status: DYNAMIC
 |  - cf-ray: 937dc829eb887bb9-LAX
 | Found By: Headers (Passive Detection)
 | Confidence: 100%

[+] robots.txt found: https://www.blogtyrant.com/robots.txt
 | Interesting Entries:
 |  - /wp-admin/
 |  - /wp-admin/admin-ajax.php
 |  - /wp-content/uploads/wpforms/
 | Found By: Robots Txt (Aggressive Detection)
 | Confidence: 100%

[+] Fantastico list found: https://www.blogtyrant.com/fantastico_fileslist.txt
 | Found By: Fantastico Fileslist (Aggressive Detection)
 | Confidence: 70%
 | Reference: https://web.archive.org/web/20140518040021/http://www.acunetix.com/vulnerabilities/fantastico-fileslist/

[+] XML-RPC seems to be enabled: https://www.blogtyrant.com/xmlrpc.php
 | Found By: Link Tag (Passive Detection)
 | Confidence: 30%
 | References:
 |  - http://codex.wordpress.org/XML-RPC_Pingback_API
 |  - https://www.rapid7.com/db/modules/auxiliary/scanner/http/wordpress_ghost_scanner/
 |  - https://www.rapid7.com/db/modules/auxiliary/dos/http/wordpress_xmlrpc_dos/
 |  - https://www.rapid7.com/db/modules/auxiliary/scanner/http/wordpress_xmlrpc_login/
 |  - https://www.rapid7.com/db/modules/auxiliary/scanner/http/wordpress_pingback_access/

[+] WordPress readme found: https://www.blogtyrant.com/readme.html
 | Found By: Direct Access (Aggressive Detection)
 | Confidence: 100%

[+] This site has 'Must Use Plugins': https://www.blogtyrant.com/wp-content/mu-plugins/
 | Found By: Direct Access (Aggressive Detection)
 | Confidence: 80%
 | Reference: http://codex.wordpress.org/Must_Use_Plugins

[+] The external WP-Cron seems to be enabled: https://www.blogtyrant.com/wp-cron.php
 | Found By: Direct Access (Aggressive Detection)
 | Confidence: 60%
 | References:
 |  - https://www.iplocation.net/defend-wordpress-from-ddos
 |  - https://github.com/wpscanteam/wpscan/issues/1299

[+] WordPress version 6.6.1 identified (Insecure, released on 2024-07-23).
 | Found By: Most Common Wp Includes Query Parameter In Homepage (Passive Detection)
 |  - https://www.blogtyrant.com/wp-includes/css/dist/block-library/style.min.css?ver=6.6.1
 | Confirmed By: Rss Generator (Aggressive Detection)
 |  - https://www.blogtyrant.com/feed/, <generator>https://wordpress.org/?v=6.6.1</generator>
 |  - https://www.blogtyrant.com/comments/feed/, <generator>https://wordpress.org/?v=6.6.1</generator>

[+] WordPress theme in use: bt2019
 | Location: https://www.blogtyrant.com/wp-content/themes/bt2019/          
 | Style URL: https://www.blogtyrant.com/wp-content/themes/bt2019/style.css  
 | Style Name: BlogTyrant 2019                                              
 | Author: AwesomeMotive                                          
 | Found By: Urls In Homepage (Passive Detection)                         
 | Confirmed By: Urls In 404 Page (Passive Detection)                       
 | Version: 2.5.11 (80% confidence)                                         
 | Found By: Style (Passive Detection)                                     
 |  - https://www.blogtyrant.com/wp-content/themes/bt2019/style.css, Match: 'Version: 2.5.11'                                                            
[+] Enumerating All Plugins (via Passive Methods)                              
[+] Checking Plugin Versions (via Passive and Aggressive Methods)             
[i] Plugin(s) Identified:                                                    
[+] google-analytics-for-wordpress                                        
 | Location: https://www.blogtyrant.com/wp-content/plugins/google-analytics-for-wordpress/                                                                 
 | Last Updated: 2025-03-27T16:04:00.000Z                                 
 | [!] The version is out of date, the latest version is 9.4.1            
 |                                                                        
 | Found By: Monster Insights Comment (Passive Detection)                  
 |                                                                         
 | Version: 9.0.0 (100% confidence)                                               | Found By: Readme - Stable Tag (Aggressive Detection)                           |  - https://www.blogtyrant.com/wp-content/plugins/google-analytics-for-wordpress/readme.txt                                                            
 | Confirmed By: Readme - ChangeLog Section (Aggressive Detection)             
 |  - https://www.blogtyrant.com/wp-content/plugins/google-analytics-for-wordpress/readme.txt                                                             
[+] google-analytics-premium                                                
 | Location: https://www.blogtyrant.com/wp-content/plugins/google-analytics-premium/                                                                  
 | Found By: Urls In Homepage (Passive Detection)                           
 | Confirmed By: Urls In 404 Page (Passive Detection)                           
 |
 | Version: 8.22.0 (100% confidence)
 | Found By: Readme - Stable Tag (Aggressive Detection)
 |  - https://www.blogtyrant.com/wp-content/plugins/google-analytics-premium/readme.txt
 | Confirmed By: Readme - ChangeLog Section (Aggressive Detection)
 |  - https://www.blogtyrant.com/wp-content/plugins/google-analytics-premium/readme.txt

[+] optin-monster
 | Location: https://www.blogtyrant.com/wp-content/plugins/optin-monster/
 |
 | Found By: Comment (Passive Detection)
 |
 | The version could not be determined.

[+] optinmonster
 | Location: https://www.blogtyrant.com/wp-content/plugins/optinmonster/
 | Last Updated: 2025-04-11T14:37:00.000Z
 | [!] The version is out of date, the latest version is 2.16.19
 |
 | Found By: Urls In Homepage (Passive Detection)
 | Confirmed By: Urls In 404 Page (Passive Detection)
 |
 | Version: 2.16.4 (100% confidence)
 | Found By: Readme - Stable Tag (Aggressive Detection)
 |  - https://www.blogtyrant.com/wp-content/plugins/optinmonster/readme.txt
 | Confirmed By: Change Log (Aggressive Detection)
 |  - https://www.blogtyrant.com/wp-content/plugins/optinmonster/CHANGELOG.md, Match: '## 2.16.4'

[+] Enumerating Config Backups (via Passive and Aggressive Methods)
 Checking Config Backups - Time: 00:00:08 <=============================================================================================================================================================================================================================================> (137 / 137) 100.00% Time: 00:00:08

[i] No Config Backups Found.

[!] No WPScan API Token given, as a result vulnerability data has not been output.
[!] You can get a free API token with 25 daily requests by registering at https://wpscan.com/register

[+] Finished: Tue Apr 29 17:31:10 2025
[+] Requests Done: 204
[+] Cached Requests: 8
[+] Data Sent: 59.835 KB
[+] Data Received: 22.522 MB
[+] Memory used: 334.406 MB
[+] Elapsed time: 00:00:28


```


