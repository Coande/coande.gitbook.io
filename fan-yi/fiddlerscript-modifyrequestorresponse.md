# Fiddler 之修改请求或响应

> 原文：[https://docs.telerik.com/fiddler/knowledgebase/fiddlerscript/modifyrequestorresponse](https://docs.telerik.com/fiddler/knowledgebase/fiddlerscript/modifyrequestorresponse)

在 Fiddler 的 onBeforeRequest 或 onBeforeResponse 函数中使用 FiddlerScript [编写规则](https://docs.telerik.com/fiddler/Extend-Fiddler/AddRules)可以自定义修改 web 请求或响应。使用哪个函数更合适取决于你在编码中需要使用的对象：OnBeforeRequest 会在每个请求前被调用，OnBeforeResponse 会在每个响应前被调用。注意：

* 没办法在 OnBeforeRequest 内使用 response 类对象，因为它们还没被创建。
* 可以在 OnBeforeResponse 内使用 request 类对象；然而，所有对它们的修改对 server 来说都是不可见的，因为在这之前 server 已经收到了请求。

添加请求头：

```text
oSession.oRequest["NewHeaderName"] = "New header value";
```

删除响应头：

```text
oSession.oResponse.headers.Remove("Set-Cookie");
```

改变一个页面请求到同一服务器的另外一个页面：

```text
if (oSession.PathAndQuery=="/version1.css") {
  oSession.PathAndQuery="/version2.css";
}
```

将一个服务器的所有请求指向到另一服务器的相同端口：

```text
if (oSession.HostnameIs("www.bayden.com")) {
  oSession.hostname="test.bayden.com";
}
```

将一个服务器的所有请求指向到两一个服务器的不同端口：

```text
if (oSession.host=="www.bayden.com:8080") {
  oSession.host="test.bayden.com:9090";
}
```

将一个服务器的所有请求指向到另一个服务器，包括  HTTPS tunnels：

```text
// 重定向浏览，包括 HTTPS tunnels
if (oSession.HTTPMethodIs("CONNECT") && (oSession.PathAndQuery == "www.example.com:443")) { 
  oSession.PathAndQuery = "beta.example.com:443"; 
}

if (oSession.HostnameIs("www.example.com")) oSession.hostname = "beta.example.com";
```

模仿 Windows 的 HOSTS 文件，修改一个 Hostname 的指向到其它 IP 地址。（无需更改 Host 请求头实现修改访问目标）:

```text
// 所有 subdomain.example.com 的请求将会定向到开发服务器 128.123.133.123
if (oSession.HostnameIs("subdomain.example.com")){
  oSession.bypassGateway = true;                   // 阻止此请求通过上游代理
  oSession["x-overrideHost"] = "128.123.133.123";  // 目标服务器的 DNS 名称或者 IP 地址
}
```

修改一个页面的请求指向到另一个页面，甚至指向到另一个服务器上的页面。（通过修改 Host 请求头实现）：

```text
if (oSession.url=="www.example.com/live.js") {
  oSession.url = "dev.example.com/workinprogress.js";
}
```

阻止上传 HTTP Cookies：

```text
oSession.oRequest.headers.Remove("Cookie");
```

解压并拆包一个 HTTP 响应，按需更新响应头：

```text
// 为了更方便的修改响应数据，取消响应的压缩或打包
oSession.utilDecodeResponse();
```

搜索并替换 HTML：

```text
if (oSession.HostnameIs("www.bayden.com") && oSession.oResponse.headers.ExistsAndContains("Content-Type","text/html")){
  oSession.utilDecodeResponse();
  oSession.utilReplaceInResponse('<b>','<u>');
}
```

忽略大小写搜索响应中的 HTML：

```text
if (oSession.oResponse.headers.ExistsAndContains("Content-Type", "text/html") && oSession.utilFindInResponse("searchfor", false)>-1){
  oSession["ui-color"] = "red";
}
```

删除所有 DIV 标签（包括标签内的内容）：

```text
// 如果响应头 content-type 为 HTML，删除所有 DIV 标签
if (oSession.oResponse.headers.ExistsAndContains("Content-Type", "html")){
  // 取消压缩和打包
  oSession.utilDecodeResponse();
  var oBody = System.Text.Encoding.UTF8.GetString(oSession.responseBodyBytes);

  // 替换所有 DIV 标签为空字符串
  var oRegEx = /<div[^>]*>(.*?)<\/div>/gi;
  oBody = oBody.replace(oRegEx, "");

  // 设置响应体内容为去掉 DIV 后的内容
  oSession.utilSetResponseBody(oBody); 
}
```

将你的浏览器伪装成 Google 爬虫：

```text
oSession.oRequest["User-Agent"]="Googlebot/2.X (+http://www.googlebot.com/bot.html)";
```

请求获得希伯来语（犹太人民族的语言以色列国通用语言）的内容：

```text
oSession.oRequest["Accept-Language"]="he";
```

阻止 .CSS 请求：

```text
if (oSession.uriContains(".css")){
  oSession["ui-color"]="orange"; 
  oSession["ui-bold"]="true";
  oSession.oRequest.FailSession(404, "Blocked", "Fiddler blocked CSS file");
}
```

伪造一个 HTTP Basic 验证（需要用户输入密码后才能显示内容）：

```text
if ((oSession.HostnameIs("www.example.com")) && 
    !oSession.oRequest.headers.Exists("Authorization")) 
  {
    // 为了阻止 IE 出现 “友好的错误信息” 而隐藏掉我们的错误信息，需要使响应体的长度大于 512 字符。
    var oBody = "<html><body>[Fiddler] Authentication Required.<BR>".PadRight(512, ' ') + "</body></html>";
    oSession.utilSetResponseBody(oBody); 
    // 构建响应头
    oSession.oResponse.headers.HTTPResponseCode = 401;
    oSession.oResponse.headers.HTTPResponseStatus = "401 Auth Required";
    oSession.oResponse["WWW-Authenticate"] = "Basic realm=\"Fiddler (just hit Ok)\"";
    oResponse.headers.Add("Content-Type", "text/html");
  }
```

修改一个请求的响应为从本地  \Captures\Responses 文件夹加载的文件（可以用于  OnBeforeRequest 或  OnBeforeResponse 函数中）：

```text
if (oSession.PathAndQuery=="/version1.css") {
  oSession["x-replywithfile"] ="version2.css";
}
```

![](../.gitbook/assets/image.png)

