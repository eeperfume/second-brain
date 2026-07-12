---
aliases: [코드로 보는 RS256과 JWKS, RS256 예제 코드, JWKS 예제 코드, JwtDecoder 예제, 비대칭 JWT 코드, OAuth 선수지식 RS256]
---

# 코드로 보는 RS256과 JWKS (OAuth 선수지식)

개념은 [[jwt-signing-and-keys]](대칭 HS256 vs 비대칭 RS256)와 [[jwt-jwks]](공개키를 나눠 주는 창구)에서 다뤘다. 여기서는 그걸 코드로 엮는다. [[token-auth-server-spring]]은 발급도 검증도 한 서버라 HS256으로 충분했지만, **검증이 다른 서비스나 제3자로 퍼지는 순간** RS256과 JWKS로 넘어간다. 그 지점이 곧 OAuth2 Resource Server와 OIDC가 사는 곳이라, 이 노트는 OAuth를 파기 전 미리 손에 익히는 프리뷰다.

## 왜 여기서 RS256인가

HS256은 하나의 비밀키로 서명도 검증도 한다. 그래서 검증하려는 서버는 모두 그 비밀키를 가져야 하고, 그 키를 가진 순간 **위조(발급)도 할 수 있다**([[jwt-signing-and-keys]]). 검증만 맡기고 싶은 다른 서비스에까지 발급 권한을 쥐여 주는 셈이라 위험하다.

RS256은 이걸 쪼갠다. 발급자만 **개인키**로 서명하고, 검증자에게는 **공개키**만 준다. 공개키로는 검증만 되고 위조는 안 되니, 검증자가 여럿이거나 남이어도 안전하다. 문제는 하나 남는다 — 그 공개키를 검증자들에게 어떻게 나눠 주나. 그 표준 창구가 JWKS다([[jwt-jwks]]).

정리하면 이 노트에는 세 역할이 나온다. **발급자**(개인키로 서명 + JWKS로 공개키 게시), **JWKS 엔드포인트**(공개키를 담은 JSON), **검증자**(JWKS를 받아 스스로 검증). 발급자와 검증자를 일부러 다른 서비스로 둔다.

## 0. 의존성

```groovy
dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-web'
    // 발급자: 키 생성·서명·JWKS 조립
    implementation 'com.nimbusds:nimbus-jose-jwt:9.40'
    // 검증자: JWKS로 JWT를 검증하는 Spring Security 디코더 (= OAuth2 Resource Server가 쓰는 그것)
    implementation 'org.springframework.boot:spring-boot-starter-oauth2-resource-server'
}
```

## 1. 발급자: RSA 키페어와 RS256 서명

먼저 키페어를 만든다. `kid`(key id)를 박아 두는 게 중요하다. 검증자가 "이 토큰은 어느 공개키로 검증해야 하나"를 이 `kid`로 찾기 때문이다([[jwt-jwks]]).

```java
@Service
public class RsaJwtService {

    private final RSAKey rsaKey;          // 개인키 + 공개키 한 쌍
    private final RSASSASigner signer;

    public RsaJwtService() throws JOSEException {
        // 예시라 시작할 때 생성한다. 실무에선 한 번 만들어 secret 저장소(키스토어·KMS)에
        // 보관하고 재사용한다. 매번 새로 만들면 재시작 때마다 옛 토큰이 다 깨진다.
        this.rsaKey = new RSAKeyGenerator(2048)
                .keyID("2025-01")                 // kid
                .keyUse(KeyUse.SIGNATURE)
                .generate();
        this.signer = new RSASSASigner(rsaKey);
    }

    public String issue(String userId) throws JOSEException {
        Instant now = Instant.now();
        SignedJWT jwt = new SignedJWT(
                new JWSHeader.Builder(JWSAlgorithm.RS256)
                        .keyID(rsaKey.getKeyID())          // 헤더에 kid를 실어 보낸다
                        .build(),
                new JWTClaimsSet.Builder()
                        .subject(userId)
                        .issuer("https://auth.myapp.com")  // iss: 누가 발급했나
                        .audience("api.myapp.com")         // aud: 누구더러 쓰라고
                        .issueTime(Date.from(now))
                        .expirationTime(Date.from(now.plusSeconds(900)))
                        .jwtID(UUID.randomUUID().toString())
                        .build());
        jwt.sign(signer);                                   // 개인키로 서명
        return jwt.serialize();
    }

    RSAKey publicJwk() {
        return rsaKey.toPublicJWK();                        // 개인키를 떼고 공개 부분만
    }
}
```

HS256 버전([[token-auth-server-spring]]의 `JwtService`)과 달라진 곳은 딱 둘이다. 서명 알고리즘이 RS256이고, 서명에 쓰는 게 공유 비밀키가 아니라 발급자만 가진 개인키다. `iss`·`aud`도 넣었는데, 검증자가 "내가 신뢰하는 발급자가 맞나, 내게 온 토큰이 맞나"를 확인하는 값이라 OAuth/OIDC에서 특히 중요해진다.

## 2. 발급자: JWKS 엔드포인트

공개키를 검증자들이 받아 갈 수 있게 URL 하나로 연다. 관례상 주소는 `/.well-known/jwks.json`이다.

```java
@RestController
public class JwksController {

    private final RsaJwtService jwt;

    public JwksController(RsaJwtService jwt) { this.jwt = jwt; }

    @GetMapping("/.well-known/jwks.json")
    public Map<String, Object> jwks() {
        // 공개키만 담은 JSON. { "keys": [ { kty, kid, use, alg, n, e } ] }
        return new JWKSet(jwt.publicJwk()).toJSONObject();
    }
}
```

`toPublicJWK()`가 개인키를 떼어 냈으니 여기 실리는 건 공개 부분(`n`, `e` 등)뿐이다([[jwt-jwks]]에서 본 그 JSON 그대로). 숨길 값이 아니라 아무나 받아 가도 된다.

## 3. 검증자(다른 서비스): JWKS로 검증

핵심은 검증자가 **발급자를 매 요청 부르지 않는다**는 것이다. JWKS를 한 번 받아 캐시하고, 그다음부터는 토큰의 `kid`로 맞는 공개키를 골라 로컬에서 검증한다.

### 3a. Spring Security 방식 (실무 기본)

Spring Security의 `NimbusJwtDecoder`가 이 일(받기·캐시·`kid` 매칭·서명 검증)을 통째로 해 준다.

```java
@Bean
JwtDecoder jwtDecoder() {
    return NimbusJwtDecoder
            .withJwkSetUri("https://auth.myapp.com/.well-known/jwks.json")
            .build();
}
```

그리고 보호할 자리에서:

```java
Jwt token = jwtDecoder.decode(bearer);   // 서명·만료 검증. 실패하면 예외
String userId = token.getSubject();
```

사실 이 디코더를 손으로 만들 일도 거의 없다. `application.yml`에 JWKS 주소만 적으면 Spring Security가 알아서 만들고, Bearer 토큰이 붙은 요청을 자동으로 검증한다.

```yaml
spring:
  security:
    oauth2:
      resourceserver:
        jwt:
          jwk-set-uri: https://auth.myapp.com/.well-known/jwks.json
```

**이 설정 한 조각이 곧 OAuth2 Resource Server다.** OAuth를 배우면 "리소스 서버가 access token(JWT)을 검증한다"는 말을 만나는데, 그 검증이 바로 지금 이 `jwk-set-uri`로 도는 것이다. 미리 만들어 본 셈이다.

### 3b. 원리: 라이브러리가 안에서 하는 일

`withJwkSetUri`가 가린 machinery를 한 겹 벗기면 [[jwt-jwks]]의 "검증자는 JWKS를 어떻게 쓰나" 그림이 그대로 코드가 된다. Nimbus 저수준 API로는 이렇다.

```java
// JWKS를 받아 캐시하고, 주기적으로 갱신하는 소스
JWKSource<SecurityContext> keys = JWKSourceBuilder
        .create(new URL("https://auth.myapp.com/.well-known/jwks.json"))
        .build();

DefaultJWTProcessor<SecurityContext> processor = new DefaultJWTProcessor<>();
// 토큰 헤더의 kid로 keys에서 공개키를 고르고, RS256으로 서명을 검증하는 selector
processor.setJWSKeySelector(
        new JWSVerificationKeySelector<>(JWSAlgorithm.RS256, keys));

JWTClaimsSet claims = processor.process(bearer, null);   // kid 매칭 + 서명 검증
```

`JWKSourceBuilder`가 [[jwt-jwks]]의 코드 예시에 나온 JS `jose`의 `createRemoteJWKSet`과 정확히 같은 역할이다 — 한 번 받아 캐시하고, 처음 보는 `kid`를 만나면 다시 받아 온다. 언어만 다를 뿐 기계는 같다.

## 4. kid와 키 교체

`kid`가 공개키를 여러 개 동시에 둘 수 있게 해 주고, 그게 무중단 키 교체를 만든다([[jwt-jwks]]). 교체 중에는 JWKS에 옛 키와 새 키를 함께 싣는다.

```java
@GetMapping("/.well-known/jwks.json")
public Map<String, Object> jwks() {
    // 교체 중: 옛 키(2025-01)와 새 키(2025-07)를 함께 노출한다.
    return new JWKSet(List.of(oldKey.toPublicJWK(), newKey.toPublicJWK()))
            .toJSONObject();
}
```

발급자는 새로 내는 토큰부터 새 `kid`로 서명하고, 옛 키는 옛 토큰이 다 만료될 때까지 JWKS에 남겨 둔다. 검증자는 토큰의 `kid`를 보고 골라 검증하므로, 옛 토큰과 새 토큰이 한동안 함께 유효하다. HS256이었다면 모든 서버에 새 비밀키를 동시에 다시 뿌려야 했을 일이, JWKS 한 곳만 갱신하면 검증자들이 알아서 따라오는 일로 바뀐다.

## 5. 여기서 OAuth/OIDC로

지금 손으로 만든 것 — 개인키로 서명하고, JWKS로 공개키를 게시하고, 검증자가 `kid`로 골라 검증하는 구조 — 가 사실 OIDC 제공자(Keycloak, Auth0, Cognito 등)가 이미 제공하는 바로 그것이다. 그들이 여는 `/.well-known/openid-configuration`을 열어 보면 `jwks_uri` 필드가 있고, 그게 2절에서 만든 JWKS 주소를 가리킨다. OIDC의 ID 토큰은 RS256으로 서명된 JWT이고, 그 검증이 3절과 똑같다.

그럼 OAuth는 이 위에 무엇을 더 얹나. 크게 보면 **토큰을 어떻게 트러스트하나**(지금까지)와 **토큰을 애초에 어떻게 얻나**(다음)의 분업이다. RS256+JWKS는 앞쪽, 곧 받은 토큰을 믿는 방법을 준다. OAuth가 더하는 건 뒤쪽이다 — 사용자를 로그인시켜 동의를 받고(authorization code + PKCE), 권한 범위(scope)를 정하고, `/authorize`·`/token` 엔드포인트로 토큰을 발급받는 흐름. 그 흐름 끝에 나오는 access token·ID 토큰을 검증하는 대목에서 이 노트가 다시 등장한다. 그래서 이걸 먼저 익혀 두면 OAuth의 절반은 이미 아는 채로 시작하는 셈이다.

## 요약

검증이 한 서버를 벗어나면 HS256이 아니라 RS256을 쓴다. 발급자만 개인키로 서명하고, 공개키는 JWKS(`/.well-known/jwks.json`)로 게시하며, 검증자는 그 JSON을 한 번 받아 캐시한 뒤 토큰의 `kid`로 맞는 공개키를 골라 로컬에서 검증한다. Spring에서는 `jwk-set-uri` 한 줄이 이 검증자(=OAuth2 Resource Server)를 만들어 준다. 키 교체도 JWKS 한 곳만 갱신하면 되고, 이 구조 전체가 OIDC 제공자가 이미 노출하는 것과 같아서 OAuth 학습의 든든한 선수지식이 된다.

back: [[jwt-signing-and-keys]] · [[jwt-jwks]] · [[token-auth-server-spring]]
