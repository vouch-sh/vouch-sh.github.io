# Spring Boot (Spring Security)

> Integrate Vouch OIDC authentication into a Spring Boot application using Spring Security OAuth2.

Source: https://vouch.sh/docs/applications/spring-boot/
Last updated: 2026-04-10

---


See the [Applications overview](/docs/applications/) for prerequisites, configuration endpoints, and available scopes.

[Spring Security OAuth2 Client](https://docs.spring.io/spring-security/reference/servlet/oauth2/client/index.html) provides OIDC auto-discovery. Key configuration:

- Add [`spring-boot-starter-oauth2-client`](https://docs.spring.io/spring-boot/reference/web/spring-security.html#web.security.oauth2.client) dependency
- Configure `spring.security.oauth2.client.provider.vouch.issuer-uri` for auto-discovery
- Set `authorization-grant-type: authorization_code` and `scope: openid,email`
- Enable PKCE with `OAuth2AuthorizationRequestCustomizers.withPkce()`
- Hardware attestation claims (`hardware_verified`, `hardware_aaguid`) are in the access token JWT — decode the payload to read them

## Example

**[web/spring-boot](https://github.com/vouch-sh/examples/tree/main/web/spring-boot)** — Complete working example with Spring Security OIDC, PKCE, and hardware claim extraction.
