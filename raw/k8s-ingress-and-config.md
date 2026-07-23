---
aliases: [Ingress, Ingress Controller, ConfigMap, Secret, 시크릿, 설정 주입, encryption-at-rest, RBAC]
---

# Ingress와 설정 (ConfigMap · Secret)

## 5. 다섯 번째 부품 Ingress

외부에 열어야 할 서비스가 여러 개일 때, LoadBalancer Service를 하나씩 쓰면 두 가지 문제점이 있다.
- 돈이다. LoadBalancer 하나는 곧 클라우드 로드밸런서 하나라 공개 IP를 하나 받고, 요금도 하나 추가될 때마다 청구된다. 즉 서비스 10개를 열기위해 로드밸런서 10개를 사는 건 돈 낭비이다.
- 멍청하다. 보통 우리가 원하는 건 이런 것이다. mysite.com/api로 오면 백엔드로, mysite.com으로 오면 프론트로, shop.mysite.com으로 오면 쇼핑 서비스로 가길 바란다. 근데 일반 Service는 URL의 경로(/api)나 호스트(shop.mysite.com)를 못 본다. Service는 그냥 "이 라벨을 Pod에게 넘겨"까지만 한다. 즉 주소를 읽고 똑똑하게 나눠보내질 못 한다.

이 두 문제를 어떻게 해결할 수 있었을까? 해법으로 Ingress가 등장했다. Ingress는 하나의 입구에서 주소 보고 알맞은 Service로 라우팅해준다. Ingress를 쓰면 공개 입구는 하나(LB 하나)만 두고, 그 뒤에서 호스트/경로를 보고 내부 Service들(ClusterIP)로 나눠 보낸다.

규칙은 다음과 같이 생겼다.

```yaml
kind: Ingress
rules:
  - host: mysite.com
    http:
      paths:
        - path: /api      # mysite.com/api → backend Service로
          backend: { service: { name: backend, port: 80 } }
        - path: /         # mysite.com/ → frontend Service로
          backend: { service: { name: frontend, port: 80 } }
```

여기서 중요하게 짚고 넘어갈 점이 있다. 바로 그 패턴이다.
- Ingress는 그냥 규칙(원하는 라우팅 상태)을 적은 YAML일 뿐이다. 즉 혼자서는 아무것도 안 한다.
- 이 규칙을 실제로 집행하는 실행자가 따로 있다. 우리는 실행자를 Ingress Controller라고 부르기로 했다. Ingress Controller의 대표적인 예시로 nginx ingress controller가 있다. 얘가 진짜 리버스 프록시로 떠서, Ingress 규칙을 읽고 그대로 트래픽을 나눠 보낸다.

느낌이 오지 않는가? 규칙(선언) 따로하고, 그걸 현실로 만드는 컨트롤러가 따로 있는 것이다. 우리가 지난번에 계속 봤던 그 구조이다. Deployment(규칙)이 있고, 그걸 집행하는 실행자인 컨트롤러가 따로 있는 것이다. Ingress도 마찬가지다. Ingress(규칙)이고 실행자로 Ingress Controller가 있는 것이다. k8s는 어디서 원하는 상태를 적는 오브젝트와 그걸 맞추려고 하는 컨트롤러가 짝을 이룬다. 이제 앞으로는 k8s의 새로운 부품을 보더라도 어떤 게 규칙이고, 어떤 게 집행자인지 알 수 있을 것이다. (이 "오브젝트 vs 컨트롤러" 구분은 [[k8s-control-plane]]에서 더 깊게 다룬다.)

![[k8s-ingress.svg]]

전체 흐름을 보면 다음과 같다. 인터넷으로 시작해서 [Ingress Controller] LB하나로 들어오고, Ingress Controller는 Ingress 규칙을 읽어 host/path로 판단한다. 그 다음 Ingress Controller의 path에 있는 /api, / 같은 게 있다. 여기서 /api는 backend Service가 되고, 이 주소로 오면 backend Pod로 보낸다. /는 frontend Service가 되고, 이 주소로 오면 frontend Pod로 보낸다.

이 때도 트레이드오프가 발생한다. Ingress Controller라는 층을 하나 더 띄우고 관리해야 한다. 즉 공짜가 아니다! 대신 로드밸런서 하나로 서비스 여러 개를 감당할 수 있게돼서 돈을 절약할 수 있고, 호스트/경로 라우팅에 HTTPS 인증서 처리(TLS)까지 한곳에서 해결할 수 있게 된다. 서비스가 딱 하나면 그냥 LoadBalancer Service가 더 단순하고, 여러 개를 HTTP로 열면 Ingress가 이득이다. (참고로 요즘은 Ingress의 후계자 격인 Gateway API로 옮겨가는 흐름인데, 개념은 똑같다.)

## 6. 여섯 번째 부품 ConfigMap / Secret

앱은 환경마다 다른 값이 필요했다. 예를들어 DB 주소, 기능 켜고/끄기. 그리고 DB비밀번호, API 키 같은 민감한 값. 이 값들을 어디서 관리해야 했을까?
- 이미지에 환경변수를 포함한다? 그럼 개발용/운영용이 다를 때마다 이미지를 새로 빌드해야 한다. 우리는 맨 처음에 컨테이너 환경을 이해하는 과정에서 이미지는 한 번 만들어서 어디서는 똑같이 실행하고 싶어했다. 근데 이걸 부정하게된다. 즉 같은 이미지가 환경마다 달라져버리게 된다.
- 비밀번호를 이미지에 포함한다? 이미지를 뜯어보면 누구나 비밀번호를 볼 수 있기 때문에 보안 사고가 발생하게 된다.

그래서 반드시 짚고 넘어가야 할 게 두 가지가 있는데,
첫 번째는 이미지는 변해선 안 된다는 불변을 지켜야하고
두 번째는 설정은 환경마다 갈아끼울 수 있다고 생각해야하는 것이다.
이 둘을 분리해서 설정은 실행할 때 바깥에서 주입해야 한다는 사실을 인지하고 있자.

그렇다면 바깥에서 주입해야 한다고 했는데 어떻게 할 수 있을까? 그 주입 통로는 두 가지 부품으로 해결할 수 있다.
1. ConfigMap
2. Secret

ConfigMap은 안 민감한 설정을 (key-value) 자료구조로 담는다. 예를들어 DB 주소, 로그 레벨, 기능 켜고/끌 수 있는 스위치 같은 것들이다.
Secret은 민감한 값을 담는다. 예를들어 DB 비밀번호, API 키, TLS 인증서 같은 것들이다.

둘 다 Pod에 환경변수로 꽂거나 파일로 마운트해서 넣는다. 예를 들어

```yaml
# ConfigMap 하나를 정의한다
kind: ConfigMap
metadata: { name: app-config }
data:
  LOG_LEVEL: "info"
  DB_HOST: "db.internal"
---
# Pod에서는 그걸 환경변수로 주입한다.
envFrom:
  - configMapRef: { name: app-config } # 이러면 컨테이너 안에서 process.env.LOG_LEVEL로 읽힌다!
```

이러면 같은 이미지더라도 개발 환경에는 LOG_LEVEL=debug라는 ConfigMap과 운영 환경에는 LOG_LEVEL=info라는 ConfigMap과 붙여서 돌릴 수 있게 된다. 이미지가 변하면 안 된다는 불변을 지켜냈고, 맨 처음에 컨테이너 환경을 채택할 때 약속했던 "이미지는 한 번만 빌드하고, 어디서든 똑같이 사용한다"는 것을 지킬 수 있게된다.

근데 반드시 짚고 넘어가야 할 점이 있다. 바로 Secret이라고 해서 암호화는 안 된다는 것이다. 무슨 얘기냐면 k8s의 Secret은 기본적으로 그냥 base64 인코딩만 돼 있다. base64는 암호화가 아니라 그냥 표현 방식이다. 즉 아무나 디코딩 과정을 통해 원래 값으로 돌릴 수가 있다. 즉 이름만 Secret이지 그 자체로 비밀번호를 숨겨주진 않는다.

> 이 "base64는 인코딩일 뿐 암호화가 아니다"는 [[session-and-jwt]]에서 이미 만난 개념이다. 거기서도 JWT의 헤더·내용은 base64로 인코딩만 돼 있어 누구나 풀어 읽을 수 있고, 서명이 지키는 건 비밀이 아니라 위조 불가라고 했다. 같은 원리가 k8s Secret에서 또 나온다.

그럼 Secret과 ConfigMap은 왜 나뉘었을까? 도대체 Secret은 ConfigMap에 비해 무엇이 더 낫길래 나눴을까? 이는 Secret만이 제공하는 세 가지 특성을 알아야한다.
- 저장소(etcd)에 저장될 때 암호화(encryption-at-rest)를 켤 수 있고, ← etcd를 통째로 훔쳐가도 열쇠 없인 못 읽게 막는다
- 누가 읽을 수 있나(RBAC)를 따로 잠글 수 있고, ← 권한 없는 사람/앱의 읽기 자체를 막는다
- 로그·화면에 값이 함부로 안 찍히게 다뤄지기 때문이다.

즉 Secret이라고 해서 암호화가 되진 않지만, Secret이라는 전용 자리 위에 암호화(encryption-at-rest)와 접근 통제(RBAC)를 얹어야 비로소 안전하게 쓸 수 있게 된다. (etcd·encryption-at-rest·RBAC가 실제로 어디서 도는지는 [[k8s-control-plane]]에서 이어진다.)

back: [[k8s-service]]
