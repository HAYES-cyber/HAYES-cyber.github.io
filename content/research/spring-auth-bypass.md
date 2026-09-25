---
title: "Spring 权限绕过的那些事儿"
date: 2026-08-01
tags: ["java", "spring", "source-code-analysis", "auth-bypass"]
summary: "从 DispatcherServlet 的路由匹配源码出发，讲清楚为什么在 Spring 框架下用 getRequestURI/getRequestURL 做鉴权、或用黑名单校验路径，容易被绕过。"
toc: true
draft: false
---

## Spring 与 Spring Boot 简介

![Spring 框架发展时间线](/img/research/spring-auth-bypass/image1.png)

Spring 框架在 2003 年正式推出，是一个轻量级的 Java 开发框架，解决了业务逻辑层和其他各层的松耦合问题，并将面向接口的编程思想贯穿整个系统应用。

Spring 框架的两大核心分别为 IOC（Inverse of Control，控制反转）和 AOP（Aspect Oriented Programming，面向切面编程）。经过长达 20 年的发展，Spring 框架已经进入 6.0 时代，目前国内在使用的版本主要还是 4.0 和 5.0。

Spring Boot 是 Spring 的组件集合，预组装了 Spring 的一系列组件，通过它可以在极短的时间内搭建一套基于 Spring 框架的 Web 系统。两者的版本对应关系如下：

| Spring Boot 版本 | Spring 版本 |
|---|---|
| 3.x | 6.x |
| 2.x | 5.x |
| 1.x | 4.x |

本文分析的源码基于 Spring Boot 2.2.0.RELEASE（对应 Spring Framework 5.2.0.RELEASE），Web 容器为默认的 Tomcat。

## 核心源码分析：路由匹配全流程

`org.springframework.web.servlet.DispatcherServlet` 是 Spring 框架分发路由的核心入口。

![DispatcherServlet 分发入口](/img/research/spring-auth-bypass/image4.png)

查看 `this.handlerMappings`，存在 8 个不同的 `HandlerMapping` 实现类：

![调试器中查看 handlerMappings，共 8 个实现类](/img/research/spring-auth-bypass/image5.png)

| 类名 | 作用 |
|---|---|
| `springfox.documentation.spring.web.PropertySourcedRequestMappingHandlerMapping` | 基于配置的 URL 处理器，例如 Swagger |
| `org.springframework.boot.actuate.endpoint.web.servlet.WebMvcEndpointHandlerMapping` | Actuator 监控框架使用 |
| `org.springframework.boot.actuate.endpoint.web.servlet.ControllerEndpointHandlerMapping` | Actuator 监控框架使用 |
| `org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerMapping` | 用户通过 `@RequestMapping` 注解实现的接口 |
| `org.springframework.boot.autoconfigure.web.servlet.WelcomePageHandlerMapping` | 默认欢迎页面处理器 |
| `org.springframework.web.servlet.handler.BeanNameUrlHandlerMapping` | 通过 bean 名称的 URL 处理器 |
| `org.springframework.web.servlet.function.support.RouterFunctionMapping` | 基于函数的 URL 处理器 |
| `org.springframework.web.servlet.handler.SimpleUrlHandlerMapping` | 基于路径表达式的 URL 处理器 |

### getHandlerInternal → lookupPath

当 mapping 为 `RequestMappingHandlerMapping` 时，进入 `mapping.getHandler` 方法，再进入 `getHandlerInternal`，最终进入 `getLookUpPathForRequest`。

`alwaysUseFullPath` 开启后 Spring 将使用全路径查找；该配置在 Spring Boot 版本 ≤ 2.3.0.RELEASE 时默认为 `false`，此时会继续进入 `getPathWithinApplication` → `getRequestUri`。

### URL 清理链：decodeAndCleanUriString

`decodeAndCleanUriString` 的处理链条依次是：

1. **`removeSemicolonContent`** — 清除 URL 中 `;` 及其后续部分，即 `/admin;xxx` 处理后变为 `/admin`。
2. **`decodeRequestString`** — 对 URL 做 URL 解码，即 `/%61%64%6d%69%6e` 处理后变为 `/admin`。
3. **`getSanitizedPath`** — 删除多余的 `/`，例如 `///admin` 处理后变为 `/admin`。
4. 剔除应用程序的上下文路径。

再看 `getPathWithinServletMapping` 剩下的逻辑：它最终会调用 Web 容器自己的 `getServletPath` 方法，该方法在不同容器下返回值有差异。以 Tomcat 为例，输入 `/api/..;/test` 时，该方法会返回 URL 规范化后的结果，即 `/test`；此时 `getPathWithinServletMapping` 通常返回空字符串，最终 URL 以 `getPathWithinApplication` 的结果为准。

### lookupHandlerMethod：“智能”的最佳匹配

分析完 `lookupPath` 的查找过程，回到 `AbstractHandlerMethodMapping.getHandlerInternal`：拿到 `lookupPath` 后会进入 `lookupHandlerMethod`，通过 Spring 格式化后的请求路径查找对应的处理方法。除 `PropertySourcedRequestMappingHandlerMapping` 外，其余 `HandlerMapping` 都执行默认的 `lookupHandlerMethod` 逻辑。

这个方法的核心是**最佳匹配**：即使用户访问的 URL 与 `@RequestMapping` 中定义的并不完全相同，Spring 也能"智能"推测出最终的处理器。典型场景是：请求 URL 为 `/admin/` 时，通过最近匹配，同样能命中路由定义为 `/admin` 的接口。

## 漏洞利用与修复

### 场景一：用 getRequestURI / getRequestURL 鉴权

在 Tomcat 容器下，通过 `getRequestURI` / `getRequestURL` 拿到的 URL 是请求的**原始** URL，未经过任何处理和转换。而前面的源码分析已经说明，Spring 框架会自动清除 URL 中的特殊字符并做 URL 解码。因此如果鉴权逻辑用的是 `getRequestURI` / `getRequestURL`，而路由匹配用的是 Spring 处理后的路径，两者对同一个 URL 的“理解”就可能不一致，从而产生权限绕过。

常见绕过方式：

- **添加无用字符**：在路径中插入 `;`、`/` 等。
- **URL 编码**：对路径做百分号编码。
- **目录穿越**（Spring Boot ≤ 2.3.0.RELEASE）：利用 `..;/` 之类的序列。

在本地测试环境中，`getRequestURI` 鉴权 + 分号绕过的实际效果如下——正常访问 `/admin` 会被拦截，插入分号后同样能拿到 `you are admin`：

![GET /;/admin 绕过鉴权，响应返回 you are admin](/img/research/spring-auth-bypass/image26.png)

URL 编码同理，`/%61%64%6d%69%6e` 解码后就是 `/admin`，一样能绕过：

![GET /%61%64%6d%69%6e 编码绕过，响应同样返回 you are admin](/img/research/spring-auth-bypass/image28.png)

例如访问 `/;//api/../admin` 时，`getRequestURI` 与 `getServletPath` 拿到的结果并不相同——前者是未处理的原始串，后者是 Tomcat 规范化后的结果。

**修复建议**：优先使用 `getServletPath` 获取请求路径，该方法拿到的 URL 已经过 Tomcat 规范化处理，和 Spring 框架对路径的清理与转换逻辑更接近，能避免因两边处理不一致导致的权限绕过。

### 场景二：不严格的 URL 黑名单校验

由于 Spring 框架存在“最佳路由匹配”算法，访问 `/admin/` 时也能匹配到路由定义为 `/admin` 的接口。常见绕过方式：

- 在路径尾部增加特殊字符，如 `/`。
- 在路径尾部增加 `.xxx` 后缀（Spring Boot ≤ 1.5.22.RELEASE 且 `useSuffixPatternMatch` 默认为 `true` 时有效）。

**修复建议**：从安全角度出发，任何时候都不建议优先用黑名单做访问策略控制；如果确实需要黑名单兜底，建议校验逻辑更严格，比如把简单的 `equals` 判断换成 `contains`。

## 总结

Spring 是一款主流、优秀的 Java 开发框架，功能非常强大。开发者使用不当时可能引入较多安全问题，但严格来说这并不属于框架本身的漏洞，而是由于开发者逻辑不够严谨，导致 Spring 框架与 Web 容器之间对同一 URL 的解析结果不一致，进而产生的逻辑漏洞。

从安全的角度，建议任何时候都不要优先通过黑名单的形式做访问策略控制。
