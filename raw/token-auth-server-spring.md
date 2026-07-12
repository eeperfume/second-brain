---
aliases: [코드로 보는 토큰 인증 서버, Spring 토큰 인증, RTR Spring 구현, refresh rotation 스프링 코드, 토큰 인증 서버 예제]
---

# 코드로 보는 토큰 인증 — 서버 (Spring Boot + Redis)

개념은 [[session-and-jwt]](access·refresh 분리), [[refresh-token-rotation]](회전·재사용 탐지), [[session-store]](Redis·TTL·denylist)에서 다뤘다. 여기서는 그 조각들을 하나의 Spring Boot 앱으로 엮어 코드로 본다. 클라이언트(React/Next) 쪽은 [[token-auth-client-react]]에서 이어 간다.

## 뭘 만드나

엔드포인트 네 개짜리 인증 서버다.

- `POST /auth/login` — access token(JWT)과 refresh token(불투명)을 함께 발급
- `GET /me` — access token으로 보호되는 요청
- `POST /auth/refresh` — refresh token을 회전(RTR)하고 새 access token을 준다. **재사용이면 패밀리째 무효화**
- `POST /auth/logout` — 그 로그인 사슬을 끊고, 원하면 지금 access token까지 즉시 차단

Redis에 두는 상태는 이게 전부다.

| 키 | 타입 | 값 | TTL |
|---|---|---|---|
| `rt:{refreshToken}` | Hash | `familyId`, `userId`, `status`(`active`\|`rotated`) | 14일 |
| `family:{familyId}` | Set | 그 패밀리에서 발급된 refresh token 전부 | 14일 |
| `denylist:{jti}` | String | `"1"` (존재 여부만 본다) | 그 access token의 **남은 수명** |

핵심은 access token은 Redis에 두지 않는다는 것이다. JWT라 서명만 검증하면 되고([[session-and-jwt]]), Redis를 보는 건 refresh token(상태 기계)과 denylist(선택적 즉시 차단)뿐이다.

## 0. 의존성과 설정

```groovy
// build.gradle
dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-web'
    implementation 'org.springframework.boot:spring-boot-starter-data-redis'
    implementation 'io.jsonwebtoken:jjwt-api:0.12.6'
    runtimeOnly    'io.jsonwebtoken:jjwt-impl:0.12.6'
    runtimeOnly    'io.jsonwebtoken:jjwt-jackson:0.12.6'
}
```

```yaml
# application.yml
spring:
  data:
    redis: { host: localhost, port: 6379 }
app:
  jwt:
    secret: ${ACCESS_SECRET}   # 실제 비밀키는 env·시크릿 스토어에서 (→ jwt-signing-and-keys)
    access-ttl: 900            # access token 수명, 15분(초)
  refresh:
    ttl: 1209600               # refresh token 수명, 14일(초)
```

`StringRedisTemplate`은 스타터가 자동 구성하므로 따로 만들 필요가 없다.

## 1. access token: 발급과 검증

access token은 HS256으로 서명한 JWT다(발급도 검증도 이 서버 하나라 대칭키로 충분하다 → [[jwt-signing-and-keys]]). `sub`에 사용자 id, `jti`에 토큰마다 다른 식별자를 넣는다. `jti`는 나중에 denylist로 이 토큰 하나만 콕 집어 죽일 때 쓴다. 검증을 다른 서비스나 제3자에 맡기는 구조(OAuth Resource Server 같은)로 가면 RS256+JWKS로 바꾸는데, 그 코드는 [[jwt-rs256-jwks-spring]]에서 잇는다.

```java
@Service
public class JwtService {

    private final SecretKey key;
    private final long accessTtl;

    public JwtService(@Value("${app.jwt.secret}") String secret,
                      @Value("${app.jwt.access-ttl}") long accessTtl) {
        this.key = Keys.hmacShaKeyFor(secret.getBytes(StandardCharsets.UTF_8));
        this.accessTtl = accessTtl;
    }

    public String issue(String userId) {
        Instant now = Instant.now();
        return Jwts.builder()
                .subject(userId)
                .id(UUID.randomUUID().toString())              // jti
                .issuedAt(Date.from(now))
                .expiration(Date.from(now.plusSeconds(accessTtl)))
                .signWith(key, Jwts.SIG.HS256)
                .compact();
    }

    // 서명·만료를 검증한다. 위조거나 만료면 JwtException을 던진다.
    public Claims parse(String token) {
        return Jwts.parser().verifyWith(key).build()
                .parseSignedClaims(token).getPayload();
    }
}
```

`parse`가 던지는 예외가 곧 "이 토큰 못 믿는다"이다. 서명이 안 맞거나 `exp`가 지났으면 예외로 튀고, 통과하면 payload를 신뢰한다. 저장소 조회가 없다는 점이 세션과 갈리는 지점이다.

## 2. refresh token: Redis에 상태를 두다

refresh token은 JWT가 아니라 **추측 불가능한 랜덤 문자열**이다([[refresh-token-rotation]]에서 "흔히 JWT가 아니다"라고 한 그 선택). 값 자체엔 정보가 없고, 진짜 상태(`familyId`, `userId`, `status`)는 Redis가 쥔다.

```java
@Service
public class RefreshTokenService {

    private final StringRedisTemplate redis;
    private final long refreshTtl;

    public RefreshTokenService(StringRedisTemplate redis,
                               @Value("${app.refresh.ttl}") long refreshTtl) {
        this.redis = redis;
        this.refreshTtl = refreshTtl;
    }

    public record Issued(String token, String familyId) {}

    // 로그인: 새 패밀리를 열고 첫 refresh token을 심는다.
    public Issued issueNewFamily(String userId) {
        return issue(userId, UUID.randomUUID().toString());
    }

    // 로그인·회전이 공통으로 쓰는 발급. 주어진 패밀리에 active 토큰 하나를 넣는다.
    private Issued issue(String userId, String familyId) {
        String token = randomToken();
        String key = "rt:" + token;
        redis.opsForHash().putAll(key, Map.of(
                "familyId", familyId, "userId", userId, "status", "active"));
        redis.expire(key, Duration.ofSeconds(refreshTtl));

        redis.opsForSet().add("family:" + familyId, token);      // 패밀리에 등록
        redis.expire("family:" + familyId, Duration.ofSeconds(refreshTtl));
        return new Issued(token, familyId);
    }

    private static String randomToken() {
        byte[] buf = new byte[32];
        new SecureRandom().nextBytes(buf);
        return Base64.getUrlEncoder().withoutPadding().encodeToString(buf);
    }
}
```

`family:{familyId}` Set이 나중에 "이 사슬에 속한 토큰 전부"를 한 번에 찾게 해 주는 장치다. 발급할 때마다 여기 등록해 둔다.

> 실무에선 여기서 한 겹 더 간다. `rt:` 키를 토큰 원문 대신 그 토큰의 SHA-256 해시로 잡으면, Redis가 뚫려도 원본 refresh token은 새지 않는다(비밀번호를 해시로 저장하는 것과 같은 이유). 조회할 때 들어온 토큰을 같은 방식으로 해시해서 찾으면 된다. 여기서는 흐름을 또렷이 보려고 원문 그대로 뒀다.

## 3. 로그인: 둘을 한꺼번에 발급

```java
@RestController
@RequestMapping("/auth")
public class AuthController {

    private final JwtService jwt;
    private final RefreshTokenService refresh;

    public AuthController(JwtService jwt, RefreshTokenService refresh) {
        this.jwt = jwt;
        this.refresh = refresh;
    }

    @PostMapping("/login")
    public ResponseEntity<?> login(@RequestBody LoginRequest req, HttpServletResponse res) {
        String userId = authenticate(req);          // 생략: 비밀번호 확인 후 사용자 id
        var issued = refresh.issueNewFamily(userId);

        setRefreshCookie(res, issued.token());       // refresh는 httpOnly 쿠키로
        return ResponseEntity.ok(Map.of("accessToken", jwt.issue(userId)));  // access는 본문으로
    }
    // ...
}
```

두 토큰을 서로 다른 통로로 보내는 게 핵심이다. access token은 응답 **본문**에 담아 클라이언트가 메모리에 들고 헤더로 실어 보내게 하고, refresh token은 **httpOnly 쿠키**에 심어 JS가 아예 만지지 못하게 한다([[session-and-jwt]]의 저장 위치 권고 그대로). 그래서 XSS가 나도 refresh token은 못 훔쳐 간다.

```java
private void setRefreshCookie(HttpServletResponse res, String token) {
    ResponseCookie cookie = ResponseCookie.from("refresh_token", token)
            .httpOnly(true)          // JS가 못 읽음 → XSS로 유출 안 됨
            .secure(true)            // HTTPS에서만
            .sameSite("Strict")      // CSRF 완화 (→ cookie-and-session)
            .path("/auth")           // /auth/* 요청에만 실린다. /me 등엔 안 붙음
            .maxAge(Duration.ofSeconds(1_209_600))
            .build();
    res.addHeader(HttpHeaders.SET_COOKIE, cookie.toString());
}
```

`path("/auth")`로 범위를 좁힌 게 의도적이다. refresh token은 오직 `/auth/refresh`와 `/auth/logout`에만 필요하니, `/me` 같은 일반 API에는 아예 딸려 가지 않게 해서 괜히 노출될 일을 줄인다.

## 4. 보호된 요청: JWT 필터

`/me` 같은 요청은 access token만 있으면 통과다. 필터에서 서명을 검증하고, 그다음 denylist에 올라온 `jti`인지만 확인한다.

```java
@Component
public class JwtAuthFilter extends OncePerRequestFilter {

    private final JwtService jwt;
    private final StringRedisTemplate redis;

    public JwtAuthFilter(JwtService jwt, StringRedisTemplate redis) {
        this.jwt = jwt;
        this.redis = redis;
    }

    @Override
    protected void doFilterInternal(HttpServletRequest req, HttpServletResponse res,
                                    FilterChain chain) throws ServletException, IOException {
        String header = req.getHeader("Authorization");
        if (header != null && header.startsWith("Bearer ")) {
            try {
                Claims c = jwt.parse(header.substring(7));           // 서명·만료 검증

                // (선택) 즉시 차단용 denylist. 평상시엔 이 한 줄이 유일한 Redis 조회다.
                if (Boolean.TRUE.equals(redis.hasKey("denylist:" + c.getId()))) {
                    res.sendError(HttpServletResponse.SC_UNAUTHORIZED);
                    return;
                }
                req.setAttribute("userId", c.getSubject());
            } catch (JwtException e) {
                res.sendError(HttpServletResponse.SC_UNAUTHORIZED);   // 위조·만료
                return;
            }
        }
        chain.doFilter(req, res);
    }
}
```

denylist 조회를 넣으면 access token 검증이 완전한 무상태에서 살짝 벗어난다([[session-store]]에서 "무상태를 일부 반납한다"고 한 그 지점). 즉시 로그아웃이 꼭 필요할 때만 켜는 선택지다. 필요 없으면 이 조회를 빼고 순수 서명 검증만 둔다.

## 5. 갱신 `/auth/refresh`: 여기가 RTR

여기가 이 노트의 핵심이다. refresh token이 오면 상태를 보고 세 갈래로 나뉜다.

```java
public sealed interface RefreshResult {
    record Rotated(String userId, RefreshTokenService.Issued next) implements RefreshResult {}
    record ReuseDetected() implements RefreshResult {}
    record Unknown()       implements RefreshResult {}
}
```

```java
// RefreshTokenService
public RefreshResult rotate(String token) {
    String key = "rt:" + token;
    HashOperations<String, Object, Object> h = redis.opsForHash();
    String status = (String) h.get(key, "status");

    if (status == null) {                 // 레코드 없음: 만료됐거나 애초에 없거나, 이미 무효화됨
        return new RefreshResult.Unknown();
    }
    if ("rotated".equals(status)) {       // 이미 한 번 쓴 토큰이 또 왔다 = 재사용 신호
        revokeFamily((String) h.get(key, "familyId"));
        return new RefreshResult.ReuseDetected();
    }
    // status == "active" → 정상 회전
    h.put(key, "status", "rotated");                            // 옛 토큰을 죽이고
    String userId   = (String) h.get(key, "userId");
    String familyId = (String) h.get(key, "familyId");
    return new RefreshResult.Rotated(userId, issue(userId, familyId));  // 새 토큰을 낸다
}

private void revokeFamily(String familyId) {
    Set<String> tokens = redis.opsForSet().members("family:" + familyId);
    if (tokens == null || tokens.isEmpty()) return;

    List<String> keys = new ArrayList<>();
    for (String t : tokens) keys.add("rt:" + t);   // 사슬의 모든 토큰 레코드
    keys.add("family:" + familyId);                // 패밀리 자체까지
    redis.delete(keys);                            // 통째로 삭제 = 무효화
}
```

세 갈래를 다시 짚으면:

- **`active`** — 정상 흐름. 옛 토큰을 `rotated`로 죽이고 같은 패밀리에 새 토큰을 발급한다. 사용자는 최신 표로 갈아탄다.
- **`rotated`** — 이미 소모된 표가 다시 왔다. 정상 사용자는 죽은 표를 다시 꺼낼 일이 없으니 도난 신호로 본다. `revokeFamily`로 그 로그인에서 나온 사슬을 통째로 지운다.
- **레코드 없음(`null`)** — 만료됐거나 애초에 없던 토큰, 또는 방금 재사용 감지로 지워진 패밀리의 토큰. 전부 거부.

> [[refresh-token-rotation]]에선 재사용 시 status를 `revoked`로 바꾼다고 했는데, 여기선 아예 삭제한다. 지우든 `revoked`로 표시하든 결과는 같다 — 그 토큰들은 이후 전부 거부된다. 지우는 쪽이 코드가 단순하고, 만료돼 사라진 토큰을 되살리는 유령 레코드가 안 남는다. 그래서 status는 `active`/`rotated` 둘이면 충분해진다.

컨트롤러는 이 결과를 HTTP로 옮기기만 한다.

```java
@PostMapping("/refresh")
public ResponseEntity<?> refresh(@CookieValue("refresh_token") String token,
                                 HttpServletResponse res) {
    RefreshResult result = refresh.rotate(token);

    if (result instanceof RefreshResult.Rotated r) {
        setRefreshCookie(res, r.next().token());               // 새 refresh 쿠키로 교체
        return ResponseEntity.ok(Map.of("accessToken", jwt.issue(r.userId())));
    }
    clearRefreshCookie(res);                                    // 죽은 쿠키는 지운다
    return ResponseEntity.status(401).body(Map.of("error", "refresh rejected"));
}
```

재사용이든 만료든 클라이언트가 받는 건 똑같은 401이다. 그다음은 재로그인뿐이다.

### 주의: `active → rotated`는 원자적이어야 한다

위 `rotate`에는 [[refresh-token-rotation]]이 열어둔 "동시 요청" 함정이 숨어 있다. 같은 `active` 토큰으로 두 요청이 거의 동시에 오면, 둘 다 `status`를 `active`로 **읽고** 둘 다 회전에 성공한다. 그러면 멀쩡한 사용자가 새 토큰 두 개를 받고, 그중 하나는 곧 `rotated`가 되어 다음 요청 때 재사용으로 오인될 수 있다.

읽기와 쓰기를 하나로 묶으면 막힌다. Redis Lua는 한 스크립트가 통째로 원자적으로 도니, "active면 rotated로 바꾸고 아니면 손대지 마"를 한 번에 처리한다.

```java
private static final RedisScript<Long> ROTATE_IF_ACTIVE = RedisScript.of("""
        if redis.call('HGET', KEYS[1], 'status') == 'active' then
          redis.call('HSET', KEYS[1], 'status', 'rotated')
          return 1
        end
        return 0
        """, Long.class);

// rotate() 안에서 h.get + h.put 대신:
Long won = redis.execute(ROTATE_IF_ACTIVE, List.of(key));
if (won == 1) {
    // 이 요청이 회전 권리를 얻었다 → 새 토큰 발급
} else {
    // active가 아니었다. 상태를 다시 읽어 'rotated'면 재사용 → revokeFamily, 아니면 Unknown
}
```

동시 요청 둘 중 하나만 `1`을 받고, 나머지는 `0`을 받아 회전하지 못한다. 클라이언트 쪽에서도 이 상황을 애초에 안 만드는 방어를 하는데(같은 순간 refresh를 한 번만 보내기), 그건 [[token-auth-client-react]]의 single-flight에서 다룬다. **서버는 경합이 와도 안 깨지게, 클라이언트는 경합을 안 만들게** — 양쪽에서 같은 문제를 막는다.

## 6. 로그아웃: 패밀리 무효화 (+ 선택적 즉시 차단)

로그아웃의 본체는 refresh 사슬을 끊는 것이다. 그래야 더는 새 access token이 안 나온다.

```java
@PostMapping("/logout")
public ResponseEntity<?> logout(@CookieValue(value = "refresh_token", required = false) String token,
                                @RequestHeader(value = "Authorization", required = false) String auth,
                                HttpServletResponse res) {
    if (token != null) refresh.revokeFamilyOf(token);   // 이 로그인 사슬 전체 차단

    // (선택) 지금 들고 있는 access token까지 즉시 죽이려면 jti를 denylist에 올린다.
    if (auth != null && auth.startsWith("Bearer ")) {
        Claims c = jwt.parse(auth.substring(7));
        long remaining = c.getExpiration().toInstant().getEpochSecond()
                       - Instant.now().getEpochSecond();
        if (remaining > 0) {
            redis.opsForValue().set("denylist:" + c.getId(), "1",
                    Duration.ofSeconds(remaining));     // TTL = 그 토큰의 남은 수명
        }
    }
    clearRefreshCookie(res);
    return ResponseEntity.ok().build();
}
```

denylist 항목의 TTL을 access token의 **남은 수명과 똑같이** 건 게 [[session-store]]의 그 트릭이다. 그 시각이 지나면 토큰은 어차피 스스로 만료돼 무효이니, 목록에 더 남겨둘 이유가 없다. 그래서 별도 청소 없이 자동으로 사라진다.

`revokeFamilyOf`는 5절의 `revokeFamily`를 토큰으로 시작하게 감싼 것이다.

```java
public void revokeFamilyOf(String token) {
    String familyId = (String) redis.opsForHash().get("rt:" + token, "familyId");
    if (familyId != null) revokeFamily(familyId);
}
```

## 7. rotation과 denylist는 다른 장치다

이 서버 코드에 둘이 다 나오지만, 위치도 목적도 다르다. [[refresh-token-rotation]]에서 개념으로 갈라놓은 것이 코드에선 이렇게 갈린다.

| | rotation | denylist |
|---|---|---|
| 사는 곳 | `RefreshTokenService.rotate` (`/auth/refresh`) | `JwtAuthFilter` + `/auth/logout` |
| 다루는 토큰 | refresh token (불투명) | access token (JWT), `jti`로 지목 |
| 목적 | 재사용을 **탐지**한다 | 만료 전에 **즉시 차단**한다 |
| Redis 상태 | `rt:` 해시 + `family:` 셋 | `denylist:` 키 (TTL=남은 수명) |

rotation은 "죽은 표가 돌아왔나"를 보는 탐지 장치고, denylist는 "이 access token은 지금부터 막아라"는 차단 목록이다. rotation의 패밀리 전체 무효화와 denylist는 별개다. 로그아웃 때 둘을 함께 쓰는 것뿐이다.

## 시나리오로 재생

`t1`, `t2`… 는 refresh token, `A1`, `A2`… 는 access token, `F`는 패밀리다. 그림으로는 [[refresh-token-rotation]]의 재사용 시나리오와 같다.

1. `POST /auth/login` → `A1` 발급, `t1` 발급(`rt:t1` = active, family `F`). 쿠키에 `t1`.
2. `GET /me` (`Bearer A1`) → 200.
3. 15분 뒤 `A1` 만료. `POST /auth/refresh`(쿠키 `t1`) → `t1` active→rotated, `A2`·`t2`(active) 발급.
4. 또 15분 뒤 `POST /auth/refresh`(쿠키 `t2`) → `t2`→rotated, `A3`·`t3` 발급.
5. **공격자가 예전에 훔쳐 둔 `t1`으로** `POST /auth/refresh` → `rt:t1`.status가 `rotated` → **재사용 감지** → `revokeFamily(F)`: `rt:t1`·`rt:t2`·`rt:t3`·`family:F` 삭제. 401.
6. 진짜 사용자가 `t3`로 갱신 시도 → `rt:t3`가 이미 없음 → 401. 재로그인해야 한다. 도둑도 여기서 끝.

한 명이 재로그인하는 불편으로 도난 전체를 끊는 그 맞바꿈이, 5번의 `revokeFamily` 한 줄에 들어 있다.

## 반대편

클라이언트는 이 서버를 어떻게 부르나 — access token을 어디에 두고, 401을 어떻게 조용히 갱신으로 넘기고, 5번 같은 오탐을 애초에 어떻게 피하나 — 는 [[token-auth-client-react]]에서 잇는다.

back: [[refresh-token-rotation]] · [[session-and-jwt]] · [[session-store]]
