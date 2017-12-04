# 常见的web后台认证机制
> Author: Chuck  
> Date: 2017年10月22日

## 1. HTTP基本认证（`Http Basic Authorization`）
HTTP基本认证（`Http Basic Authorization`）是一种用来允许网页浏览器或其他客户端程序在请求时提供用户名和口令形式的身份凭证的一种登录验证方式。  
在发送之前是以用户名追加一个冒号然后串接上口令，并将得出的结果字符串再用`Base64`算法编码。例如，提供的用户名是`Aladdin`、口令是`open sesame`，则拼接后的结果就是`Aladdin:open sesame`，然后再将其用`Base64`编码，得到`QWxhZGRpbjpvcGVuIHNlc2FtZQ==`。最终将Base64编码的字符串发送出去，由接收者解码得到一个由冒号分隔的用户名和口令的字符串。  
```js
res.setHeader('WWW-Authenticate', 'Basic realm="Secure Area"');
res.writeHead(401, {'Content-Type': 'text/plain'});
```
`WWW-Authenticate`头的属性值为`<type> realm=<realm>`：`<type>` 指的是验证的方案（“基本验证方案”是最常见的验证方案，即`Basic`）。`realm` 用来描述进行保护的区域，或者指代保护的范围。它可以是类似于 "Access to the staging site" 的消息，这样用户就可以知道他们正在试图访问哪一空间。
同时返回状态码 `401 Unauthorized`，告诉客户端请求未被授权。

基本认证的一个优点是基本上所有流行的网页浏览器都支持基本认证。缺点是如果没有使用SSL/TLS这样的传输层安全的协议，那么以明文传输的密钥和口令很容易被拦截。该方案也同样没有对服务器返回的信息提供保护。基本认证很少在可公开访问的互联网网站上使用，有时候会在小的私有系统中使用（如路由器网页管理接口）。后来的机制(HTTP摘要认证)[https://zh.wikipedia.org/wiki/HTTP%E6%91%98%E8%A6%81%E8%AE%A4%E8%AF%81]是为替代基本认证而开发的，允许密钥以相对安全的方式在不安全的通道上传输。

## 基于 `cookie` 的认证机制
HTTP是一个“无状态”协议，这意味着Web应用程序服务器在响应客户端请求时不会将多个请求链接到任何一个客户端。然而，许多Web应用程序的安全和正常运行都取决于系统能够区分用户并识别用户及其权限。  

每次 HTTP 请求浏览器都会将 cookie 发送给服务器，使得客户端与服务器之间的通信是“有状态”的。  

使用 HTTP 协议规定的 set-cookie 头来设置 cookie，Express 中进行了封装：
```js
res.cookie('loginname', 'allen', {
    signed: true,
    maxAge: 600000,
    httpOnly: true,
    path: '/',
});
```
设置 cookie 的一些参数会影响将 cookie 发送给服务器端的过程：
- path：表示 cookie 影响到的路径，匹配该路径才发送这个 cookie。
- expires 和 maxAge：告诉浏览器这个 cookie 什么时候过期。
  - expires 是 UTC 格式时间。
  - maxAge 是 cookie 多久后过期的相对时间。
  - 当不设置这两个选项时，会产生 session cookie，当用户关闭浏览器时，就被清除。maxAge 优先级高于 expires。
- secure：当 secure 值为 true 时，cookie 在 HTTP 中是无效，在 - HTTPS 中才有效。
- httpOnly：浏览器不允许脚本操作 document.cookie 去更改 cookie。一般情况下都应该设置这个为 true，这样可以避免被 xss 攻击拿到 cookie。
- signed​​​​​​​：使用签名，默认为false。express 会使用req.secret 来完成签名，需要cookie-parser配合使用。

### cookie 的使用

1. 使用加密签名 cookie
2. ​​​​​​​使用 cookie - session
cookie 的弊端：
- cookie 完全暴露给客户端，依赖于安全的存储与传输
- cookie 中数据字段太多会影响传输效率
session 的运作通过一个 session_id 来进行。session_id 通常是存放在客户端的 cookie 中，session 本身则可以存放在服务器的内存或数据库中。服务器端可以安全存储用户信息相关的敏感数据。  
服务器接收到请求时，会检查 cookie 中保存的 session_id 并通过这个 session_id 与服务器端的 session data 关联起来，从而既保存了用户状态，又保证了数据存储的安全。  
Express 提供 express-session 中间件方便 cookie/session 的生成与存放。  

## 基于 Token 的身份验证​​​​​​​
使用基于 Token 的身份验证方法，在服务端不需要存储用户的登录记录。大概的流程是这样的：
1. 客户端使用用户名跟密码请求登录
2. 服务端收到请求，去验证用户名与密码
3. 验证成功后，服务端会签发一个 Token，再把这个 Token 发送给客户端
4. 客户端收到 Token 以后可以把它存储起来，比如放在 Cookie 里或者 Local Storage 里
5. 客户端每次向服务端请求资源的时候需要带着服务端签发的 Token
6. 服务端收到请求，然后去验证客户端请求里面带着的 Token，如果验证成功，就向客户端返回请求的数据

### 基于 JWT 的 Token 认证机制
`​​​​​​​JWT`: `JSON Web Tokens`，一种 token 认证实现的规范。
JWT 标准的 Token 有三个部分：
- header：头部，描述关于该JWT的最基本的信息，例如其类型以及签名所用的算法等。
- payload：载荷， Token 的具体内容。
- signature：签名，由头部与载荷加密生成。
头部与载荷原始信息都为 JSON 格式，序列化成 JSON 字符串然后进行 Base64 编码。头部与载荷的 Base64编码字符串用“.”连接后，再由一个 Secret 进行加密。
```js
var encodedString = base64UrlEncode(header) + "." + base64UrlEncode(payload); 
var signature = HMACSHA256(encodedString, 'secret');
```
最后，将这三个字符串依次用“.”连接，拼成的字符串就是完整的 JWT  

载荷（Payload）
```js
{
 "iss": "Online JWT Builder",  // Issuer，签发者
  "iat": 1416797419,  // issued at: 在什么时候签发的(UNIX时间)；
  "exp": 1448333419,  // Expiration time，过期时间
  "aud": "www.example.com", // Audience，接收方
  "sub": "jrocket@example.com",  // Subject，主题
  "GivenName": "Johnny", 
  "Surname": "Rocket", 
  "Email": "jrocket@example.com", 
  "Role": [ "Manager", "Project Administrator" ] 
}
```
头部（Header）
```js
{
  "typ": "JWT", // Token 类型
  "alg": "HS256" // 使用的加密算法
}
```
使用 JWT 的 URL 示例：
https://your.awesome-app.com/make-friend/?jwt=eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJmcm9tX3VzZXIiOiJCIiwidGFyZ2V0X3VzZXIiOiJBIn0.rSWamyAYwuHCo7IFAgd1oRpSP7nzL7BF5t7ItqpKViM

### 基于 JWT 的 Token 认证流程：
- 客户端（APP客户端或浏览器）通过GET或POST请求访问资源（页面或调用API）；
- 认证服务作为一个Middleware HOOK 对请求进行拦截，首先在cookie中查找Token信息，如果没有找到，则在HTTP Authorization Head中查找；
- 如果找到Token信息，则根据配置文件中的签名加密秘钥，调用JWT Lib对Token信息进行解密和解码；
- 完成解码并验证签名通过后，对Token中的exp、nbf、aud等信息进行验证；
- 全部通过后，根据获取的用户的角色权限信息，进行对请求的资源的权限逻辑判断；
- 如果权限逻辑判断通过则通过Response对象返回；否则则返回HTTP 401；
注意点：​​​​​​​
- 一个Token就是一些信息的集合，在 Token 中包含足够多的信息，以便在后续请求中减少查询数据库的几率；
- 服务端需要对cookie和HTTP Authrorization Header进行 Token 信息的检查；
- token是被签名的，所以我们可以认为一个可以解码认证通过的 token 是由我们系统发放的，其中带的信息是合法有效的；

## 开放授权认证（`Open Authorization`）
规范出自 [IETF RFC 6749](http://www.rfcreader.com/#rfc6749)  
OAuth是一个关于授权（authorization）的开放网络标准，在全世界得到广泛应用，目前的版本是2.0版。  
​​​​​​​OAuth1.0 因各种问题被淘汰，OAuth2.0不向后兼容  

OAuth 允许用户提供一个令牌，而不是用户名和密码来访问他们存放在特定服务提供者的数据。每一个令牌授权一个**特定的网站**（例如，视频编辑网站）在**特定的时段**（例如，接下来的2小时内）内访问**特定的资源**（例如仅仅是某一相册中的视频）。这样，OAuth让用户可以授权第三方网站访问他们存储在另外服务提供者的**某些特定信息**，而非所有内容。  
认证流程的参与者：
- `RO (resource owner)`:  资源所有者，对资源具有授权能力的人。
- `Client`: 第三方应用，它获得RO的授权后便可以去访问RO的资源。
- `​​​​​​​AS (authorization server)`:  授权服务器，它认证RO的身份，为RO提供授权审批流程，并最终颁发授权令- `牌(Access Token)。
- `RS (resource server)`: 资源服务器，它存储资源，并处理对资源的访问请求。

为了便于协议的描述，这里只是在逻辑上把AS与RS区分开来；在物理上，AS与RS的功能可以由同一个服务器来提供服务。

### OAuth 2.0的运行流程
```
     +--------+                               +---------------+
     |        |--(A)- Authorization Request ->|   Resource    |
     |        |                               |     Owner     |
     |        |<-(B)-- Authorization Grant ---|               |
     |        |                               +---------------+
     |        |
     |        |                               +---------------+
     |        |--(C)-- Authorization Grant -->| Authorization |
     | Client |                               |     Server    |
     |        |<-(D)----- Access Token -------|               |
     |        |                               +---------------+
     |        |
     |        |                               +---------------+
     |        |--(E)----- Access Token ------>|    Resource   |
     |        |                               |     Server    |
     |        |<-(F)--- Protected Resource ---|               |
     +--------+                               +---------------+
```
- （A）用户打开客户端以后，客户端要求用户给予授权。
- （B）用户同意给予客户端授权。
- （C）客户端使用上一步获得的授权，向认证服务器申请令牌。
- （D）认证服务器对客户端进行认证以后，确认无误，同意发放令牌。
- （E）客户端使用令牌，向资源服务器申请获取资源。
- （F）资源服务器确认令牌无误，同意向客户端开放资源。

OAuth2.0 授权模式
客户端必须得到用户的授权（authorization grant），才能获得令牌（access token）。OAuth 2.0定义了四种授权方式:
- 授权码模式`（authorization code）`
- 简化模式`（implicit）`
- 密码模式`（resource owner password credentials）`
- 客户端模式`（client credentials）`
其中，授权码模式（authorization code），功能最完整、流程最严密的授权模式。  
通过客户端的后台服务器，与"服务提供商"的认证服务器（AS (authorization server)）授权服务器行互动。  

### 授权码模式（`authorization code`）
```
     +----------+
     | Resource |
     |   Owner  |
     |          |
     +----------+
          ^
          |
         (B)
     +----|-----+          Client Identifier      +---------------+
     |         -+----(A)-- & Redirection URI ---->|               |
     |  User-   |                                 | Authorization |
     |  Agent  -+----(B)-- User authenticates --->|     Server    |
     |          |                                 |               |
     |         -+----(C)-- Authorization Code ---<|               |
     +-|----|---+                                 +---------------+
       |    |                                         ^      v
      (A)  (C)                                        |      |
       |    |                                         |      |
       ^    v                                         |      |
     +---------+                                      |      |
     |         |>---(D)-- Authorization Code ---------'      |
     |  Client |          & Redirection URI                  |
     |         |                                             |
     |         |<---(E)----- Access Token -------------------'
     +---------+       (w/ Optional Refresh Token)
```
- （A）用户访问客户端，后者将前者导向认证服务器。
- （B）用户选择是否给予客户端授权。
- （C）假设用户给予授权，认证服务器将用户导向客户端事先指定的"重定向URI"（redirection URI），同时附上一个- 授权码。
- （D）客户端收到授权码，附上早先的"重定向URI"，向认证服务器申请令牌。这一步是在客户端的后台的服务器上完成- 的，对用户不可见。
- （E）认证服务器核对了授权码和重定向URI，确认无误后，向客户端发送访问令牌（access token）和更新令牌（refresh token）。

#### 授权码模式各个步骤所需参数
A步骤中，客户端申请认证的URI，包含以下参数：
- response_type：表示授权类型，必选项，此处的值固定为"code"
- client_id：表示客户端的ID，必选项
- redirect_uri：表示重定向URI，可选项
- scope：表示申请的权限范围，可选项
- state：表示客户端的当前状态，可以指定任意值，认证服务器会原封不动地返回这个值。

例如：
```http
GET /authorize?response_type=code&client_id=s6BhdRkqt3&state=xyz
&redirect_uri=https%3A%2F%2Fclient%2Eexample%2Ecom%2Fcb 
HTTP/1.1
Host: server.example.com
```
C步骤中，  
如果用户同意授权，AS 服务器回应客户端的URI，包含以下参数：  
- code：表示授权码，必选项。该码的有效期应该很短，通常设为10分钟，客户端只能使用该码一次，否则会被授权服务器拒绝。该码与客户端ID和重定向URI，是一一对应关系。
- state：如果客户端的请求中包含这个参数，认证服务器的回应也必须一模一样包含这个参数。
```http
HTTP/1.1 302 Found
Location: https://client.example.com/cb?code=SplxlOBeZQQYbYS6WxSbIA&state=xyz
```
D步骤中，客户端向认证服务器申请令牌的HTTP请求，包含以下参数：
- grant_type：表示使用的授权模式，必选项，此处的值固定为"authorization_code"。
- code：表示上一步获得的授权码，必选项。
- redirect_uri：表示重定向URI，必选项，且必须与A步骤中的该参数值保持一致。
- client_id：表示客户端ID，必选项。

例如：
```http
POST /token HTTP/1.1
Host: server.example.com
Authorization: Basic czZCaGRSa3F0MzpnWDFmQmF0M2JW
Content-Type: application/x-www-form-urlencoded

grant_type=authorization_code&code=SplxlOBeZQQYbYS6WxSbIA
&redirect_uri=https%3A%2F%2Fclient%2Eexample%2Ecom%2Fcb
```
E步骤中，认证服务器发送的HTTP回复，包含以下参数：
- access_token：表示访问令牌，必选项。
- token_type：表示令牌类型，该值大小写不敏感，必选项，可以是bearer类型或mac类型。
- expires_in：表示过期时间，单位为秒。如果省略该参数，必须其他方式设置过期时间。
- refresh_token：表示更新令牌，用来获取下一次的访问令牌，可选项。
- scope：表示权限范围，如果与客户端申请的范围一致，此项可省略。

例如：
```http
HTTP/1.1 200 OK
Content-Type: application/json;charset=UTF-8
// 响应头中指定不缓存结果
Cache-Control: no-store 
Pragma: no-cache
{
  "access_token":"2YotnFZFEjr1zCsicMWpAA",
  "token_type":"example",
  "expires_in":3600,
  "refresh_token":"tGzv3JOkF0XG5Qx2TlKWIA",
  "example_parameter":"example_value"
}
```

#### 更新令牌 (`refresh token`)
如果用户访问的时候，客户端的"访问令牌"已经过期，则需要使用"更新令牌"向 AS 申请一个新的访问令牌  
客户端发出更新令牌的HTTP请求，包含以下参数：
- granttype：表示使用的授权模式，此处的值固定为"refreshtoken"，必选项。
- refresh_token：表示早前收到的更新令牌，必选项。
- scope：表示申请的授权范围，不可以超出上一次申请的范围，如果省略该参数，则表示与上一次一致。
```http
POST /token HTTP/1.1
Host: server.example.com
Authorization: Basic czZCaGRSa3F0MzpnWDFmQmF0M2JW
Content-Type: application/x-www-form-urlencoded
grant_type=refresh_token&refresh_token=tGzv3JOkF0XG5Qx2TlKWIA
```

## 双因素认证 (2FA)
能认证身份的三个因素：
- 秘密信息：只有该用户知道、其他人不知道的某种信息，比如密码。
- 个人物品：该用户的私人物品，比如身份证、钥匙。
- 生理特征：该用户的遗传特征，比如指纹、相貌、虹膜等等。
常规中单纯的密码有泄露的可能，需要除此外的另一重因素辅助验证  
常用的双因素组合是密码 + 某种个人物品，比如网上银行的 U 盾。用户插上 U 盾，再输入密码，才能登录网上银行。  
但是，U 盾一般不会随身携带且易遗失，手机才是最好的替代品。密码 + 手机就成了最佳的双因素认证方案。  
用户名密码 + 手机验证码的双因素认证被广泛采纳，国内的很多网站要求，用户输入密码时，还要提供短消息发送的验证码，以证明用户确实拥有该手机。  

短信验证码方式虽然方便，但是发送短信服务提高了成本，并且短信也有可能被拦截泄露。因此，安全的双因素认证不是密码 + 短消息，而是TOTP。  

### TOTP （Time-based One-time Password）

TOTP 的全称是"基于时间的一次性密码"（Time-based One-time Password）。它是公认的可靠解决方案，已经写入国际标准 RFC6238。  
它的步骤如下
1. 用户开启双因素认证后，服务器生成一个密钥。
2. 服务器提示用户扫描二维码（或者使用其他方式），把密钥保存到用户的手机。也就是说，服务器和用户的手机，现在都有了同一把密钥。
3. 用户登录时，手机客户端使用这个密钥和当前时间戳，生成一个哈希，有效期默认为30秒。用户在有效期内，把这个哈希提交给服务器。
4. 服务器也使用密钥和当前时间戳，生成一个哈希，跟用户提交的哈希比对。只要两者不一致，就拒绝登录。

#### TOTP 的算法
手机客户端和服务器，如何保证30秒期间都得到同一个哈希？
公式如下：​​​​​​​  
`TC = floor((unixtime(now) − unixtime(T0)) / TS)`  
上面的公式中，TC 表示一个时间计数器，unixtime(now)是当前 Unix 时间戳，unixtime(T0)是约定的起始时间点的时间戳，默认是0，也就是1970年1月1日。TS 则是哈希有效期的时间长度，默认是30秒。因此，上面的公式就变成下面的形式：  
`TC = floor(unixtime(now) / 30)`  
所以，只要在 30 秒以内，TC 的值都是一样的。前提是服务器和手机的时间必须同步。  
接下来，就可以算出哈希了。  
`TOTP = HASH(SecretKey, TC)`  
上面代码中，HASH就是约定的哈希函数，默认是 SHA-1。  
#### TOTP NodeJS 的实现
```js
// 安装 2fa模块
$ npm install --save 2fa

// 然后，生成一个32位字符的密钥
var tfa = require('2fa');

tfa.generateKey(32, function(err, key) {
  console.log(key);
});
// b5jjo0cz87d66mhwa9azplhxiao18zlx

// 生成哈希
var tc = Math.floor(Date.now() / 1000 / 30);
var totp = tfa.generateCode(key, tc);
console.log(totp); // 683464
```

## 参考资料
- [本人内部分享PPT - Web ​​常见的认证机制](https://ppt.baomitu.com/d/ed72d6a3)
