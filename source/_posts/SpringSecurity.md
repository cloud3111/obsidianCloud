---
banner: "[[pixel-banner-image.png]]"
title: SpringSecurity权限校验框架
tags:
  - SpringSecurity
  - CSRF
  - XSS
categories: 编程
date: 2025-09-09T21:05:00
---

	没有永远的朋友,只有永远的利益
# SpringSecurity
## 安全框架
- 是一个**对用户进行认证和授权的框架**, 类似的还有像Shrio框架等等
- 认证:  就是 对来访者进行**登陆校验和后面的认证校验**; 
- 授权:  是对用户的操作进行**权限校验**
- 其实认证的底层其实是**过滤器链的调用**, 使用了提供各种功能的过滤器: 核心过滤器链有
	- **UsernamePasswordAuthenticationFilter**: 处理填写账号密码后的登陆逻辑
	- **CsrfFilter**: 处理跨域问题的过滤器
	- **ExceptionTranslationFilter**: 处理过滤器链中发生的任何异常
	- **FilterSecurityInterceptor**: 负责权限校验的过滤器

## 认证流程
### 概念理解
- **Authentication接口**: 它的实现类UsernamePasswordAuthenticationToken，表示当前访问系统的用户，封装了用户名和密码.
- **AuthenticationManager接口**：定义了认证Authentication的方法叫authenticate
- **UserDetailsService接口**：加载用户特定数据的核心接口。UserDetailsServiceImpl里面定义了一个根据用户名查询用户信息的方法loadUserByUsername。
- **UserDetails接口**：用户实体类接口。通过UserDetailsService根据用户名获取处理的用户信息要封装成UserDetails对象返回通过用 LoginUser implements UserDetails。然后将这些信息封装到Authentication对象中。LoginUser loginUser = (LoginUser) authenticate.getPrincipal();
### 分类
- **登陆校验**: 用户首次点击登陆按钮之后的登陆操作
- **后续校验**: 用户在首次登陆后不需要每次都进行登陆校验, 只需要保存登陆凭证Token即可
### 流程
- 登陆: 也就是首次登陆时的进行的校验, 校验通过生成JWT令牌, 并将令牌存入Redis, 除此之外, 定义了一个UserDetailsService, 在这个实现类中去查询数据库
- 认证: 之后不需要每次都进行登陆校验, 只需要对Redis中的JWT令牌做校验就行, 使用userid去redis中获取对应的LoginUser对象, 然后**封装Authentication对象存入SecurityContextHolder,** 所以需要一个认证过滤器做这些事
## 授权流程
### 概念理解
- 在后台进行用户权限的判断，判断当前用户是否有相应的权限，必须具有所需权限才能进行相应的操作
- ​ **RBAC权限模型**（Role-Based Access Control）即：**基于角色的权限控制**。
### 流程
- 使用默认的**FilterSecurityInterceptor**来进行权限校验
- 在FilterSecurityInterceptor中会**从SecurityContextHolder获取其中的Authentication**，然后获取其中的权限信息
- 在项目中需要把当前登录用户的权限信息也存入Authentication
- 所以权限的查询是在UserDetailsService的loadUserByUsername方法中查到的
### 注解使用
- 使用@PreAuthorize注解，它内部其实是调用authentication的getAuthorities方法获取用户的权限列表。然后判断我们存入的方法参数数据在权限列表中
## 异常处理
- 认证失败或者是授权失败的情况下也能和我们的接口一样返回相同结构的json
- 出现了异常会被**ExceptionTranslationFilter**捕获到
- 如果是认证过程中出现的异常会被封装成AuthenticationException然后调用AuthenticationEntryPoint对象的方法去进行异常处理。
- ​ 如果是授权过程中出现的异常会被封装成AccessDeniedException然后调用AccessDeniedHandler对象的方法去进行异常处理。
- ​ 所以如果我们需要**自定义异常处理**，我们只需要自定义AuthenticationEntryPoint和AccessDeniedHandler然后配置给SpringSecurity即可
## CSRF攻击
### 概念理解
- 用户正常登陆网站a, 登陆成功后有一个只允许a访问的cookie, 只要cookie不失效, 在这个期间就可以携带cookie正常访问网站a
- 但是用户也登陆了网站b, 也有一个b的cookie, 在访问b的过程中有一个指向a的资源跳转, **使得用户携带a的cookie在不知情的情况下去访问a获取资源**, 这就是CSRF攻击
- CSRF攻击防御的重点是利用cookie的值只能被第一方读取，无法读取第三方的cookie值。这上面的案例中再b网站中去访问a网站, b网站的cookie就是第一方cookie
### 防御手段
1. 验证 HTTP **Referer** 字段(来源网站)
2. ​ SpringSecurity去防止CSRF攻击的方式就是通过**csrf_token**。后端会生成一个csrf_token，前端发起请求的时候需要携带这个csrf_token,后端会有过滤器进行校验，如果没有携带或者是伪造的就不允许访问
3. 在HTTP头中自定义属性并验证；
4. Chrome 浏览器端启用 **SameSite cookie**

- 总结: **CSRF攻击依靠的是cookie中所携带的认证信息。以前的项目是把token存储在cookie中, 请求时自动携带, 现在的项目都是存储在本地/会话存储中, 并且在请求的时候是手动设置token到请求头中, 所以没有csrf攻击的可能性**
## XSS攻击
### 概念理解
- 跟SQL注入有点类似, 两者都是把“攻击载荷”塞进应用的数据流中，最后在不该执行的地方被执行
### SQL注入
- 什么是SQL注入: SQL 注入是攻击者把恶意的 SQL 代码作为输入注入到应用中，使数据库在未经预期的情况下执行这些语句
	- `' OR '1'='1` —— 常见的“总是真”的条件，用来绕过认证（示意）
	- `'; DROP TABLE users; --` —— 示例说明拼接带来的风险（危险且破坏性强）
- 但是我们现在的数据库都是采用预编译语句, 所有不用担心SQL注入
### XSS
- 当带有js攻击脚本的评论被存储到数据库中时, 读取评论列表可能会有这条带有攻击的评论, 前端在进行数据展示的时候如果没有防护的话, 这段js代码就会把恶意执行造成攻击
- 如何应对 XSS 攻击?
	- 对输入进行过滤，过滤标签等，只允许合法值。
	- HTML 转义
	- 对于链接跳转，如`<a href="xxx"` >等，要校验内容，禁止以 script 开头的非法链接。
	- 限制输入长度