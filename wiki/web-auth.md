---
type: summary
title: 세션과 JWT 인증
aliases: [웹 인증, 세션·JWT 인증]
description: HTTP 무상태에서 쿠키·세션·JWT가 상태를 어디에 두느냐로 갈리는 흐름과 서명·키·JWKS, refresh token rotation, 그리고 Spring+Redis·React/Next.js 구현과 RS256·JWKS 코드(OAuth 선수지식)까지
tags: [auth, session, jwt, cookie, oauth, security]
date: 2026-07-10
---

# 세션과 JWT 인증

> 한 줄: HTTP는 상태가 없어 요청 사이에 사용자를 기억하지 못한다. 그 문제를 푸는 방식이 쿠키·세션·JWT이고, 이 셋이 갈리는 지점은 "상태를 어디에 두느냐"이다.

웹 인증을 기초부터 따라간 노트 묶음의 요약이다. 여섯 개의 노트로 이어진다. [[cookie-and-session]] → [[session-store]] → [[session-and-jwt]]까지 온 다음 둘로 갈린다. 하나는 [[jwt-signing-and-keys]] → [[jwt-jwks]](서명과 키)로, 다른 하나는 [[refresh-token-rotation]](refresh token 관리)으로 이어진다. 여기에 개념을 실제 코드로 엮은 구현 세 편이 나란히 붙는다.

## HTTP는 상태가 없다

요청 하나하나가 독립적이라, 서버는 방금 온 요청과 다음 요청이 같은 사람인지 기본적으로 모른다. 그래서 클라이언트는 요청마다 자신이 조금 전의 그 사람임을 증명할 무언가를 실어 보내야 한다. 쿠키·세션·JWT는 모두 이 한 문제의 답이다.

## 쿠키와 세션은 상태를 서버에 둔다

쿠키는 서버가 준 쪽지를 브라우저가 저장했다가 요청마다 자동으로 붙여 보내는 운반 수단이다. 중요한 데이터를 쿠키에 통째로 넣으면 위조되고 무거우므로, 데이터는 서버가 갖고 클라이언트에는 그것을 가리키는 번호표(세션 ID)만 준다. 이것이 세션이다. 쿠키와 세션은 대립이 아니라, 세션이 쿠키를 운반 수단으로 쓰는 관계다. → [[cookie-and-session]]

세션 상태를 서버 어디에 둘지 정할 때 부딪히는 트레이드오프는 속도·생존·공유 세 가지이다. 인메모리는 빠르지만 재시작에 날아가고 공유가 안 돼, 여러 서버가 공유하는 Redis로 간다. 디스크 저장(RDB·AOF)으로 생존을 조절하고 TTL로 만료를 맡긴다. → [[session-store]]

## 세션과 JWT는 상태를 어디에 두는가

세션은 상태를 서버가 쥐고 클라이언트엔 번호표만 준다(상태 있음). JWT는 정보를 서명된 토큰 자체에 담아 서버는 위조만 검증한다(상태 없음). 이 하나의 차이에서 모든 게 나온다. 세션은 서버에서 지우는 즉시 죽으니 무효화가 쉽지만 상태를 들고 있어 확장(scale-out)이 번거롭다. JWT는 저장소 없이 확장하기 쉽지만 한 번 내준 토큰을 무효화하기 어렵다. 그래서 토큰 인증을 채택한 실무에서는 보편적으로 둘을 섞어 절충한다. 수명이 짧은 access token(JWT)으로 확장의 이득은 지키고, 서버가 추적하는 refresh token으로 무효화 수단을 되찾는다. → [[session-and-jwt]]

## JWT의 서명과 키

상태를 서버에 두지 않으니, JWT를 믿을 근거는 서명뿐이다. HS256(대칭)은 하나의 비밀키로 서명·검증해 검증자 모두가 그 키를 나눠 가져야 하고, 그 키로 위조도 가능하다. RS256(비대칭)은 개인키로 서명하고 공개키로 검증해, 검증을 여러 서비스나 외부에 맡길 수 있다. "구글 계정으로 로그인"(OpenID Connect, OIDC)처럼 제3자가 검증하는 경우 비대칭이 사실상 필수다. → [[jwt-signing-and-keys]]

RS256으로 검증을 밖에 맡기려면 공개키부터 검증자들에게 나눠 줘야 한다. 그 표준 창구가 JWKS다. 발급자가 공개키를 `/.well-known/jwks.json`에 JSON으로 공개하고, 검증자는 한 번 받아 캐시한 뒤 토큰의 `kid`로 맞는 키를 골라 로컬에서 검증한다. 키 교체도 JWKS 한 곳만 갱신하면 된다. → [[jwt-jwks]]

## refresh token rotation은 탈취를 알아챈다

앞서 토큰 인증을 채택한 실무에서는 보편적으로 수명이 짧은 access token(JWT)과 서버가 추적하는 refresh token을 섞어 절충한다고 했다. 그 refresh token은 오래 사는 bearer token이다. 토큰을 가진 사람(bearer)에게 그대로 접근 권한을 부여하는 인증 방식이라, 탈취당한 것인지 정상적인 것인지 토큰만 봐서는 구별할 수 없다. 그래서 한 번 쓰고 버리게 만들어 refresh token을 재사용할 수 없게 한다. 갱신마다 새 refresh token으로 바꾸고 직전 토큰은 무효화하면, 그 무효화된 토큰이 다시 나타나는 것 자체가 도난 신호가 된다. 만약 무효화된 토큰으로 갱신을 시도하는 요청이 오면 도난으로 보고, 그 토큰이 속한 로그인의 refresh token 전체(token family)를 무효화해 진짜 사용자와 탈취자를 함께 끊는다. 이것을 refresh token rotation이라 한다. 이 rotation은 반드시 서버가 상태를 쥐고 있어야 돌아가기 때문에 상태가 없다는 JWT의 장점도 무색해진다. 그래서 JWT의 장점은 살려두기 위해 access token은 JWT 그대로 두고, refresh token은 흔히 세션 ID 같은 불투명 토큰(opaque token)으로 구현한다. → [[refresh-token-rotation]]

## 코드로 구현하기

여기까지의 개념을 실제로 돌아가는 하나의 흐름으로 엮으면, 역할이 서버와 클라이언트로 나뉜다. 서버는 토큰의 생명주기를 관리한다. access token은 HS256 방식으로 서명한 JWT로 발급해 서명만 검증하고, refresh token은 저장소로 널리 쓰이는 Redis에 상태를 두어 앞 섹션의 rotation을 재사용 감지부터 family 무효화까지 그대로 구현한다. 여기에 무효화하기 어려운 access token을 즉시 차단해야 할 때 쓰는 denylist를 더한다. 차단할 access token의 ID를 남은 수명만큼 Redis에 올려 두고, 요청마다 대조해 걸리면 서명이 유효해도 거절하는 장치다. → [[token-auth-server-spring]]

클라이언트는 그 서버를 가져다 쓴다. access token은 메모리에, refresh token은 httpOnly 쿠키에 두고, 401을 만나면 `fetch` 래퍼가 조용히 refresh해서 원 요청을 재시도한다. → [[token-auth-client-react]]

서버와 클라이언트 구현은 [[refresh-token-rotation]]의 "동시 요청" 문제, 곧 갱신이 겹치면 정상 사용자도 재사용으로 몰리는 문제를 함께 막는다. 클라이언트는 refresh를 한 번에 하나만 보내 경합을 애초에 안 만들고(single-flight), 서버는 경합이 들어와도 한 요청만 성공하게 처리해 안 깨진다. 또 코드로 보면 rotation(refresh token 재사용 감지)과 denylist(access token 즉시 차단)가 서로 다른 문제를 해결하기 위한 장치임이 분명해진다. 작동하는 시점도, 관리하는 토큰도 다르다.

이 구현에서는 토큰 발급과 검증을 같은 서버가 하므로 HS256으로 충분했다. 검증이 다른 서비스나 제3자로 퍼지면 앞서 본 RS256+JWKS 구조로 넘어간다. 이 구조는 "구글 계정으로 로그인" 같은 OIDC 제공자가 이미 쓰고 있는 것과 같아서, OAuth를 배우기 전 미리 손에 익히는 선수지식이 된다. → [[jwt-rs256-jwks-spring]]

## 요약

세션이냐 JWT냐, HS256이냐 RS256이냐는 모두 "상태와 신뢰를 어디에 두고 누가 검증하느냐"로 갈린다. 서버가 쥐면 무효화가 쉽고 확장(scale-out)이 번거롭다. 토큰이 지니면 확장이 쉽고 무효화가 어렵다. 이 트레이드오프가 인증 설계의 바탕이다.

## 출처

- [[cookie-and-session]] · [[session-store]] · [[session-and-jwt]] · [[refresh-token-rotation]] · [[jwt-signing-and-keys]] · [[jwt-jwks]] (이 저장소에서 기초부터 쓴 개념 노트)
- [[token-auth-server-spring]] · [[token-auth-client-react]] · [[jwt-rs256-jwks-spring]] (개념을 Spring+Redis / React+Next.js 코드로 엮은 구현 walkthrough. RS256·JWKS 편은 OAuth 선수지식)
