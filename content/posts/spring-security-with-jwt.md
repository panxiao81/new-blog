---
title:  使用 Spring Security 原生组件们实现的 JWT 验证登录
date: 2023-12-29T15:44:00+08:00
categories: ['Java', 'Spring']
tags: ['java', 'spring', 'springboot', 'springsecurity', 'oauth2']
draft: false
---

## TL;DR

这篇文章描述了不引入任何第三方库，只使用 Spring Security 及其配套组件实现 JWT 认证的模式。能够带来如下优势。

- 更通用，且无第三方库引入，可简单跟随 Spring Boot 大版本更新。
- 扩展性强，可作为单体应用使用，也可与每个微服务简单继承，后续更可简单修改配置即可兼容 OAuth2 与 OIDC，也可将登录模块扩展成单独服务。

## Intro

这是不一定会有下一篇文章的第一篇。

JWT Authentication 是目前微服务时代常用的用户认证方式。由于 [12 factors](https://12factor.net/) 建议 Web 服务应是无状态的，因此用户认证若使用 Session 则需要在其他持久化存储中维护状态。一种常用的手段是使用 Redis 存储 Session 信息，Web 服务从 Redis 中查找 Session 的 Key 来确认用户。

另一种方案是使用 Token 认证，客户端请求 API 时在 HTTP Header 中带上服务器端颁发的 Token，服务器通过该 Token 与授权服务请求用户信息。

常用的一种 Token 为 [JSON Web Token](https://datatracker.ietf.org/doc/html/rfc7519)，

> JSON Web Token (JWT) 是一种紧凑的、URL 安全的表示方式，用于在两方之间传输声明 (Claims)。JWT 中的声明被编码为一个 JSON 对象，该对象被用作 JSON Web Signature (JWS) 结构的有效载荷，或者作为 JSON Web Encryption (JWE) 结构的明文，使得声明可以被数字签名或用消息认证码 (MAC) 和/或加密来保护完整性。
> -- RFC7519

类似的解决方案还有 [SAML(Security Assertion Markup Language Tokens)](https://docs.oasis-open.org/security/saml/v2.0/saml-core-2.0-os.pdf) 与 [SMT(Simple Web Token)](https://learn.microsoft.com/en-us/previous-versions/azure/azure-services/hh781551(v=azure.100)?redirectedfrom=MSDN)

JWT 被 OAuth2 协议作为默认 Bearer Token 实现。因此可以利用 Spring Security 对 OAuth2 的集成实现 JWT 认证。

## 验证与 Decode JWT

Spring OAuth2 Resource Server 提供了 `BearerTokenAuthenticationFilter` 将 Bearer Token 解析为 `BearerTokenAuthenticationToken` 并使用 `AuthenticationManager` 接口进行认证。且默认已经提供 `JwtAuthenticationProvider`，我们只需要传入自定义的实现了 `JwtDecoder` 接口的 Bean 即可完成。验证完成后，Spring Security 上下文中的 `Authentication` 即为 `JwtAuthenticationToken`，其中的 `Principal` 即为 `JwtDecoder` 中 `decode` 方法返回的 `Jwt`，因此可以轻松的与 `@PreAuthorize` 等注解和 `SpEL` 表达式兼容。

可见，要实现 JWT 认证，不需要自行实现 Filter 等，只需要实现 `JwtDecoder` 即可。

同时，Spring Security 在 `spring-security-oauth2-jose` 中提供了一套使用 Nimbus 的 JWT 实现，该依赖已包含在 `spring-boot-starter-oauth2-resource-server` 中，无需再手动引入。

### 使用 Spring 提供的 `JwtDecoder` 实现。

在配置类中注入 `JwtDecoder` 的 Bean，若没有其他 `AuthenticationProvider` 实现类，则 Spring Boot 会自动注入 `JwtAuthenticationProvider`。

`JwtDecoder` 的实现类 [`NumbusJwtDecoder`](https://javadoc.io/doc/org.springframework.security/spring-security-oauth2-jose/latest/org/springframework/security/oauth2/jwt/NimbusJwtDecoder.html)支持三种方式构造，分别为传入 `JWK Set Uri`，传入非对称加密的 `Public Key`，以及对称加密的共享 `Key`。本例为简化实现，使用共享 Key 方式构造。

首先，为解耦，创建配置类，使得可以使用 `Properties` 进行变量配置。

```java
@Getter
@Setter
@ConfigurationProperties(prefix = "jwt")
@Component
public class JwtProperties {
    @Min(32)
    private String secret = "Please-Change-This-Not-Safety-Key-Please-Please";
    private String issuer;
    /*
    * for minutes, default: 30
    */
    private Long expireTime = 30L;
    private Long refreshExpireTime = 60L;
}
```

在配置类中注入，并构造 `JwtDecoder`

```java
@Configuration
public class SecurityConfiguration {
    @Bean
    public SecretKey jwtSecretKey(JwtProperties jwtProperties) {
        // https://docs.oracle.com/en/java/javase/21/docs/specs/security/standard-names.html#keygenerator-algorithms
        return new SecretKeySpec(jwtProperties.getSecret().getBytes(), "HmacSHA256");
    }

    @Bean
    public JwtDecoder jwtDecoder(SecretKey key) {
        return NimbusJwtDecoder.withSecretKey(key).macAlgorithm(MacAlgorithm.HS256)
                .build();
    }
}
```

这里使用了 `HS256` 算法进行签名来保证 JWT 不会被篡改。

由于未使用加密，因此 JWT 为明文。在保密性要求较高的场合请使用 RSA 非对称加密证书对 JWT 进行加密，或使用 `JWK Set` 以方便证书轮换。

接下来配置 `SecurityFilterChain` 即可。

```java
@Configuration
public class SecurityConfiguratuon {
    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http
                .httpBasic(AbstractHttpConfigurer::disable)
                .formLogin(AbstractHttpConfigurer::disable)
                .sessionManagement(c -> c.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
                .csrf(AbstractHttpConfigurer::disable)
                .cors(AbstractHttpConfigurer::disable)
                .authorizeHttpRequests(c -> c
                        .requestMatchers("/login/**").permitAll()
                        .requestMatchers("/error").permitAll()
                        .requestMatchers("/register").permitAll()
                        .anyRequest().authenticated()
                )
                .oauth2ResourceServer(c -> c.jwt(Customizer.withDefaults()));

        return http.build();
    }
}
```

## 简单的 Authentication Server 实现

显然，最好的方法依然是实现完整的 OAuth2 协议，但就简单的应用而言，只要保证有简单的登录注册，登录时发 JWT 的功能就够了，这里的 JWT 相当于只作为 Token 使用。

登录毫无疑问也是通过 Spring Security 进行扩展的，因此仍然是要实现传统的 `DaoAuthenticationProvider`，我基于 JPA 实现了一个通用版本的 `UserDetailsService`。

```java
public class JpaUserDetailsService<T extends UserDetails> implements UserDetailsService {
    private final UserDetailsRepository<T, ?> repository;

    public JpaUserDetailsManager(UserDetailsRepository<T, ?> userDetailsRepository) {
        this.repository = userDetailsRepository;
    }

    @Override
    public UserDetails loadUserByUsername(String username) throws UsernameNotFoundException {
        return repository.findByUsername(username)
                .orElseThrow(() -> new UsernameNotFoundException("User %s not found".formatted(username)));
    }
}
```

为了这个 Repository，写 Example 查询需要用反射拿到泛形的实际类型，但由于臭名昭著的类型擦除，实际是做不到的。因此不如写一个 Repository Interface 让 `UserDetails` 实现类的 Entity 从这个接口继承。

```java
@NoRepositoryBean
public interface UserDetailsRepository<T extends UserDetails, ID> extends JpaRepository<T, ID> {
    Optional<T> findByUsername(String username);
    boolean existsByUsername(String username);
}
```

当然另一个方法是构造 JpaUserDetailsService 的时候将 `Class<T extends UserDetails>` 实际类型传入，然后用反射解决问题，但我主张少用反射，而且这样也完全可以解决问题。

下一步是构造 JWT 的生成工具们。我看很多喜欢写静态工具类，我这里为了一定的灵活度，参考了 Spring Security Authentication Server 的实现做了简单的包装。

规定一个具有 `generate` 方法的 `JwtTokenGenerate` 接口。

```java
public interface OAuth2TokenGenerator<T extends OAuth2Token> {
    T generate(JwtTokenContext context);
}
```

为其创建两个实现类，第一个为 `JwtAccessTokenGenerator`，另一个为 `DelegatingTokenGenerator`，作为多 Generator 的聚合。

```java
public class JwtAccessTokenGenerator implements OAuth2TokenGenerator<Jwt> {
    private final JwtEncoder jwtEncoder;
    private final JwtProperties jwtProperties;

    public JwtAccessTokenGenerator(JwtEncoder jwtEncoder, JwtProperties jwtProperties) {
        this.jwtEncoder = jwtEncoder;
        this.jwtProperties = jwtProperties;
    }

    @Override
    public Jwt generate(JwtTokenContext context) {
        if (context.getType() == null ||
            !context.getType().equals(JwtTokenType.ACCESS_TOKEN)) {
            return null;
        }

        Instant issuedAt = Instant.now();
        Instant expiresAt = issuedAt.plus(jwtProperties.getExpireTime(), ChronoUnit.MINUTES);

        JwtClaimsSet.Builder claimsBuilder = JwtClaimsSet.builder();
        if (StringUtils.hasText(jwtProperties.getIssuer())) {
            claimsBuilder.issuer(jwtProperties.getIssuer());
        }

        claimsBuilder
                .subject(context.getAuthentication().getName())
                .issuedAt(issuedAt)
                .expiresAt(expiresAt)
                .id(UUID.randomUUID().toString())
                .notBefore(issuedAt)
                .claims((map) -> map.putAll(context.getClaims()));

        // TODO: Properties
        JwsAlgorithm jwsAlgorithm = MacAlgorithm.HS256;
        JwsHeader.Builder jwsHeaderBuilder = JwsHeader.with(jwsAlgorithm);


        return this.jwtEncoder.encode(JwtEncoderParameters.from(jwsHeaderBuilder.build(), claimsBuilder.build()));
    }
}
```

```java
public class JwtDelegatingTokenGenerator implements OAuth2TokenGenerator<OAuth2Token> {
    private final List<OAuth2TokenGenerator<OAuth2Token>> tokenGenerators;
    @SafeVarargs
    public JwtDelegatingTokenGenerator(OAuth2TokenGenerator<? extends OAuth2Token>... OAuth2TokenGenerators) {
        Assert.notEmpty(OAuth2TokenGenerators, "jwtTokenGenerators cannot be empty");
        Assert.noNullElements(OAuth2TokenGenerators, "jwtTokenGenerator cannot be null");
        this.tokenGenerators = Collections.unmodifiableList(asList(OAuth2TokenGenerators));
    }

    @SuppressWarnings("unchecked")
    private static List<OAuth2TokenGenerator<OAuth2Token>> asList(OAuth2TokenGenerator<? extends OAuth2Token>... oAuth2TokenGenerators) {
        List<OAuth2TokenGenerator<OAuth2Token>> list = new ArrayList<>();

        for (OAuth2TokenGenerator<? extends OAuth2Token> oAuth2TokenGenerator : oAuth2TokenGenerators) {
            list.add((OAuth2TokenGenerator<OAuth2Token>) oAuth2TokenGenerator);
        }

        return list;
    }

    @Override
    public OAuth2Token generate(JwtTokenContext context) {
        for (OAuth2TokenGenerator<OAuth2Token> tokenGenerator : tokenGenerators) {
            OAuth2Token token = tokenGenerator.generate(context);
            if (token != null) {
                return token;
            }
        }
        return null;
    }
}
```

这样即可在接口基础上扩展其他实现类，如 `RefreshTokenGenerator` 等。

用于构造 Token 的上下文 POJO 实现如下

```java
@Getter
@Builder
public class JwtTokenContext {
    private final JwtTokenType type;
    private final Map<String, Object> claims;
    private final Authentication authentication;

    public JwtTokenContext(JwtTokenType type, Map<String, Object> claims, Authentication authentication) {
        this.type = type;
        this.claims = claims;
        this.authentication = authentication;
    }

    public static class JwtTokenContextBuilder {
        public JwtTokenContext build() {
            Map<JwtTokenType, Function<JwtTokenContextBuilder, Void>> action = new HashMap<>();

            action.put(JwtTokenType.ACCESS_TOKEN, (b) -> {
                Assert.notNull(b.authentication, "Authentication cannot be null");
                if (b.claims == null) {
                    b.claims = new HashMap<>();
                }
                return null;
            });
            action.put(JwtTokenType.REFRESH_TOKEN, (builder) -> {
                Assert.notNull(builder.authentication, "Authentication cannot be null");
                return null;
            });

            action.get(this.type).apply(this);
            return new JwtTokenContext(this.type, this.claims, this.authentication);
        }
    }
}
```

这里重写了 lombok 生成的 Builder，增加了 Build 前的判断。

`JwtTokenType` 也是一个简单的 POJO

```java
@Data
public final class JwtTokenType implements Serializable {
    private final String value;

    public static final JwtTokenType ACCESS_TOKEN = new JwtTokenType("access_token");
    public static final JwtTokenType REFRESH_TOKEN = new JwtTokenType("refresh_token");
}
```

有了以上包装，就可以简单的实现业务逻辑了。

首先创建一些 Bean

```java
@Configuration
public class AuthorizationConfiguration {
    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http
                .httpBasic(AbstractHttpConfigurer::disable)
                .formLogin(AbstractHttpConfigurer::disable)
                .sessionManagement(c -> c.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
                .csrf(AbstractHttpConfigurer::disable)
                .cors(AbstractHttpConfigurer::disable)
                .authorizeHttpRequests(c -> c
                        .requestMatchers("/login/**").permitAll()
                        .requestMatchers("/error").permitAll()
                        .requestMatchers("/register").permitAll()
                        .anyRequest().authenticated()
                );

        return http.build();
    }

    @Bean
    public SecretKey jwtSecretKey(JwtProperties jwtProperties) {
        // https://docs.oracle.com/en/java/javase/21/docs/specs/security/standard-names.html#keygenerator-algorithms
        return new SecretKeySpec(jwtProperties.getSecret().getBytes(), "HmacSHA256");
    }

    @Bean
    public JwtEncoder jwtEncoder(SecretKey key) {
        JWKSource<SecurityContext> secret = new ImmutableSecret<>(key);
        return new NimbusJwtEncoder(secret);
    }

    @Bean
    public OAuth2TokenGenerator<OAuth2Token> jwtTokenGenerator(JwtEncoder jwtEncoder, JwtProperties jwtProperties) {
        JwtAccessTokenGenerator jwtAccessTokenGenerator = new JwtAccessTokenGenerator(jwtEncoder, jwtProperties);
        return new JwtDelegatingTokenGenerator(jwtAccessTokenGenerator);
    }

    @Bean
    public BCryptPasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }

    @Bean
    public JpaUserDetailsService<UserEntity> userDetailsService(UserEntityRepository userEntityRepository) {
        return new JpaUserDetailsManager<>(userEntityRepository);
    }

    @Bean
    public DaoAuthenticationProvider daoAuthenticationProvider(UserDetailsService userDetailsService, PasswordEncoder passwordEncoder) {
        DaoAuthenticationProvider daoAuthenticationProvider = new DaoAuthenticationProvider();
        daoAuthenticationProvider.setPasswordEncoder(passwordEncoder);
        daoAuthenticationProvider.setUserDetailsService(userDetailsService);
        return daoAuthenticationProvider;
    }

    @Bean
    public AuthenticationManager authenticationManager(DaoAuthenticationProvider daoAuthenticationProvider, JwtAuthenticationProvider jwtAuthenticationProvider) {
        return new ProviderManager(daoAuthenticationProvider, jwtAuthenticationProvider);
    }
}
```

接下来根据业务创建 MVC 即可。

```java
@RestController
public class AuthController {
    private final AuthService authService;

    public AuthController(AuthService authService) {
        this.authService = authService;
    }

    @PostMapping("/login")
    public ResponseEntity<?> login(@RequestBody LoginRequest loginRequest) {
        LoginResponse response = authService.login(loginRequest);
        return ResponseEntity.status(HttpStatusCode.valueOf(200))
                .body(response);
    }

    @PostMapping("/register")
    public ResponseEntity<?> register(@RequestBody RegisterRequest registerRequest) {
        UserEntity user = authService.register(registerRequest);
        return ResponseEntity.status(HttpStatusCode.valueOf(201)).body(
                new RegisterResponse(user)
        );
    }

    @GetMapping("/user/me")
    public ResponseEntity<?> me(@AuthenticationPrincipal Jwt principal) {
        return ResponseEntity.ok(principal);
    }
}
```

```java
@Service
public class AuthService {
    private final AuthenticationManager authenticationManager;
    private final PasswordEncoder passwordEncoder;
    private final OAuth2TokenGenerator OAuth2TokenGenerator;
    private final UserEntityRepository userEntityReposity

    public AuthService(AuthenticationManager authenticationManager, PasswordEncoder passwordEncoder, OAuth2TokenGenerator OAuth2TokenGenerator, UserEntityRepository userEntityRepository) {
        this.authenticationManager = authenticationManager;
        this.passwordEncoder = passwordEncoder;
        this.OAuth2TokenGenerator = OAuth2TokenGenerator;
        this.userEntityRepository = userEntityRepository;
    }

    public LoginResponse login(LoginRequest loginRequest) {
        Authentication authentication = authenticationManager.authenticate(
                new UsernamePasswordAuthenticationToken(loginRequest.username(), loginRequest.password())
        );
        UserEntity user = (UserEntity) authentication.getPrincipal();

        JwtTokenContext access = JwtTokenContext.builder()
                .type(JwtTokenType.ACCESS_TOKEN)
                .claims(Map.of(
                        "authorities", user.getAuthorities()
                ))
                .authentication(authentication)
                .build();

        return new LoginResponse(user, OAuth2TokenGenerator.generate(access).getTokenValue());
    }

    public UserEntity register(RegisterRequest registerRequest) {
        UserEntity user = (UserEntity) UserEntity.builder()
                        .username(registerRequest.username())
                        .password(registerRequest.password())
                                .passwordEncoder(passwordEncoder::encode)
                                        .build();

        userEntityRepository.saveAndFlush(user);
        return user;
    }

}
```

> Github Repo: [panxiao81/spring-security-jwt-using-oauth2](https://github.com/panxiao81/spring-security-jwt-using-oauth2)
>
> Reference:
>
> [OAuth 2.0 Resource Server JWT :: Spring Security](https://docs.spring.io/spring-security/reference/servlet/oauth2/resource-server/jwt.html#oauth2resourceserver-jwt-decoder-algorithm)
>
> [spring-projects/spring-authorization-server](https://github.com/spring-projects/spring-authorization-server)
