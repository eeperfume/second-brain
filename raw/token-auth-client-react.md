---
aliases: [코드로 보는 토큰 인증 클라이언트, React 토큰 처리, Next 토큰 인증, refresh 래퍼, single-flight refresh, 토큰 인증 클라이언트 예제]
---

# 코드로 보는 토큰 인증 — 클라이언트 (React / Next.js / TS)

[[token-auth-server-spring]]에서 서버가 access token(본문)과 refresh token(httpOnly 쿠키)을 어떻게 내주고 회전시키는지 봤다. 여기서는 그걸 받아 쓰는 쪽이다. 브라우저가 access token을 어디에 들고, 만료됐을 때 어떻게 사용자 모르게 갱신하고, 서버의 재사용 감지에 스스로 걸려드는 사고를 어떻게 피하는지를 코드로 본다. 개념 뿌리는 [[session-and-jwt]]와 [[refresh-token-rotation]]이다.

HTTP 클라이언트는 axios 같은 라이브러리 없이 브라우저·Node에 내장된 `fetch`만 쓴다. 의존성을 하나 줄이면 그만큼 공급망으로 딸려 들어오는 보안 표면도 준다. 대신 axios가 알아서 해 주던 것(요청마다 헤더 붙이기, 401 가로채 재시도하기)을 우리가 얇은 래퍼로 직접 만들어야 하는데, 그 래퍼가 이 노트의 절반이다.

## 뭘 만드나

- access token을 **메모리**에 두는 저장소
- 요청에 `Authorization`을 자동으로 붙이고 refresh 쿠키를 함께 보내는 `fetch` 래퍼
- 401을 만나면 **조용히 `/auth/refresh`를 부르고 원 요청을 재시도**하는 래퍼
- 동시에 여러 요청이 401을 받아도 refresh는 **한 번만** 나가게 하는 single-flight
- 새로고침으로 메모리가 날아간 뒤 access token을 되찾는 부트스트랩

## 1. access token은 메모리, refresh token은 httpOnly 쿠키

저장 위치가 이 설계의 뼈대다([[session-and-jwt]]의 결론).

- **access token → 메모리(자바스크립트 변수)**. localStorage에 두지 않는다. XSS 스크립트가 읽어 갈 수 있기 때문이다. 메모리에 두면 새로고침 때 사라지지만, 그건 refresh로 되찾으면 된다(5절).
- **refresh token → httpOnly 쿠키**. 서버가 심고 JS는 아예 못 만진다. 그래서 우리 코드 어디에도 refresh token을 다루는 줄이 없다. 요청 갈 때 브라우저가 알아서 실어 보낼 뿐이다.

```ts
// auth-store.ts — access token은 여기, 오직 메모리에만 산다.
let accessToken: string | null = null;

export const getAccessToken = () => accessToken;
export const setAccessToken = (t: string | null) => { accessToken = t; };
```

전역 상태 라이브러리(zustand 등)의 persist를 쓰고 싶어도 access token은 빼야 한다. persist는 대개 localStorage로 내려가서 같은 위험을 지기 때문이다.

## 2. `fetch` 래퍼: Authorization 자동 첨부

```ts
// api.ts
import { getAccessToken, setAccessToken } from './auth-store';

const BASE_URL = '/api';   // Spring 서버 (프록시나 BFF 뒤. 7절 참고)

// 나가는 요청마다 메모리의 access token을 헤더에 붙이고,
// credentials로 refresh 쿠키를 함께 실어 보낸다.
function fetchWithAuth(path: string, init: RequestInit = {}): Promise<Response> {
  const token = getAccessToken();
  const headers = new Headers(init.headers);
  if (token) headers.set('Authorization', `Bearer ${token}`);
  return fetch(BASE_URL + path, { ...init, headers, credentials: 'include' });
}
```

`credentials: 'include'`가 refresh token 쿠키를 실어 보내는 스위치다. 서버 쿠키가 `SameSite=Strict`에 `path=/auth`이므로, 실제로 이 쿠키가 붙는 건 `/auth/refresh` 같은 요청뿐이다.

## 3. 401 → 조용히 refresh → 재시도

access token은 15분이면 죽는다. 죽은 채로 요청하면 서버가 401을 준다([[token-auth-server-spring]]의 `JwtAuthFilter`). 사용자에게 "다시 로그인하세요"를 띄우는 대신, `fetchWithAuth`를 한 겹 더 감싼 `apiFetch`가 뒤에서 갱신하고 원래 요청을 다시 보낸다. 앱 코드는 `fetch` 대신 이 `apiFetch`를 부른다.

```ts
// 401이면 한 번 refresh하고 원 요청을 그대로 재시도한다.
export async function apiFetch(path: string, init: RequestInit = {}): Promise<Response> {
  const res = await fetchWithAuth(path, init);
  if (res.status !== 401) return res;

  try {
    await refreshAccessToken();          // 4절. 새 access token을 메모리에 채운다
  } catch {
    setAccessToken(null);
    redirectToLogin();                   // refresh도 실패 = 패밀리가 죽었거나 만료
    return res;
  }
  return fetchWithAuth(path, init);      // 새 토큰으로 딱 한 번 재시도
}
```

재시도는 딱 한 번뿐이다. 갱신 뒤 다시 보낸 요청이 또 401이어도 `apiFetch`는 그 응답을 그대로 돌려줄 뿐 다시 refresh하지 않는다. 래퍼 구조 자체가 무한 루프를 막아서, axios 버전에서 쓰던 `_retried` 같은 표시가 필요 없다.

## 4. 핵심: single-flight refresh

여기가 [[token-auth-server-spring]]의 재사용 감지와 맞물리는 지점이다. 화면에 요청 세 개가 거의 동시에 나갔고 access token이 막 만료됐다고 하자. 셋 다 401을 받는다. 3절의 래퍼가 각자 `refreshAccessToken`을 부르면 **`/auth/refresh`가 세 번** 나간다.

문제는 이거다. 첫 refresh가 쿠키의 `t1`을 `rotated`로 만들고 `t2`를 심는다. 그런데 브라우저 쿠키는 아직 `t1`인 채로 두 번째·세 번째 refresh가 나간다 — 서버는 **이미 rotated된 `t1`이 또 왔다**고 보고 재사용으로 판정해 패밀리를 통째로 무효화한다. 도둑이 아니라 우리 프런트가 자기 발등을 찍는 것이다.

막는 법은 refresh를 **딱 한 번만** 실제로 보내고, 나머지는 그 하나의 결과를 함께 기다리게 하는 것이다. 진행 중인 refresh Promise 하나를 공유하면 된다.

```ts
let refreshing: Promise<string> | null = null;

export function refreshAccessToken(): Promise<string> {
  // 이미 refresh가 진행 중이면 새로 쏘지 않고 그 Promise를 그대로 돌려준다.
  if (!refreshing) {
    // apiFetch가 아니라 fetch를 직접 쓴다 — 여기서 또 401 처리에 들어가면 재귀가 된다.
    refreshing = fetch(`${BASE_URL}/auth/refresh`, { method: 'POST', credentials: 'include' })
      .then(async (res) => {
        // fetch는 axios와 달리 4xx·5xx에도 예외를 안 던진다. 직접 확인해 실패로 바꾼다.
        if (!res.ok) throw new Error('refresh rejected');
        const { accessToken } = await res.json();
        setAccessToken(accessToken);
        return accessToken as string;
      })
      .finally(() => { refreshing = null; });   // 끝나면 다음 refresh를 다시 허용한다
  }
  return refreshing;
}
```

이제 세 요청이 동시에 `refreshAccessToken()`을 불러도, 실제 `/auth/refresh`는 한 번만 나간다. 나머지 둘은 같은 `refreshing` Promise를 기다렸다가 같은 새 토큰을 받아 각자 재시도한다. 서버로 가는 refresh가 하나뿐이니 `t1`이 두 번 제출될 일이 없고, 재사용 오탐도 안 생긴다.

> [[refresh-token-rotation]]의 "동시 요청" 문제를 양쪽에서 막는 그림이 이걸로 완성된다. **클라이언트는 single-flight로 경합을 애초에 안 만들고**, [[token-auth-server-spring]]의 Lua CAS는 그래도 경합이 들어왔을 때 서버가 안 깨지게 한다. 클라이언트 방어는 오탐을 줄이고, 서버 방어는 최후의 보루다. 탭을 여러 개 띄우면 브라우저 컨텍스트가 갈라져 single-flight가 안 통하므로, 서버 쪽 방어가 여전히 필요하다.

## 5. 새로고침 후 부트스트랩

access token을 메모리에 두는 대가는 새로고침 때 사라진다는 것이다. 하지만 refresh token 쿠키는 살아 있으니, 앱이 뜰 때 한 번 `/auth/refresh`를 불러 access token을 되찾으면 된다.

```tsx
// AuthProvider.tsx (Next.js App Router, 클라이언트 컴포넌트)
'use client';
import { useEffect, useState } from 'react';
import { refreshAccessToken } from './api';

export function AuthProvider({ children }: { children: React.ReactNode }) {
  const [ready, setReady] = useState(false);

  useEffect(() => {
    // 새로고침으로 메모리의 access token은 날아갔다.
    // 쿠키의 refresh token으로 조용히 되찾는다. 실패하면(비로그인) 그냥 진행.
    refreshAccessToken().catch(() => {}).finally(() => setReady(true));
  }, []);

  if (!ready) return null;   // 첫 refresh가 끝날 때까지 잠깐 대기(또는 스피너)
  return <>{children}</>;
}
```

이 한 번의 부트스트랩 refresh가 "새로고침해도 로그인이 유지되는" 경험을 만든다. 서버 입장에선 이것도 정상 회전이라, `t1`→`t2`로 한 칸 돌아갈 뿐이다.

## 6. 로그인과 로그아웃

```ts
export async function login(email: string, password: string) {
  const res = await fetch(`${BASE_URL}/auth/login`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    credentials: 'include',
    body: JSON.stringify({ email, password }),
  });
  if (!res.ok) throw new Error('login failed');
  const { accessToken } = await res.json();
  setAccessToken(accessToken);   // refresh_token은 서버가 httpOnly 쿠키로 심는다
}

export async function logout() {
  await apiFetch('/auth/logout', { method: 'POST' });   // 서버가 패밀리 무효화 + 쿠키 삭제
  setAccessToken(null);                                 // 메모리의 access token도 버린다
}
```

로그인 응답에서 우리가 직접 만지는 건 access token뿐이다. refresh token은 코드에 등장하지 않는다 — 서버가 쿠키로 심고, 브라우저가 자동으로 나르고, 서버가 로그아웃 때 지운다. JS는 그 존재만 알 뿐 값은 끝까지 모른다. 이게 httpOnly의 요점이다.

## 7. cross-site 쿠키 피하기: same-origin으로 묶기

브라우저가 다른 출처의 Spring 서버(`api.myapp.com`)를 직접 부르면, refresh 쿠키가 cross-site가 되어 `SameSite=None; Secure`가 강제되고 CORS 자격증명 설정도 얽힌다([[cookie-and-session]]의 `SameSite` 이야기). 해법의 방향은 하나다 — 브라우저가 보기에 프런트와 API를 **같은 출처**로 묶어 쿠키를 same-site로 남기는 것.

가장 흔하고 단순한 건 **리버스 프록시로 경로를 분기**하는 것이다. Nginx(또는 ALB·CloudFront·API 게이트웨이)를 앞에 두고 한 도메인으로 받아, 프런트와 `/api/*`를 뒤쪽 서버로 나눠 보낸다. 브라우저는 `app.myapp.com` 한 곳만 보므로 쿠키가 same-site다. 애플리케이션 코드가 필요 없는 인프라 레벨 라우팅이라 기본값처럼 쓴다.

```nginx
server {
  server_name app.myapp.com;

  location /api/ {
    proxy_pass http://spring-internal/;   # /api/* → Spring
  }
  location / {
    proxy_pass http://next-internal/;     # 나머지 → Next(프런트)
  }
}
```

Next 생태계에서 흔한 또 다른 갈래가 **BFF(Backend for Frontend)**다. Next의 Route Handler를 앞에 두고, 브라우저는 same-origin인 Next만 부르고, Next 서버가 쿠키를 들고 Spring에 프록시한다. 리버스 프록시가 단순 경로 분기라면, BFF는 그 자리에서 서버가 쿠키·시크릿을 직접 쥐고 여러 API를 합치거나 응답을 프런트에 맞게 가공까지 하는 게 값하는 지점이다. 반대로 아무 가공 없이 그대로 넘기기만 한다면 그건 "Next로 구현한 리버스 프록시"에 가깝고, 그럴 바엔 Nginx가 더 가볍다.

```ts
// app/api/auth/refresh/route.ts — Next를 same-origin BFF로
import { cookies } from 'next/headers';
import { NextResponse } from 'next/server';

export async function POST() {
  const jar = await cookies();
  const refresh = jar.get('refresh_token')?.value;

  const res = await fetch('http://spring-internal/auth/refresh', {
    method: 'POST',
    headers: { cookie: `refresh_token=${refresh ?? ''}` },
  });

  const out = NextResponse.json(await res.json(), { status: res.status });
  const rotated = res.headers.get('set-cookie');
  if (rotated) out.headers.set('set-cookie', rotated);   // 회전된 새 쿠키를 브라우저로 전달
  return out;
}
```

이러면 브라우저↔Next는 same-origin이라 쿠키가 `SameSite=Strict`로 깔끔히 붙고, Spring은 내부망에만 노출된다. 규모가 작거나 프런트를 따로 호스팅하면 2절처럼 브라우저가 Spring을 직접 부르고 CORS·`SameSite=None`을 감수해도 된다. 세 갈래 중 무엇을 고르든 클라이언트 코드(2~6절)는 그대로다 — 요청이 향하는 출처만 바뀔 뿐이다.

> 페이지 보호(로그인 안 한 사용자 막기)를 Next `middleware`로 하려면 주의가 필요하다. access token은 메모리에 있어 서버 미들웨어가 못 본다. 그래서 미들웨어는 refresh 쿠키의 존재 정도만 보고 대략 거르고, 정확한 인증 판단은 실제 API 호출의 401로 처리하는 편이 맞다. 쿠키가 있다고 유효한 것도 아니니(회전으로 죽었을 수 있다), 미들웨어의 판단은 어디까지나 1차 거름망이다.

## 반대편

이 클라이언트가 부르는 서버 — access·refresh 발급, 회전, 재사용 감지, denylist — 는 [[token-auth-server-spring]]에 있다.

back: [[refresh-token-rotation]] · [[session-and-jwt]] · [[token-auth-server-spring]]
