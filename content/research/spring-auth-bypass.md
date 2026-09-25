---
title: "The Trouble with Spring Authorization Bypasses"
date: 2026-08-01
tags: ["java", "spring", "source-code-analysis", "auth-bypass"]
summary: "Starting from the routing-matching source in DispatcherServlet, this explains why authorizing on getRequestURI/getRequestURL — or blacklist-validating paths — in Spring is easy to bypass."
toc: true
draft: false
---

## A Quick Primer on Spring and Spring Boot

![Spring framework release timeline](/img/research/spring-auth-bypass/image1.png)

The Spring framework was officially released in 2003. It's a lightweight Java development framework that solves loose-coupling problems between the business logic layer and other layers, and carries interface-oriented programming philosophy through the entire application stack.

Spring's two core pillars are IoC (Inversion of Control) and AOP (Aspect Oriented Programming). After 20+ years of development, Spring is now in its 6.0 era, though the versions still most commonly deployed domestically are 4.0 and 5.0.

Spring Boot is a curated bundle of Spring components — pre-assembled so you can stand up a Spring-based web system in very little time. The version mapping between the two is:

| Spring Boot version | Spring version |
|---|---|
| 3.x | 6.x |
| 2.x | 5.x |
| 1.x | 4.x |

The source analyzed in this post is based on Spring Boot 2.2.0.RELEASE (corresponding to Spring Framework 5.2.0.RELEASE), with Tomcat as the default web container.

## Source-Code Deep Dive: The Full Route-Matching Flow

`org.springframework.web.servlet.DispatcherServlet` is the core entry point Spring uses to dispatch routes.

![DispatcherServlet dispatch entry point](/img/research/spring-auth-bypass/image4.png)

Inspecting `this.handlerMappings` turns up 8 different `HandlerMapping` implementations:

![Viewing handlerMappings in the debugger — 8 implementations in total](/img/research/spring-auth-bypass/image5.png)

| Class | Purpose |
|---|---|
| `springfox.documentation.spring.web.PropertySourcedRequestMappingHandlerMapping` | Configuration-driven URL handler, e.g. Swagger |
| `org.springframework.boot.actuate.endpoint.web.servlet.WebMvcEndpointHandlerMapping` | Used by the Actuator monitoring framework |
| `org.springframework.boot.actuate.endpoint.web.servlet.ControllerEndpointHandlerMapping` | Used by the Actuator monitoring framework |
| `org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerMapping` | Endpoints implemented via the `@RequestMapping` annotation |
| `org.springframework.boot.autoconfigure.web.servlet.WelcomePageHandlerMapping` | Default welcome-page handler |
| `org.springframework.web.servlet.handler.BeanNameUrlHandlerMapping` | URL handler keyed off bean names |
| `org.springframework.web.servlet.function.support.RouterFunctionMapping` | Function-based URL handler |
| `org.springframework.web.servlet.handler.SimpleUrlHandlerMapping` | Path-expression-based URL handler |

### getHandlerInternal → lookupPath

When the mapping is a `RequestMappingHandlerMapping`, execution enters `mapping.getHandler`, then `getHandlerInternal`, and eventually `getLookUpPathForRequest`.

When `alwaysUseFullPath` is enabled, Spring looks up the full path. That setting defaults to `false` on Spring Boot ≤ 2.3.0.RELEASE, in which case execution continues on into `getPathWithinApplication` → `getRequestUri`.

### The URL Sanitization Chain: decodeAndCleanUriString

`decodeAndCleanUriString` runs the following processing chain, in order:

1. **`removeSemicolonContent`** — strips `;` and everything after it from the URL, so `/admin;xxx` becomes `/admin`.
2. **`decodeRequestString`** — URL-decodes the path, so `/%61%64%6d%69%6e` becomes `/admin`.
3. **`getSanitizedPath`** — collapses redundant `/` characters, so `///admin` becomes `/admin`.
4. Strips the application's context path.

Now look at what's left of `getPathWithinServletMapping`: it ultimately calls the web container's own `getServletPath` method, whose return value differs across containers. On Tomcat, for example, given the input `/api/..;/test`, this method returns the normalized URL — `/test`. In that case `getPathWithinServletMapping` typically returns an empty string, and the final URL is whatever `getPathWithinApplication` produced.

### lookupHandlerMethod: "Smart" Best-Match

Having walked through how `lookupPath` is resolved, back in `AbstractHandlerMethodMapping.getHandlerInternal`: once `lookupPath` is obtained, execution enters `lookupHandlerMethod`, which looks up the matching handler method using Spring's formatted request path. Every `HandlerMapping` except `PropertySourcedRequestMappingHandlerMapping` runs this default `lookupHandlerMethod` logic.

The core of this method is **best-match**: even when the URL a user requests doesn't exactly match what's defined in `@RequestMapping`, Spring can still "intelligently" infer the intended handler. A classic case: requesting `/admin/` will, via closest-match resolution, still hit a route defined as `/admin`.

## Exploitation and Remediation

### Scenario 1: Authorizing on getRequestURI / getRequestURL

On Tomcat, the URL obtained via `getRequestURI` / `getRequestURL` is the **raw** request URL — untouched by any processing or transformation. But as the source analysis above showed, Spring automatically strips special characters from URLs and URL-decodes them. So if the authorization logic reads from `getRequestURI` / `getRequestURL` while route matching reads from Spring's processed path, the two can end up with different "interpretations" of the same URL — and that mismatch is exactly what produces an authorization bypass.

Common bypass techniques:

- **Inserting junk characters**: adding `;`, `/`, etc. into the path.
- **URL encoding**: percent-encoding the path.
- **Directory traversal** (Spring Boot ≤ 2.3.0.RELEASE): using sequences like `..;/`.

In a local test environment, here's `getRequestURI`-based authorization plus a semicolon bypass in practice — a plain request to `/admin` gets blocked, but inserting a semicolon still returns `you are admin`:

![GET /;/admin bypasses authorization, response returns you are admin](/img/research/spring-auth-bypass/image26.png)

URL encoding works the same way — `/%61%64%6d%69%6e` decodes to `/admin`, and bypasses just as easily:

![GET /%61%64%6d%69%6e encoded bypass, response also returns you are admin](/img/research/spring-auth-bypass/image28.png)

For example, requesting `/;//api/../admin` produces different results from `getRequestURI` and `getServletPath` — the former is the untouched raw string, the latter is Tomcat's normalized result.

**Remediation**: prefer `getServletPath` for obtaining the request path. Its return value has already been normalized by Tomcat, which tracks much more closely with how Spring cleans and transforms paths — avoiding the authorization bypass that results from the two sides disagreeing.

### Scenario 2: Loose URL Blacklist Validation

Because Spring's "best route match" algorithm exists, a request to `/admin/` can also match a route defined as `/admin`. Common bypass techniques:

- Appending special characters to the end of the path, such as `/`.
- Appending a `.xxx` suffix to the end of the path (effective when Spring Boot ≤ 1.5.22.RELEASE and `useSuffixPatternMatch` defaults to `true`).

**Remediation**: from a security standpoint, blacklists are never the preferred mechanism for access control. If a blacklist fallback is genuinely necessary, make the validation logic stricter — for instance, swap a naive `equals` check for a `contains` check.

## Summary

Spring is a mainstream, excellent Java framework with substantial capability. Improper use by developers can introduce plenty of security issues — but strictly speaking, this isn't a vulnerability in the framework itself. It's a logic flaw that arises because developer logic isn't rigorous enough, letting Spring and the web container disagree on how the same URL should be parsed.

From a security standpoint: never make a blacklist your primary mechanism for access-control policy.
