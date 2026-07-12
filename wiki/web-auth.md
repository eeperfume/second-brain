---
type: summary
title: 세션과 JWT 인증
aliases: [웹 인증, 세션·JWT 인증]
description: HTTP 무상태에서 쿠키·세션·JWT가 상태를 어디에 두는가로 갈리는 흐름과 서명·키·JWKS, refresh token 회전, 그리고 Spring+Redis·React/Next 구현과 RS256·JWKS 코드(OAuth 선수지식)까지
tags: [auth, session, jwt, cookie, oauth, security]
date: 2026-07-10
---

# 세션과 JWT 인증

> 한 줄: HTTP는 상태가 없어 요청 사이에 사용자를 기억하지 못한다. 그 문제를 푸는 방식이 쿠키·세션·JWT이고, 갈림의 핵심 축은 "상태를 어디에 두는가"다.

웹 인증을 바닥부터 따라간 노트 묶음의 요약이다. 여섯 노트가 이어진다. [[cookie-and-session]] → [[session-store]] → [[session-and-jwt]]까지 온 다음, 한 갈래는 [[jwt-signing-and-keys]] → [[jwt-jwks]](서명과 키)로, 다른 갈래는 [[refresh-token-rotation]](refresh token 관리)으로 뻗는다. 여기에 개념을 실제 코드로 엮은 구현 companion 두 편([[token-auth-server-spring]], [[token-auth-client-react]])이 붙는다.

## 뿌리 문제: HTTP는 상태가 없다

요청 하나하나가 독립적이라, 서버는 방금 온 요청과 다음 요청이 같은 사람인지 기본적으로 모른다. 그래서 요청마다 "나 아까 그 사람"임을 증명할 무언가를 실어 보내야 한다. 쿠키·세션·JWT는 모두 이 한 문제의 답이다.

## 쿠키와 세션: 상태는 서버에

쿠키는 서버가 준 쪽지를 브라우저가 저장했다가 요청마다 자동으로 붙여 보내는 운반 수단이다. 중요한 데이터를 쿠키에 통째로 넣으면 위조되고 무거우므로, 데이터는 서버가 갖고 클라이언트에는 그것을 가리키는 번호표(세션 ID)만 준다. 이것이 세션이다. 쿠키와 세션은 대립이 아니라, 세션이 쿠키를 운반 수단으로 쓰는 관계다. → [[cookie-and-session]]

세션 상태를 서버 어디에 두느냐는 속도·생존·공유 세 축으로 저울질한다. 인메모리는 빠르지만 재시작에 날아가고 공유가 안 돼, 여러 서버가 공유하는 Redis로 간다. RDB·AOF로 생존을 조절하고 TTL로 만료를 맡긴다. → [[session-store]]

## 세션과 JWT: 상태를 어디에 두는가

세션은 상태를 서버가 쥐고 클라이언트엔 번호표만 준다(상태 있음). JWT는 정보를 서명된 토큰 자체에 담아 서버는 위조만 검증한다(상태 없음). 이 하나의 차이에서 모든 게 나온다. 세션은 서버에서 지우면 즉시 무효라 무효화가 쉽지만 상태를 들고 있어 확장이 번거롭다. JWT는 저장소 없이 확장하기 쉽지만 한 번 내준 토큰을 무효화하기 어렵다. 실무는 짧은 access token(JWT)과 서버가 추적하는 refresh token을 섞어 절충한다. → [[session-and-jwt]]

## refresh token 회전: 탈취를 알아채는 법

절충의 축인 refresh token은 오래 사는 bearer token이라, 훔친 것과 진짜를 토큰만 봐서는 구별할 수 없다. 그래서 한 번 쓰고 버리게 만든다. 갱신마다 새 refresh token으로 바꾸고 옛것을 무효화하면, 이미 죽은 토큰이 다시 나타나는 것 자체가 도난 신호가 된다. 감지되면 그 로그인에서 나온 token family 전체를 무효화해 진짜 사용자와 도둑을 함께 끊는다. 이 상태 관리 때문에 refresh token은 흔히 JWT가 아니라 서버가 추적하는 불투명 토큰으로 구현된다. → [[refresh-token-rotation]]

## JWT의 서명과 키

JWT의 신뢰는 서명에서 온다. HS256(대칭)은 하나의 비밀키로 서명·검증해 검증자 모두가 그 키를 나눠 가져야 하고, 그 키로 위조도 가능하다. RS256(비대칭)은 개인키로 서명하고 공개키로 검증해, 검증을 여러 서비스나 외부에 맡길 수 있다. "구글 계정으로 로그인"(OIDC)처럼 제3자가 검증하는 경우 비대칭이 사실상 필수다. → [[jwt-signing-and-keys]]

RS256에서 공개키를 검증자들에게 나눠 주는 표준 창구가 JWKS다. 발급자가 공개키를 `/.well-known/jwks.json`에 JSON으로 공개하고, 검증자는 한 번 받아 캐시한 뒤 토큰의 `kid`로 맞는 키를 골라 로컬에서 검증한다. 키 교체도 JWKS 한 곳만 갱신하면 된다. → [[jwt-jwks]]

## 코드로 구현하기

개념을 하나의 동작하는 흐름으로 엮으면 이렇게 갈린다. 서버(Spring Boot + Redis)는 토큰의 수명주기를 소유한다. access token은 HS256 JWT로 발급해 서명만 검증하고, refresh token은 불투명 문자열로 Redis에 `rt:`·`family:` 상태를 두며 `/auth/refresh`에서 회전시킨다. 이미 쓴(rotated) 토큰이 또 오면 재사용으로 보고 그 패밀리를 통째로 무효화한다. 로그아웃은 패밀리를 끊고, 즉시 차단이 필요하면 access token의 `jti`를 denylist에 남은 수명만큼 올린다. → [[token-auth-server-spring]]

클라이언트(React/Next/TS)는 그 서버를 소비한다. access token은 메모리에, refresh token은 httpOnly 쿠키에 두고, 401을 만나면 `fetch` 래퍼가 조용히 refresh해서 원 요청을 재시도한다(axios 없이 순수 `fetch`로). → [[token-auth-client-react]]

두 편을 관통하는 핵심은 [[refresh-token-rotation]]의 "동시 요청" 문제를 양쪽에서 막는다는 것이다. 클라이언트는 single-flight로 refresh를 한 번만 보내 경합을 애초에 안 만들고, 서버는 Lua compare-and-set으로 경합이 들어와도 안 깨진다. 또 코드로 보면 rotation(refresh token 재사용 탐지)과 denylist(access token 즉시 차단)가 서로 다른 장치임이 분명해진다. 사는 위치도, 다루는 토큰도 다르다.

위 서버는 발급도 검증도 한 서버라 HS256으로 충분했다. 검증이 다른 서비스나 제3자로 퍼지면 RS256+JWKS로 넘어간다. 발급자만 개인키로 서명하고, 공개키는 JWKS(`/.well-known/jwks.json`)로 게시하며, 검증자는 그걸 받아 토큰의 `kid`로 골라 스스로 검증한다. Spring에선 `jwk-set-uri` 한 줄이 이 검증자(=OAuth2 Resource Server)를 만든다. 이 구조가 OIDC 제공자가 이미 노출하는 것과 같아, OAuth를 배우기 전 미리 손에 익히는 선수지식이 된다. → [[jwt-rs256-jwks-spring]]

## 한 축으로

세션이냐 JWT냐, HS256이냐 RS256이냐는 모두 "상태와 신뢰를 어디에 두고 누가 검증하느냐"로 갈린다. 서버가 쥐면 무효화가 쉽고 확장이 번거롭다. 토큰이 지니면 확장이 쉽고 무효화가 어렵다. 이 대칭이 인증 설계의 바탕이다.

## 출처

- [[cookie-and-session]] · [[session-store]] · [[session-and-jwt]] · [[refresh-token-rotation]] · [[jwt-signing-and-keys]] · [[jwt-jwks]] (이 저장소에서 바닥부터 쓴 개념 노트)
- [[token-auth-server-spring]] · [[token-auth-client-react]] · [[jwt-rs256-jwks-spring]] (개념을 Spring+Redis / React+Next 코드로 엮은 구현 walkthrough. RS256·JWKS 편은 OAuth 선수지식)
