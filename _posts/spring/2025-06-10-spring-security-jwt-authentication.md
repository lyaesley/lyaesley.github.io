---
title: "Spring Security JWT 인증 구현 완벽 가이드"
excerpt: "Spring Boot 3.x와 Spring Security 6.x를 활용한 JWT 토큰 기반 인증 시스템 구현 방법을 단계별로 알아보겠습니다."
categories:
  - Spring
tags:
  - Spring Security
  - JWT
  - Authentication
  - Spring Boot
  - Security
toc: true
toc_sticky: true
date: 2025-06-10
last_modified_at: 2025-06-10
---

## 들어가며

현대 웹 애플리케이션에서 JWT(JSON Web Token)는 사용자 인증과 권한 부여의 표준이 되었습니다. 특히 RESTful API나 마이크로서비스 아키텍처에서 stateless한 특성 때문에 널리 사용되고 있습니다.

이번 포스트에서는 **Spring Boot 3.x**와 **Spring Security 6.x**를 활용하여 JWT 기반 인증 시스템을 구현하는 방법을 자세히 알아보겠습니다.

## JWT란?

JWT(JSON Web Token)는 당사자 간에 정보를 안전하게 전송하기 위한 컴팩트하고 자체 포함된 방법을 정의하는 개방형 표준(RFC 7519)입니다.

### JWT의 구조

JWT는 점(.)으로 구분된 세 부분으로 구성됩니다:

```
Header.Payload.Signature
```

1. **Header**: 토큰 타입과 해싱 알고리즘 정보
2. **Payload**: 클레임(claims) 정보
3. **Signature**: 토큰의 무결성을 검증하는 서명

## 프로젝트 설정

### 의존성 추가

`build.gradle`에 필요한 의존성을 추가합니다:

```gradle
dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-web'
    implementation 'org.springframework.boot:spring-boot-starter-security'
    implementation 'org.springframework.boot:spring-boot-starter-data-jpa'
    implementation 'org.springframework.boot:spring-boot-starter-validation'
    
    // JWT 관련 라이브러리
    implementation 'io.jsonwebtoken:jjwt-api:0.12.3'
    implementation 'io.jsonwebtoken:jjwt-impl:0.12.3'
    implementation 'io.jsonwebtoken:jjwt-jackson:0.12.3'
    
    // 데이터베이스
    runtimeOnly 'com.h2database:h2'
    
    testImplementation 'org.springframework.boot:spring-boot-starter-test'
    testImplementation 'org.springframework.security:spring-security-test'
}
```

## JWT 유틸리티 클래스 구현

### JwtTokenProvider

```java
@Component
@Slf4j
public class JwtTokenProvider {
    
    private final SecretKey secretKey;
    private final long validityInMilliseconds;
    
    public JwtTokenProvider(@Value("${jwt.secret}") String secret,
                           @Value("${jwt.validity}") long validityInMilliseconds) {
        this.secretKey = Keys.hmacShaKeyFor(secret.getBytes(StandardCharsets.UTF_8));
        this.validityInMilliseconds = validityInMilliseconds;
    }
    
    /**
     * JWT 토큰 생성
     */
    public String createToken(String username, Collection<? extends GrantedAuthority> authorities) {
        Claims claims = Jwts.claims().setSubject(username);
        claims.put("roles", authorities.stream()
                .map(GrantedAuthority::getAuthority)
                .collect(Collectors.toList()));
        
        Date now = new Date();
        Date validity = new Date(now.getTime() + validityInMilliseconds);
        
        return Jwts.builder()
                .setClaims(claims)
                .setIssuedAt(now)
                .setExpiration(validity)
                .signWith(secretKey, SignatureAlgorithm.HS512)
                .compact();
    }
    
    /**
     * JWT 토큰에서 사용자명 추출
     */
    public String getUsername(String token) {
        return Jwts.parserBuilder()
                .setSigningKey(secretKey)
                .build()
                .parseClaimsJws(token)
                .getBody()
                .getSubject();
    }
    
    /**
     * JWT 토큰에서 권한 정보 추출
     */
    public Collection<? extends GrantedAuthority> getAuthorities(String token) {
        Claims claims = Jwts.parserBuilder()
                .setSigningKey(secretKey)
                .build()
                .parseClaimsJws(token)
                .getBody();
        
        @SuppressWarnings("unchecked")
        List<String> roles = (List<String>) claims.get("roles");
        
        return roles.stream()
                .map(SimpleGrantedAuthority::new)
                .collect(Collectors.toList());
    }
    
    /**
     * JWT 토큰 유효성 검증
     */
    public boolean validateToken(String token) {
        try {
            Jwts.parserBuilder()
                .setSigningKey(secretKey)
                .build()
                .parseClaimsJws(token);
            return true;
        } catch (JwtException | IllegalArgumentException e) {
            log.debug("Invalid JWT token: {}", e.getMessage());
            return false;
        }
    }
}
```

## JWT 인증 필터 구현

### JwtAuthenticationFilter

```java
@Component
@RequiredArgsConstructor
public class JwtAuthenticationFilter extends OncePerRequestFilter {
    
    private static final String AUTHORIZATION_HEADER = "Authorization";
    private static final String BEARER_PREFIX = "Bearer ";
    
    private final JwtTokenProvider jwtTokenProvider;
    
    @Override
    protected void doFilterInternal(HttpServletRequest request, 
                                  HttpServletResponse response, 
                                  FilterChain filterChain) throws ServletException, IOException {
        
        String token = resolveToken(request);
        
        if (token != null && jwtTokenProvider.validateToken(token)) {
            Authentication authentication = getAuthentication(token);
            SecurityContextHolder.getContext().setAuthentication(authentication);
        }
        
        filterChain.doFilter(request, response);
    }
    
    private String resolveToken(HttpServletRequest request) {
        String bearerToken = request.getHeader(AUTHORIZATION_HEADER);
        if (StringUtils.hasText(bearerToken) && bearerToken.startsWith(BEARER_PREFIX)) {
            return bearerToken.substring(BEARER_PREFIX.length());
        }
        return null;
    }
    
    private Authentication getAuthentication(String token) {
        String username = jwtTokenProvider.getUsername(token);
        Collection<? extends GrantedAuthority> authorities = jwtTokenProvider.getAuthorities(token);
        
        UserDetails principal = User.builder()
                .username(username)
                .password("")
                .authorities(authorities)
                .build();
        
        return new UsernamePasswordAuthenticationToken(principal, token, authorities);
    }
}
```

## Security 설정

### SecurityConfig

```java
@Configuration
@EnableWebSecurity
@EnableMethodSecurity
@RequiredArgsConstructor
public class SecurityConfig {
    
    private final JwtAuthenticationFilter jwtAuthenticationFilter;
    private final JwtAuthenticationEntryPoint jwtAuthenticationEntryPoint;
    
    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }
    
    @Bean
    public AuthenticationManager authenticationManager(
            AuthenticationConfiguration config) throws Exception {
        return config.getAuthenticationManager();
    }
    
    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .csrf(AbstractHttpConfigurer::disable)
            .sessionManagement(session -> 
                session.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/auth/**").permitAll()
                .requestMatchers("/h2-console/**").permitAll()
                .anyRequest().authenticated())
            .exceptionHandling(ex -> 
                ex.authenticationEntryPoint(jwtAuthenticationEntryPoint))
            .addFilterBefore(jwtAuthenticationFilter, UsernamePasswordAuthenticationFilter.class);
        
        return http.build();
    }
}
```

## 인증 컨트롤러 구현

### AuthController

```java
@RestController
@RequestMapping("/api/auth")
@RequiredArgsConstructor
@Validated
public class AuthController {
    
    private final AuthenticationManager authenticationManager;
    private final JwtTokenProvider jwtTokenProvider;
    private final UserService userService;
    
    @PostMapping("/login")
    public ResponseEntity<TokenResponse> login(@Valid @RequestBody LoginRequest request) {
        Authentication authentication = authenticationManager.authenticate(
            new UsernamePasswordAuthenticationToken(request.getUsername(), request.getPassword())
        );
        
        UserDetails userDetails = (UserDetails) authentication.getPrincipal();
        String token = jwtTokenProvider.createToken(
            userDetails.getUsername(), 
            userDetails.getAuthorities()
        );
        
        return ResponseEntity.ok(new TokenResponse(token));
    }
    
    @PostMapping("/register")
    public ResponseEntity<Void> register(@Valid @RequestBody RegisterRequest request) {
        userService.createUser(request);
        return ResponseEntity.status(HttpStatus.CREATED).build();
    }
}
```

## DTO 클래스

### LoginRequest

```java
@Getter
@NoArgsConstructor
@AllArgsConstructor
public class LoginRequest {
    
    @NotBlank(message = "사용자명은 필수입니다.")
    private String username;
    
    @NotBlank(message = "비밀번호는 필수입니다.")
    private String password;
}
```

### TokenResponse

```java
@Getter
@AllArgsConstructor
public class TokenResponse {
    private String accessToken;
    private String tokenType = "Bearer";
    
    public TokenResponse(String accessToken) {
        this.accessToken = accessToken;
    }
}
```

## 설정 파일

### application.yml

```yaml
jwt:
  secret: mySecretKeyForJWTTokenGenerationThatShouldBeLongEnough
  validity: 86400000  # 24시간 (밀리초)

spring:
  datasource:
    url: jdbc:h2:mem:testdb
    driver-class-name: org.h2.Driver
    username: sa
    password: 
  
  jpa:
    hibernate:
      ddl-auto: create-drop
    show-sql: true
    properties:
      hibernate:
        format_sql: true

logging:
  level:
    org.springframework.security: DEBUG
    io.jsonwebtoken: DEBUG
```

## 테스트

### 회원가입 테스트

```bash
curl -X POST http://localhost:8080/api/auth/register \
-H "Content-Type: application/json" \
-d '{
  "username": "testuser",
  "password": "testpassword",
  "email": "test@example.com"
}'
```

### 로그인 테스트

```bash
curl -X POST http://localhost:8080/api/auth/login \
-H "Content-Type: application/json" \
-d '{
  "username": "testuser",
  "password": "testpassword"
}'
```

### 인증이 필요한 API 호출

```bash
curl -X GET http://localhost:8080/api/users/me \
-H "Authorization: Bearer YOUR_JWT_TOKEN"
```

## 보안 고려사항

### 1. Secret Key 관리

JWT 서명에 사용되는 Secret Key는 다음과 같이 관리해야 합니다:

- 환경변수를 통한 관리
- 충분한 길이 (최소 256비트)
- 정기적인 갱신

### 2. 토큰 만료 시간

- Access Token: 짧은 만료 시간 (15분 ~ 1시간)
- Refresh Token: 긴 만료 시간 (7일 ~ 30일)

### 3. HTTPS 사용

프로덕션 환경에서는 반드시 HTTPS를 사용하여 토큰 탈취를 방지해야 합니다.

## 결론

Spring Security와 JWT를 활용한 인증 시스템을 구현해보았습니다. 이 구조는 stateless한 특성으로 마이크로서비스 환경에 적합하며, 확장성이 뛰어납니다.

다음 포스트에서는 Refresh Token을 활용한 토큰 갱신 메커니즘과 보다 고급스러운 보안 기능들을 다뤄보겠습니다.

## 참고 자료

- [Spring Security Reference](https://docs.spring.io/spring-security/reference/)
- [JWT.io](https://jwt.io/)
- [JJWT Library](https://github.com/jwtk/jjwt)
