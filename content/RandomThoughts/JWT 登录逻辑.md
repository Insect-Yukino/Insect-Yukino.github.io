+++
date = '2025-09-24T22:28:00+08:00'
draft = false
title = 'JWT 登录逻辑'
+++

在使用 spring security 处理登录逻辑时，使用的是表单登录逻辑，前端向后端 `/login` 地址提交一个登录表单，其中包含 `username`、`password` 等信息。UsernamePasswordAuthenticationFilter 就会拦截这个指定的登录地址，然后进行身份验证

> UsernamePasswordAuthenticationFilter 拦截请求之后，会解析请求，什么请求可以解析出 username 和 password 呢，只有是**表单提交**（`application/x-www-form-urlencoded` 或 `multipart/form-data`）UsernamePasswordAuthenticationFilter  才能够自己解析账号和密码并进行身份认证。如果是前后端项目请求是 `application/json`，发送的是 json 体，它就不能解析出用户信息并且进行身份认证。这时就需要自己在 controller 层中手动调用
>
> ```java
> Authentication authenticate = authenticationManager.
>     authenticate(new UsernamePasswordAuthenticationToken(request.getUsername(), request.getPassword()));
> SecurityContextHolder.getContext().setAuthentication(authenticate);
> ```

如果是使用 JWT 进行的登录

一般来说是不会使用表单的，它会使用短信登录，前端会先向后端 `/auth/sms/send` 提交一个手机号，该地址对应的 controller 层内部会调用阿里云相关 api，阿里云将验证码发送给客户端。当客户端拿到验证码输入之后再次提交，这次会向 `/auth/sms/login` 提交一个请求，该请求对应的 controller 内部会生成一个 jwt 返回给前端，前端拿到这个 token 保存起来，之后的访问就会通过自定义的 `JwtAuthenticationFilter` 拦截器获取请求携带 jwt 的信息，然后生成相应的 security 

```properties
# Redis
spring.redis.host=127.0.0.1
spring.redis.port=6379

# 阿里云短信（properties 模式）
aliyun.sms.access-key-id=你的AK
aliyun.sms.access-key-secret=你的SK
aliyun.sms.region-id=cn-hangzhou
aliyun.sms.sign-name=你的短信签名
aliyun.sms.template-code=SMS_123456789  # 模板需包含占位符 ${code}

# JWT
security.jwt.secret=changeThisToA256bitSecret_very_secret_and_long_123456
security.jwt.expiration-minutes=60
```

配置类

```java
@Component
@ConfigurationProperties(prefix = "aliyun.sms")
public class AliyunSmsProperties {
    private String accessKeyId;
    private String accessKeySecret;
    private String regionId;
    private String signName;
    private String templateCode;
    // getters/setters
}
```

```java
@Component
@ConfigurationProperties(prefix = "security.jwt")
public class JwtProperties {
    private String secret;
    private int expirationMinutes;
    // getters/setters
}
```

阿里云短信服务，发送验证码

```java
@Service
@RequiredArgsConstructor
public class AliyunSmsService {
    private final AliyunSmsProperties props;

    private Client client() throws Exception {
        Config config = new Config()
                .setAccessKeyId(props.getAccessKeyId())
                .setAccessKeySecret(props.getAccessKeySecret());
        config.endpoint = "dysmsapi.aliyuncs.com";
        return new Client(config);
    }

    public void sendCode(String phone, String code) throws Exception {
        SendSmsRequest req = new SendSmsRequest()
                .setPhoneNumbers(phone)
                .setSignName(props.getSignName())
                .setTemplateCode(props.getTemplateCode())
                .setTemplateParam("{\"code\":\"" + code + "\"}");
        SendSmsResponse resp = client().sendSms(req);
        if (!"OK".equalsIgnoreCase(resp.getBody().getCode())) {
            throw new RuntimeException("短信发送失败: " + resp.getBody().getMessage());
        }
    }
}
```

JWT 工具

```java
@Component
public class JwtTokenProvider {
    private final SecretKey key;
    private final long expireMs;

    public JwtTokenProvider(JwtProperties props) {
        this.key = Keys.hmacShaKeyFor(props.getSecret().getBytes(StandardCharsets.UTF_8));
        this.expireMs = props.getExpirationMinutes() * 60_000L;
    }

    public String createToken(String subject, Map<String,Object> claims) {
        long now = System.currentTimeMillis();
        return Jwts.builder()
                .setSubject(subject)
                .addClaims(claims)
                .setIssuedAt(new Date(now))
                .setExpiration(new Date(now + expireMs))
                .signWith(key, SignatureAlgorithm.HS256)
                .compact();
    }

    public Jws<Claims> parse(String token) {
        return Jwts.parserBuilder().setSigningKey(key).build().parseClaimsJws(token);
    }
}
```

JWT 鉴权过滤器，用来校验登录后请求

```java
public class JwtAuthenticationFilter extends OncePerRequestFilter {
    private final JwtTokenProvider jwt;

    public JwtAuthenticationFilter(JwtTokenProvider jwt) { this.jwt = jwt; }

    @Override
    protected void doFilterInternal(HttpServletRequest req, HttpServletResponse res, FilterChain chain)
            throws ServletException, IOException {
        String auth = req.getHeader(HttpHeaders.AUTHORIZATION);
        if (StringUtils.hasText(auth) && auth.startsWith("Bearer ")) {
            String token = auth.substring(7);
            try {
                Claims claims = jwt.parse(token).getBody();
                String subject = claims.getSubject(); // 这里我们用 phone 作为 subject
                Object rolesObj = claims.get("roles");
                List<SimpleGrantedAuthority> authorities = Stream
                        .of(String.valueOf(rolesObj == null ? "" : rolesObj).split(","))
                        .filter(StringUtils::hasText)
                        .map(r -> new SimpleGrantedAuthority("ROLE_" + r.trim()))
                        .toList();

                SecurityContextHolder.getContext().setAuthentication(
                        new UsernamePasswordAuthenticationToken(subject, null, authorities));
            } catch (Exception ignored) {
                // 无效/过期 token：保持匿名，让后续链路返回 401/403
            }
        }
        chain.doFilter(req, res);
    }
}
```

security 配置

```java
@Configuration
public class SecurityConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http, com.example.security.jwt.JwtTokenProvider jwt) throws Exception {
        return http
            .csrf(csrf -> csrf.disable())
            .cors(Customizer.withDefaults())
            .sessionManagement(sm -> sm.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
            .authorizeHttpRequests(reg -> reg
                .requestMatchers(HttpMethod.OPTIONS, "/**").permitAll()
                .requestMatchers("/auth/sms/send", "/auth/sms/login").permitAll()
                .anyRequest().authenticated()
            )
            .addFilterBefore(new com.example.security.jwt.JwtAuthenticationFilter(jwt),
                    org.springframework.security.web.authentication.UsernamePasswordAuthenticationFilter.class)
            .build();
    }
}
```

短信登录

```java
@RestController
@RequestMapping("/auth/sms")
@RequiredArgsConstructor
public class SmsAuthController {
    private final StringRedisTemplate redis;
    private final AliyunSmsService smsService;
    private final JwtTokenProvider jwt;

    private static final String CODE_KEY = "sms:code:";
    private static final String LIMIT_KEY = "sms:limit:"; // 限流
    private static final SecureRandom RND = new SecureRandom();

    @PostMapping("/send")
    public ResponseEntity<?> send(@RequestParam String phone) {
        if (!StringUtils.hasText(phone) || !phone.matches("^1\\d{10}$")) {
            return ResponseEntity.badRequest().body(Map.of("message", "手机号格式不正确"));
        }
        // 每号 60s 发送一次
        Boolean ok = redis.opsForValue().setIfAbsent(LIMIT_KEY + phone, "1", 60, TimeUnit.SECONDS);
        if (Boolean.FALSE.equals(ok)) {
            return ResponseEntity.badRequest().body(Map.of("message", "发送过于频繁，请稍后重试"));
        }

        String code = String.format("%06d", RND.nextInt(1_000_000));
        redis.opsForValue().set(CODE_KEY + phone, code, Duration.ofMinutes(5));

        try {
            smsService.sendCode(phone, code);
            return ResponseEntity.ok(Map.of("message", "验证码已发送"));
        } catch (Exception e) {
            redis.delete(LIMIT_KEY + phone); // 失败解除限流
            return ResponseEntity.internalServerError().body(Map.of("message", "短信发送失败"));
        }
    }

    @PostMapping("/login")
    public ResponseEntity<?> login(@RequestBody LoginReq req) {
        if (!StringUtils.hasText(req.getPhone()) || !req.getPhone().matches("^1\\d{10}$")) {
            return ResponseEntity.badRequest().body(Map.of("message", "手机号格式不正确"));
        }
        if (!StringUtils.hasText(req.getCode())) {
            return ResponseEntity.badRequest().body(Map.of("message", "验证码不能为空"));
        }
        String key = CODE_KEY + req.getPhone();
        String val = redis.opsForValue().get(key);
        if (val == null) {
            return ResponseEntity.status(401).body(Map.of("message", "验证码已过期"));
        }
        if (!val.equals(req.getCode())) {
            return ResponseEntity.status(401).body(Map.of("message", "验证码不正确"));
        }
        // 一次性
        redis.delete(key);

        // TODO 这里可查/建用户，绑定角色等；本例直接签发含 phone/roles 的 JWT
        String token = jwt.createToken(req.getPhone(), Map.of("roles", "USER"));
        return ResponseEntity.ok(Map.of("token", token, "tokenType", "Bearer"));
    }

    @Data
    public static class LoginReq {
        private String phone;
        private String code;
    }
}
```

**spring security 是如何识别当前用户已经登录了呢，就是通过 SecurityContextHolder.getContext() 有没有认证数据来判断的**
