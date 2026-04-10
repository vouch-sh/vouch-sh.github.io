---
title: "Spring Boot (Spring Security)"
description: "Integrate Vouch OIDC authentication into a Spring Boot application using Spring Security OAuth2."
weight: 8
params:
  category: "server"
  language: "Java"
---

See the [Applications overview](/docs/applications/) for prerequisites, configuration endpoints, and available scopes.

Spring Security OAuth2 Client provides OIDC auto-discovery and PKCE support via `OAuth2AuthorizationRequestCustomizers.withPkce()`.

## Example

**[web/spring-boot](https://github.com/vouch-sh/examples/tree/main/web/spring-boot)** -- Complete working example with Spring Security OIDC configuration and PKCE.
