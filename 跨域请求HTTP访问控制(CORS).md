## 跨域 HTTP 请求

当一个资源从与该资源本身所在的服务器不同的域或端口请求一个资源时，资源会发起一个跨域 HTTP 请求。  
比如在自己的站点通过`<script>`标签加载一个公共cdn的js文件：`<script src="https://cdn.bootcss.com/jquery/3.3.1/core.js"></script>`  
出于安全原因，浏览器限制从脚本内发起的跨源HTTP请求。 例如，`XMLHttpRequest`和`Fetch API`遵循同源策略。 这意味着使用这些API的Web应用程序只能从加载应用程序的同一个域请求HTTP资源，除非使用CORS头文件。  
跨域资源共享（ `CORS` ）机制允许 Web 应用服务器进行跨域访问控制，从而使跨域数据传输得以安全进行。浏览器支持在 API 容器中（例如 `XMLHttpRequest` 或 `Fetch` ）使用 CORS，以降低跨域 HTTP 请求所带来的风险。

## 三种访问控制场景
根据跨域资源共享标准([cross-origin sharing standard](http://www.w3.org/TR/cors/))以及浏览器安全策略，可以分三个场景来解释跨域资源共享机制的工作原理。

### 简单的跨站请求
所谓的简单请求，就是不会触发 CORS 预检请求的跨域请求。
若请求满足以下条件则是简单跨域请求：
- 使用下列方法之一：
  - `GET`
  - `HEAD`
  - `POST`
- `Content-Type` 的值仅限于下列三者之一：
  -  `text/plain`
  -  `multipart/form-data`
  -  `application/x-www-form-urlencoded`
- 在请求中，不会发送自定义的头部（如X-Modified）

满足以上条件的跨域请求，只需要响应头中有 `Access-Control-Allow-Origin`字段，如：`Access-Control-Allow-Origin: *`或者 `Access-Control-Allow-Origin: http://example.com `，则表明，该资源可以被任意外域访问或被 example.com 访问。  
这种简单请求适用于较普遍的跨域请求应用场景。比如平时跨域调用一下别人的后台接口，后端只要简单加上 `ACAO` 这么简单一个响应头就可以搞定，相对于不使用 `CORS` 的 `JSONP` 请求，无论是前后端工作量都小不少。

### 预检请求（preflight）
预检请求 是指客户端在发送真实的HTTP请求之前，先使用 `OPTIONS` 方法发起一个预检请求到服务器，以获知服务器是否允许该实际请求。"预检请求"的使用，可以避免跨域请求对服务器的用户数据产生未预期的影响。  
当请求满足下述任一条件时，即应首先发送预检请求：
- 使用了下面任一 HTTP 方法：
  - `PUT`
  - `DELETE`
  - `CONNECT`
  - `OPTIONS`
  - `TRACE`
  - `PATCH`
- `Content-Type` 的值仅限于下列三者之一：
  -  `text/plain`
  -  `multipart/form-data`
  -  `application/x-www-form-urlencoded`
- 发送自定义的头信息，如x-pingaruner

比如下面的 `XHR` 请求，因为请求包含了一个自定义的请求首部字段（`X-PINGOTHER: pingpong`）,请求的 `Content-Type` 为 `application/xml`。因此，该请求需要首先发起“预检请求”。
```js
var invocation = new XMLHttpRequest();
var url = 'http://bar.other/resources/post-here/';
var body = '<?xml version="1.0"?><person><name>Arun</name></person>';
    
function callOtherDomain(){
  if(invocation)
    {
      invocation.open('POST', url, true);
      invocation.setRequestHeader('X-PINGOTHER', 'pingpong');
      invocation.setRequestHeader('Content-Type', 'application/xml');
      invocation.onreadystatechange = handler;
      invocation.send(body); 
    }
}
```
这个请求的示意图如下：  
![preflight request](https://fedt-blog.b0.upaiyun.com/uploads/1531151334000.png)  
预检请求中同时携带了下面两个首部字段：
```bash
# 告知服务器，实际请求将使用 POST 方法
Access-Control-Request-Method: POST 
# 告知服务器，实际请求将携带两个自定义请求首部字段：X-PINGOTHER 与 Content-Type
Access-Control-Request-Headers: X-PINGOTHER, Content-Type
```
预检请求的响应响应头中包含以下四个字段：  
```bash
# 表明服务器允许 ORIGIN 为 http://foo.example 的客户端发送的请求
Access-Control-Allow-Origin: http://foo.example
# 表明服务器允许客户端使用 POST, GET 和 OPTIONS 方法发起请求
Access-Control-Allow-Methods: POST, GET, OPTIONS
# 表明服务器允许请求中携带字段 X-PINGOTHER 与 Content-Type
Access-Control-Allow-Headers: X-PINGOTHER, Content-Type
# 表明该响应的有效时间为 86400 秒，在有效时间内，浏览器无须为同一请求再次发起预检请求
Access-Control-Max-Age: 86400
```
预检请求返回 200 状态码后，客户端才会将实际请求发送至服务器端，如果预检请求中的某一项如 Method 不在服务器端支持之内，则真实请求无法被发送到服务器端。

### 附带身份凭证的请求
有些时候我们还希望在发送复杂的跨域请求时，还携带上 `cookies` 和 HTTP 认证信息，比如在本地起了一个dev-server方便前端代码的调试。  
一般而言，对于跨域 `XMLHttpRequest` 或 `Fetch` 请求，浏览器不会发送身份凭证信息。如果要发送凭证信息，需要设置 `XMLHttpRequest` 的某个特殊标志位。  
附带身份凭证的跨域请求可以是一个简单请求也可以是一个预检请求。下面是一个简单`XHR`的凭证请求：  
```js
var invocation = new XMLHttpRequest();
var url = 'http://bar.other/resources/credentialed-content/';
    
function callOtherDomain(){
  if(invocation) {
    invocation.open('GET', url, true);
    invocation.withCredentials = true;
    invocation.onreadystatechange = handler;
    invocation.send(); 
  }
}
```
`xhr.withCredentials` 标志设置为 `true`，则本次请求会带上 `cookie`。  
此时，如果服务器端的响应中未携带 `Access-Control-Allow-Credentials: true`，浏览器将不会把响应内容返回给请求的发送者。  
#### 附带身份凭证的请求与通配符
此类跨域请求值得注意的地方就是，凭证请求的响应头信息中的`Access-Control-Allow-Origin`字段**不能为**通配符 `*`，而必须指定允许跨域请求的域如：`https://example.com`，否则该次跨域请求还是会失败。

## 结语
`CORS`对跨域请求提供很大的便利，使得前端的跨域请求不再依赖`JSONP`。`CORS`相关的问题在前后端分离的项目中很常见，除此之外，使用了`ServiceWorker`后，向CDN发起的`fetch`请求也会涉及。这篇文章旨在对`CORS`的前端请求，后端响应做一个简单的归纳，从而快速反应什么样的场景，应该设置什么样的响应头。

## 参考链接
- [HTTP访问控制（CORS）](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Access_control_CORS)
- [Server-Side Access Control](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Server-Side_Access_Control)
- [人生苦短，了解一下前端必须明白的http知识点](https://juejin.im/post/5b34e6ba51882574d20bbdd4)