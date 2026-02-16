---
title: "Spring Boot (Spring Security)"
description: "Integrate Vouch OIDC authentication into a Spring Boot application using Spring Security OAuth2."
weight: 8
params:
  category: "server"
  language: "Java"
---

> See the [Applications overview]({{< ref "/docs/applications" >}}) for prerequisites, configuration endpoints, and available scopes.

Add the [Spring Security](https://spring.io/projects/spring-security) OAuth2 Client dependency to your `pom.xml`:

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-oauth2-client</artifactId>
</dependency>
```

Configure the OIDC provider in `application.yml`:

```yaml
spring:
  security:
    oauth2:
      client:
        registration:
          vouch:
            client-id: ${VOUCH_CLIENT_ID}
            client-secret: ${VOUCH_CLIENT_SECRET}
            scope: openid, email
            authorization-grant-type: authorization_code
            redirect-uri: "{baseUrl}/login/oauth2/code/vouch"
        provider:
          vouch:
            issuer-uri: "{{< instance-url >}}"
```

Configure Spring Security in your application:

```java
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.annotation.web.configuration.EnableWebSecurity;
import org.springframework.security.web.SecurityFilterChain;

@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/", "/login").permitAll()
                .anyRequest().authenticated()
            )
            .oauth2Login(oauth2 -> oauth2
                .defaultSuccessUrl("/dashboard", true)
            );
        return http.build();
    }
}
```

Create a controller to access user information:

```java
import org.springframework.security.core.annotation.AuthenticationPrincipal;
import org.springframework.security.oauth2.core.oidc.user.OidcUser;
import org.springframework.stereotype.Controller;
import org.springframework.ui.Model;
import org.springframework.web.bind.annotation.GetMapping;

@Controller
public class DashboardController {

    @GetMapping("/dashboard")
    public String dashboard(@AuthenticationPrincipal OidcUser user, Model model) {
        model.addAttribute("name", user.getFullName());
        model.addAttribute("email", user.getEmail());
        model.addAttribute("sub", user.getSubject());
        return "dashboard";
    }
}
```
