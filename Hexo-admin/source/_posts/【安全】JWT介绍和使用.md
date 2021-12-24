---
title: 【安全】JWT介绍和使用
date: 2021-12-24 11:11:49
categories: 安全
tags: [JWT, 签名]
---

### Why use it?

> JWT可以存储用户基本信息，常用于微服务中对请求的鉴权。
>
> JWT是无状态的，方便后端服务的水平扩展

<!-- more --> 

### 一、JWT的构成

一共三部分，第一部分我们称它为头部（header),第二部分我们称其为载荷（payload)，第三部分是签名（signature).

#### 1.header

jwt的头部承载两部分信息：

- 声明类型，这里是jwt
- 声明加密的算法 通常直接使用 HMAC SHA256

完整的头部就像下面这样的JSON：

```json
{
  'typ': 'JWT',
  'alg': 'HS256'
}
```

然后将头部进行base64编码，构成了第一部分.

```json
eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9
```

#### 2.payload

载荷就是存放有效信息的地方。这个名字像是特指飞机上承载的货品，这些有效信息包含三个部分

- 标准中注册的声明
- 公共的声明
- 私有的声明

**标准中注册的声明** (建议但不强制使用) ：

- **iss**: jwt签发者
- **sub**: jwt所面向的用户
- **aud**: 接收jwt的一方
- **exp**: jwt的过期时间，这个过期时间必须要大于签发时间
- **nbf**: 定义在什么时间之前，该jwt都是不可用的
- **iat**: jwt的签发时间
- **jti**: jwt的唯一身份标识，主要用来作为一次性token,从而回避重放攻击

**公共的声明** ：
 公共的声明可以添加任何的信息，一般添加用户的相关信息或其他业务需要的必要信息.但不建议添加敏感信息，因为该部分在客户端可解密。

**私有的声明** ：
 私有声明是提供者和消费者所共同定义的声明，一般不建议存放敏感信息，因为base64只是进行编码，意味着该部分信息可以归类为明文信息。

定义一个payload:

```json
{
  "sub": "1234567890",
  "name": "John Doe",
  "admin": true
}
```

然后将其进行base64加密，得到JWT的第二部分。

```json
eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkpvaG4gRG9lIiwiYWRtaW4iOnRydWV9
```

#### 3.signature

签名可以校验payload里面的信息是否被篡改，它由编码过后的header和payload采用header里声明的算法使用secret key进行加密。

```json
signature = Crypto(secret, base64(header), base64(payload))
```

下面是个签名示例

```json
jbcOUQ2bbiYlfVtprEkaT_S6Y6yQnBDOAKDHIHjvl7g
```

如果你感兴趣，可以拷贝下面的JWT去[https://jwt.io](https://jwt.io/)进行解密.

```json
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiIxZGZlZThkOC05OGE1LTQzMTQtYjRhZS1mYjU1YzRiMTg4NDUiLCJlbWFpbCI6ImFyaWVsQGNvZGluZ2x5LmlvIiwibmFtZSI6IkFyaWVsIFdlaW5iZXJnZXIiLCJyb2xlIjoiVVNFUiIsImlhdCI6MTU5ODYwODg0OCwiZXhwIjoxNTk4NjA5MTQ4fQ.oa3ziIZAoVFdn-97rweJAjjFn6a4ZSw7ogIHA74mGq0
```

既然JWT不能对信息加密，那它是如何防止信息被篡改的呢？比如我把角色改为Admin。服务端对JWT进行校验的时候会 对header和payload再次使用secret key加密，和传进来的JWT第三部分进行比较，如果不匹配则JWT不合法。

### 二、常见的误解、问题和技巧

#### 1.短暂的有效期和主动失效

有效期较短的token是被高度推荐的，一些服务的token过期时间为5分钟在发布它们之后。

在token过期之后，auth server将会颁发一个包含最新的身份信息的access token，这个过程被叫做 token refresh。举个例子，如果用户角色从 Admin 变更为 User，拥有较短的生命周期的token可以保证用户的token包含最近的用户角色.

综上所述，较短的有效期的token有以下两个好处:

- 如果你的token被盗取，攻击者使用你的token的有效时间窗口将会很短
- JWT是无状态的，服务端无法主动令其失效，因此，较短的生命周期可以让我们在用户权限和角色方面保证强大的一致性

#### 2.Refresh Token

接上文，Refresh Token在很多使用JWT的系统的是必须要用的。

它的使用很简单，在最初authentication success的时候，给用户颁发两个token

- Access Token : 每个请求必须携带的JSON Web Token ，包含用户声明的信息
- Refresh Token : 这个特殊类型的token是被持久化保存在数据库中的，被Auth Server所持有。它经常不是JWT，而是一个独一无二的hash值。

我们已经知道，Access Token会携带在每个请求中并会在某个时间节点失效。然后前端将会发送一个Refresh请求并携带 Refresh Token。Auth Server将会生成一个新的Access Token返回给用户，用户使用这个token直到它过期，然后再刷新一遍，Over and over。

Refresh Token的有效期一般会长达数月，当它失效后，用户必须重新登录。

#### 4.Secret VS Private-Public Key

流行的 HS256 签名算法的缺点是在生成和验证的时候都需要访问密钥。对于单一程序而言，这不是一个问题，但如果你有一个由多个服务构建的分布式系统，彼此独立运行，你必须在两个糟糕的选择之间进行选择：

- 你可以选择使用专用服务进行 token 的生成和验证。从客户端接收 token 的任何服务，都需要调用身份验证服务来验证 token。对于繁忙的系统来说，这会在身份验证服务上产生性能问题。
- 你可以为所有需要从客户端接收 token 的服务配置密钥，以便它们可以验证 token，无需调用身份验证服务。但是，在多个地方使用密钥会增加其受到攻击的风险，并且一旦受到攻击，攻击者就可以生成有效 token 并模拟系统中的任何用户。

因此对于这些程序，最好将签名密钥安全存储在身份验证服务中，并且仅用于生成密钥，而其他所有服务无需访问密钥就可以验证这些 token。实际上，这可以通过公钥加密来实现。

公钥加密基于两个组件的加密密钥：一个公钥和一个私钥。在命名实现时，公钥组件可以自由共享。可以使用公钥加密来完成两个工作流程：

- 消息加密：如果你想要向某人发送加密消息，我可以使用他的公钥来加密。加密的消息只能用他的私钥来解密。
- 消息签名：如果我想要签署一条消息它来自我，我可以使用自己的私钥来生成签名。任何对验证消息感兴趣的人，都可以使用我的公钥来确认签名是否有效。

JWT 的签名算法实现了上面的第二种情况。使用服务器的私钥对 token 进行签名，然后任何使用服务器公钥的人都可以验证它们，任何想要拥有它的人都可以免费使用。举个例子来说下面，我使用 `RS256` 签名算法，它是 RSA-SHA256 的简写。

下一步是为程序生成公钥/私钥集 (通常称为 "秘钥对")。生成 RSA 密钥有几种不同的方法，但我喜欢使用 openssh 中的 ssh-keygen 工具：

```bash
$ ssh-keygen -t rsa -b 4096
Generating public/private rsa key pair.
Enter file in which to save the key (/home/miguel/.ssh/id_rsa): jwt-key
Enter passphrase (empty for no passphrase):
Enter same passphrase again:
Your identification has been saved in jwt-key.
Your public key has been saved in jwt-key.pub.
The key fingerprint is:
SHA256:ZER3ddV4/smE0rnoNesS+IwCNSbwu5SThfiWWtLYRVM miguel@MS90J8G8WL
The key's randomart image is:
+---[RSA 4096]----+
|       .+E. ....=|
|   .   + . .  ..o|
|    + o +   . oo |
|   . + O   . + ..|
|    = @ S . o + o|
|   o #   . o + o.|
|    * +   = o o  |
|   . . . . = .   |
|        .   o.   |
+----[SHA256]-----+
```

`ssh-keygen` 命令的 -t 选项定义了，我正在请求的秘钥对，-b 选项指定密钥大小为 4096 位，这被认为是一个非常安全的密钥长度。当你运行该命令时，系统提示你需要为秘钥对提供文件名，这里我在当前路径下使用 `jwt-key` 。然后，系统将提示你输入密码来保护密钥，密钥需要保留为空。

当运行完命令，你会在当前目录下获得两个文件，`jwt-key` 和 `jwt-key.pub`。前者是私钥，将用于生成 token 签名，因此你需要保存好它。特别是，你不应该将私钥提交到代码仓库，而应直接在服务器上安装 (如果你需要重建服务器，则应做好备份)。.pub 文件用于验证 token。由于此文件没有敏感信息，因此你可以在需要验证 token 的任何项目上随意添加该文件的副本。

使用此秘钥对生成 token 的过程与我之前的内容非常相似。我们先使用私钥创建一个新的 token:

```python
import jwt
private_key = open('jwt-key').read()
token = jwt.encode({'user_id': 123}, private_key, algorithm='RS256').decode('utf-8')
print(token)
```

然后再使用公钥来验证：

```python
import jwt
public_key = open('jwt-key.pub').read()
token = 'eyJ0eXAiOiJKV1QiLCJhbGciOiJSUzI1NiJ9.eyJ1c2VyX2lkIjoxMjN9.HT1kBSdGFAznrhbs2hB6xjVDilMUmKA-_36n1pLLtFTKHoO1qmRkUcy9bJJwGuyfJ_dbzBMyBwpXMj-EXnKQQmKlXsiItxzLVIfC5qE97V6l6S0LzT9bzixvgolwi-qB9STp0bR_7suiXaON-EzBWFh0PzZi7l5Tg8iS_0_iSCQQlX5MSJW_-bHESTf3dfj5GGbsRBRsi1TRBzvxMUB6GhNsy6rdUhwoTkihk7pljISTYs6BtNoGRW9gVUzfA2es3zwBaynyyMeSocYet6WJri97p0eRnVGtHSWwAmnzZ-CX5-scO9uYmb1fT1EkhhjGhnMejee-kQkMktCTNlPsaUAJyayzdgEvQeo5M9ZrfjEnDjF7ntI03dck1t9Bgy-tV1LKH0FWNLq3dCJJrYdQx--A-I7zW1th0C4wNcDe_d_GaYopbtU-HPRG3Z1SPKFuX1m0uYhk9aySvkec66NBfvV2xEgo8lRZyNxntXkMdeJCEiLF1UhQvvSvmWaWC-0uRulYACn4H-tZiaK7zvpcPkrsfJ7iR_O1bxMPziKpsM4b7c7tmsEcOUZY-IHEI9ibd54_A1O72i08sCWKT5CXyG70MAPqyR0MFlcV7IuDtBW3LCqyvfsDVk4eIj8VcSU1OKQJ1Gl-CTOHEyN-ncV3NslVLaT9Q1C4E7uK2QpS8z0'
payload = jwt.decode(token, public_key, algorithms=['RS256'])
print(payload)
```

如果你感兴趣可以使用我的公钥来验证：

```python
public_key = 'ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAACAQDKQ43JBiv6FwtMpq6hStz59tSUo8mEU2SwQSg6mGmm4RBZ7uudRSnSwcvGSv45J1rKx1GUSuGYs/To4Ehsney1utN3jUY0XPhSUgxgCIFz66R9eAu92pykWBcSF96mj753LsyBStqq1nCu1D0dEJOBQnz9J2Zzs0N71ZabEvb6hXE8e119j2LFEUQWfM+3+AzrgsVXsTFE8y3LKeWohzQRAY3cAsdPPKI2ayHtlW6DoQ9LLqnsj4YkCe0hDIP/unTyx5OijbD6r8AeAyY4EGIL3c3jYhla6pclSqyLyq7kI0lUAQq4IXS4DYdqNHX01fI57tZeim0+nM3/9H1vJC2EPCDZDRdC2reJ4go/AssyQGamlmlu21Xt3n7rthYWg1u6Y/wpabDBEQcoXJPrBwyuGK8VRQrUU16GMahSiTpHxwvKHBMVyghLPaC8SBf9/edJDxBIq9s2wpnNBdD8TohqAXQILnJt5QBbiG2L/3uMqrEg+udWURBOcVok9TnS6RTtvezM1dSlcYoNMhGYNLMKgBByMOM/bMqJg8udqWqPW2/i/eCNPTLRPqZR7Kdo/GGcMY9pSsDvbzM/IHdeV1JhTyRcSi0xJDCs2M3hmWLg1rJNf7wQZKISIXefIItrABRoF1MuKOsk2CgPnZJgjVhzjzSP7EmyzRAv0E8gCeCtDQ== root@VM-20-3-centos'
```

### 总结

推荐大家采用 RS256 非对称加密算法来生成和验证JWT Token，token的有效期要短一点，并且要有Refresh Token机制。

