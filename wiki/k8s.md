---
type: summary
title: 쿠버네티스 기초
aliases: [k8s, 쿠버네티스, 쿠버네티스 기초]
description: 컨테이너가 많아질 때의 관리 문제를 "원하는 상태를 선언하면 제어 루프가 현실을 거기 맞춘다"로 푸는 흐름 — 물리서버→VM→컨테이너, Pod/ReplicaSet/Deployment, Service·Ingress, ConfigMap/Secret, 그리고 control plane(두뇌)까지
tags: [k8s, kubernetes, container, orchestration, devops, infra]
date: 2026-07-23
---

# 쿠버네티스 기초

> 한 줄 요약: 컨테이너가 많아지면 사람 손으로 관리가 안 된다. k8s는 "원하는 상태를 선언하면 제어 루프가 현실을 거기 맞춘다"로 그 문제를 푼다. 이 하나의 발상에서 모든 부품이 나온다.

쿠버네티스를 기초부터 따라간 노트 묶음의 요약이다. 여섯 개의 노트로 이어진다. [[container-and-isolation]] → [[k8s-orchestration]] → [[k8s-workloads]] → [[k8s-service]] → [[k8s-ingress-and-config]] → [[k8s-control-plane]] 순서로, "왜 컨테이너인가"에서 시작해 "그걸 누가 다 관리하나"를 거쳐 "그 마법을 부리는 두뇌"까지 내려간다.

## 컨테이너로 오는 길

물리 서버는 앱 하나당 서버 한 대라 자원이 놀고 이식이 안 됐다. VM은 하이퍼바이저로 서버를 쪼개 격리를 얻었지만, 각 VM이 게스트 OS를 통째로 짊어져 무거웠다. 컨테이너는 발상을 바꿔, OS를 복제하지 않고 호스트 커널을 공유하면서 프로세스만 격리한다. 그래서 가볍고 어디서든 똑같이 도는 대신, 커널을 공유하니 격리 강도는 VM보다 약하다. VM과 컨테이너를 가르는 칼금은 하나다. 커널을 각자 갖느냐, 공유하느냐. → [[container-and-isolation]]

## 컨테이너가 많아지면 — 오케스트레이션과 선언형

컨테이너가 수백 개, 서버가 여러 대가 되면 배치·복구·스케일·무중단 배포·서비스 찾기를 사람 손으로 못 한다. 이걸 대신하는 게 오케스트레이터이고 그 표준이 k8s다. k8s는 이 문제들을 하나의 모양으로 푼다. 사람은 "원하는 상태"만 선언하고(선언형), k8s는 관찰→비교→조치→반복이라는 제어 루프(control loop / reconciliation)로 현실을 그 상태에 맞춘다. 명령을 내리는 게 아니라 상태를 적어두면, k8s가 24시간 그 상태를 유지한다. → [[k8s-orchestration]]

## 무엇을 띄우나 — Pod · ReplicaSet · Deployment

k8s는 컨테이너를 직접 다루지 않고 Pod로 감싼다. Pod는 소모품이라 죽으면 새로 뜨고 IP가 바뀐다. ReplicaSet은 "같은 Pod를 N개 유지"로 그 제어 루프를 실제로 돌리는 실체(self-healing)이고, Deployment는 ReplicaSet들의 개수를 조절해 무중단 배포와 rollback을 맡는다. 버전마다 ReplicaSet이 하나씩 생겨 스냅샷으로 남고, Deployment는 그 사이를 오가는 지휘자다. → [[k8s-workloads]]

## 어떻게 찾아가나 — Service

Pod의 IP가 계속 바뀌고 여러 개로 복제돼 있어도, Service가 고정된 이름·IP(대표번호)를 주고 살아있는 Pod로 로드밸런싱한다. Service는 특정 Pod를 지목하지 않고 라벨(selector)로 고른다. Deployment는 라벨을 붙이는 쪽, Service는 읽는 쪽이고, 둘은 서로를 모른 채 라벨이라는 약속 위에서만 만난다. 이 느슨한 결합 덕에 밑에서 Pod가 마음껏 죽고 살아도 연결이 안 끊긴다. → [[k8s-service]]

## 바깥에서 들어오기와 설정 — Ingress · ConfigMap · Secret

외부로 여는 서비스가 여럿이면 LoadBalancer는 하나당 돈이 붙고 주소(host/path)를 못 본다. Ingress는 입구 하나에서 주소를 읽어 여러 Service로 라우팅한다. 여기서 규칙(Ingress 오브젝트)과 실행자(Ingress Controller)가 나뉘는데, 이게 k8s를 관통하는 "오브젝트 vs 컨트롤러" 패턴이다. 한편 설정은 이미지에 박지 않고 ConfigMap·Secret으로 바깥에서 주입해, "이미지는 한 번 빌드, 어디서든 똑같이"를 지킨다. 다만 Secret은 이름과 달리 기본이 base64 인코딩(암호화 아님)이라, 진짜 안전은 encryption-at-rest와 RBAC를 얹어야 나온다. 이 "base64는 인코딩일 뿐"은 [[session-and-jwt]]의 JWT에서 이미 만난 개념이다. → [[k8s-ingress-and-config]]

## 두뇌 — Control Plane

클러스터는 결정을 내리는 Control Plane(두뇌)과 Pod를 실행하는 Worker(일꾼)로 나뉜다. 두뇌 안에는 유일한 입구인 API 서버, 모든 상태를 기억하는 etcd, 배치를 정하는 스케줄러, 제어 루프들이 사는 컨트롤러 매니저가 있고, 일꾼에는 kubelet과 container runtime이 있다. `kubectl apply` 한 번이 API 서버→etcd→컨트롤러들→스케줄러→kubelet으로 이어지며 원하는 상태를 현실로 만든다. 핵심은 어떤 부품도 서로 직접 명령하지 않고 API 서버(공유 상태)만 보고 각자 반응한다는 것이다. 죽었다 살아나도 상태만 다시 읽으면 되니(level-triggered) 놓친 명령이 없고, 그래서 튼튼하다. → [[k8s-control-plane]]

## 요약

k8s의 모든 부품은 "원하는 상태(오브젝트=데이터)"와 "그걸 현실로 맞추는 컨트롤러(제어 루프)"라는 한 패턴의 반복이다. 이 구조가 자동 복구·스케일·무중단 배포를 주는 대신, 배우고 운영하기 어려운 복잡성을 함께 준다. 그래서 k8s는 "규모가 만드는 복잡성"을 "도구가 만드는 복잡성"과 맞바꾼 것이고, 관리할 컨테이너가 많으면 남는 장사, 적으면 손해다. "우리에게 k8s가 정말 필요한가"는 항상 먼저 물어야 하는, 정답 없는 선택이다.

## 출처

- [[container-and-isolation]] · [[k8s-orchestration]] · [[k8s-workloads]] · [[k8s-service]] · [[k8s-ingress-and-config]] · [[k8s-control-plane]] (이 저장소에서 대화로 학습하며 기초부터 쓴 개념 노트)
