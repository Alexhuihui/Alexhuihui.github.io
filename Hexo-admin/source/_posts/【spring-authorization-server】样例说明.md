---
title: 【spring-authorization-server】样例说明
date: 2022-02-23 15:21:05
categories: spring-authorization-server
tags: [oauth2, spring-authorization-server]
---

### 介绍

前段时间spring-security团队推出了新的基于Oauth2.0的授权服务器实现，叫做spring-authorization-server，让我们来探究一下里面example。

<!-- more --> 

client : 

客户端首先向授权服务器的授权节点（Authorization endpoint）发起授权请求，通过@RegisteredOAuth2AuthorizedClient("messaging-client-authorization-code")

```
OAuth2AuthorizedClientArgumentResolver
```

获取注解里面定义的 clientRegistrationId = messaging-client-authorization-code

```
DefaultOAuth2AuthorizedClientManager
```

在authorize()中去寻找OAuth2AuthorizedClient，发现没有之后再去寻找ClientRegistration，在InMemoryClientRegistrationRepository中可以找到clientRegistrationId对应的ClientRegistration。

![](https://raw.githubusercontent.com/Alexhuihui/photo/main/20220223155142.png)

然后用这个clientRegistration构建OAuth2AuthorizationContext，再使用DelegatingOAuth2AuthorizedClientProvider去调用authorizedClientProviders中的authorize()

```
AuthorizationCodeOAuth2AuthorizedClientProvider
```

抛出ClientAuthorizationRequiredException被OAuth2AuthorizationRequestRedirectFilter捕获

然后返回一个重定向地址：http://auth-server:9000/oauth2/authorize?response_type=code&client_id=messaging-client&scope=message.read%20message.write&state=NVdF9Kzi80STPhHHHs9BTv9oAJlODlzpFEFVtl_TI6g%3D&redirect_uri=http://127.0.0.1:8080/authorized





auth-server:

```
OAuth2AuthorizationEndpointFilter
```

处理授权请求，发现没有认证，然后就会经过DelegatingAuthenticationEntryPoint选择对应的LoginUrlAuthenticationEntryPoint返回401状态码

![](https://raw.githubusercontent.com/Alexhuihui/photo/main/20220224104403.png)