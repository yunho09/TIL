---
tags: [쿠버네티스, Deployment, ReplicaSet, Service, ClusterIP, Ingress, 롤아웃]
updated: 2026-11-13
---

# Deployment 롤아웃과 Service 타입 개념 정리

Deployment는 원하는 상태(이미지·replicas)만 적으면 ReplicaSet을 통해 파드 수와 버전을 맞춰 주는 리소스이고, Service는 계속 바뀌는 파드들 앞에 고정 주소(ClusterIP·DNS 이름)를 두고 트래픽을 나눠 주는 추상화다. Notion k8s 노트를 공부하며 헷갈렸던 지점(롤아웃 범위, 롤백 원리, 타입별 접근 범위, 헤드리스, Ingress와의 역할 분담)을 정리한다. 파드 쪽은 [[파드-개념과-kind-실험-결과]].

## Deployment → ReplicaSet → Pod
- **Deployment**는 버전 관리(업데이트·롤백), **ReplicaSet**은 개수 유지. Deployment는 파드를 직접 만들지 않고 **ReplicaSet의 replicas 숫자만** 올리고 내린다. 파드 생성·삭제는 각 ReplicaSet이 한다.
- 버전(파드 템플릿)이 바뀔 때마다 그 버전용 ReplicaSet이 새로 생긴다. Deployment가 만든 ReplicaSet은 직접 고치지 않는다(Deployment가 자기 기준으로 되돌림).
- YAML의 `selector.matchLabels`와 `template.metadata.labels`는 **반드시 일치**해야 한다(안 맞으면 생성 거부 — 안 맞으면 "내 파드 0개"로 보고 계속 새로 만들게 되므로). `metadata.labels`(Deployment 자신의 태그)는 파드 선택에 안 쓰인다.
- 다른 Deployment와 selector 레이블이 겹치면 서로 상대 파드를 자기 것으로 세어 충돌한다. 데이터·고정 이름이 필요한 앱은 StatefulSet.

## 롤아웃 · 업데이트 방식
- **롤아웃** = 새 버전으로 바꿔 가는 과정 전체. **파드 템플릿이 바뀔 때만** 시작된다(이미지 변경이 대표적). `replicas`만 바꾸는 스케일링은 롤아웃이 아니다(새 ReplicaSet·버전 안 생김).
- `RollingUpdate`(기본): 조금씩 교체 → 안 끊기지만 잠깐 두 버전이 같이 돎. `maxSurge`(초과 허용 수)·`maxUnavailable`(빠져도 되는 수) 기본 둘 다 25%, **둘 다 0 불가**. 파드 4개면 3~5개 사이 유지.
- `Recreate`: 옛 파드 전부 종료 후 새로 띄움 → 잠깐 끊기지만 두 버전이 동시에 돌지 않음(DB 스키마 변경 등).

## 롤백
- 옛 ReplicaSet을 **지우지 않고 0개로 남겨 둔다**. 롤백 = 옛 ReplicaSet 개수를 올리고 현재 것을 내리는, 방향만 반대인 롤아웃(무중단, YAML 재작성 불필요).
- `revisionHistoryLimit` 기본 10(0이면 옛 ReplicaSet을 안 남겨 롤백 불가). **자동 롤백은 없다** — 계속 실패해도 "실패" 표시만 하고 사람이 되돌려야 한다.
- 일시 중지(`pause`): 템플릿 수정이 기록만 되고 롤아웃은 안 일어나며, `resume` 때 **한 번에 롤아웃**(이미지·env·메모리를 따로 고쳐 3번 롤아웃되는 낭비 방지). 중지 중에도 기존 파드와 자가 치유는 계속되고, **중지 중엔 롤백 불가**(먼저 재개). 재개를 잊으면 배포가 안 된다.

## 스케일링
- `replicas` 숫자 변경 → Deployment가 ReplicaSet 개수 변경 → 같은 템플릿으로 파드 증감. **HPA**는 부하(예: CPU 80% 유지, min 2·max 10)에 따라 `replicas`를 대신 바꿔 주므로 HPA를 붙였으면 `replicas`를 직접 바꾸지 않는다. 수평 확장=파드 수, 수직 확장=파드 하나의 CPU·메모리.

## Service
- **왜**: 파드 IP는 교체 때마다 바뀌므로, Service가 안 바뀌는 **가상 IP(ClusterIP)와 DNS 이름**을 준다. 부르는 쪽은 백엔드 파드 수·위치·IP를 몰라도 되는 디커플링.
- **ClusterIP는 가상 IP**: 그 IP를 가진 파드·서버는 없고, 각 노드의 규칙이 요청을 가로채 실제 파드 중 하나로 전달한다. Service를 지우기 전까지 안 바뀜(파드 IP와 대비). 클러스터 안에서만 접속 가능.
- **셀렉터 → EndpointSlice**: 레이블이 맞는 **요청받을 준비가 된 파드만**(시작 중·종료 중 제외) 목록으로 유지하고 거기에 로드밸런싱. 새 파드는 같은 레이블만 있으면 자동 편입.
- 포트: `port`(Service가 받음)·`targetPort`(파드 앱이 실제 듣는 포트, 생략 시 port와 동일)·NodePort일 때 `nodePort`(밖에서 노드로 들어오는 포트). `protocol` 기본 TCP, `type` 기본 ClusterIP.
- 이름(`metadata.name`)이 곧 접속 주소. 같은 네임스페이스면 `backend`, 다른 네임스페이스면 `backend.shop`.

### 타입 = "누가 접속할 수 있나"
| 타입 | 접근 | 비고 |
|---|---|---|
| ClusterIP (기본) | 클러스터 안만 | 파드끼리 통신 |
| NodePort | 밖에서 `노드IP:30000~32767` | 모든 노드에 같은 포트, ClusterIP 위에 얹힘. 낯선 포트·노드 IP 의존 → 테스트용 |
| LoadBalancer | 밖에서 공인 주소 | 클라우드 LB 필요(kind 같은 로컬에선 `pending`), LB마다 비용, NodePort·ClusterIP 포함 |
| ExternalName | 안에서 부르면 바깥 도메인으로 | 파드·셀렉터 없이 **CNAME 별명**만. 외부 DB 주소 변경 시 Service만 수정 |
- Service는 "밖에서 온 요청만" 다루는 게 아니라, 안·밖 어디서 온 요청이든 받아 파드들로 나눠 준다. 안에서 쓰는 경우(프론트→백엔드→DB)가 더 많고, 타입은 **바깥 출입문을 달지 여부**.
- **헤드리스**(`clusterIP: None`): 가상 IP·트래픽 분배 없이 이름 조회 시 **파드 IP 목록을 그대로** 반환 → 어느 파드로 갈지는 호출하는 쪽이 고른다. StatefulSet과 함께 `db-0`처럼 고정 이름으로 특정 파드(예: 쓰기 담당)를 지목할 때 쓴다.
  - 셀렉터는 **그대로 있다**: 레이블로 파드를 찾고 준비된 파드 명단을 갱신하는 일은 일반 Service와 같고, 없는 건 **대표 IP로 대신 나눠 보내는 일**뿐. 헤드리스 Service는 요청을 넘기지 않고 "명단 관리 + DNS 이름 붙이기"만 한다(요청은 Service를 거치지 않고 파드 IP로 직행).
  - DNS 기록 비교(파드 `db-0`=10.244.0.5, `db-1`=10.244.0.6, Service IP 10.96.0.15 가정): 일반 `db` → 10.96.0.15 한 줄 / 헤드리스 `db` → 10.244.0.5, 10.244.0.6 전부, StatefulSet이면 추가로 `db-0.db` → 10.244.0.5, `db-1.db` → 10.244.0.6(형식 `파드이름.서비스이름`).
  - 파드 IP는 파드가 **살아 있는 동안엔 안 바뀌고** 교체될 때만 바뀐다. 코드에는 이름만 쓰고 요청 때마다 DNS로 다시 물으면 지금 IP를 얻으므로 문제없다(IP를 코드에 적는 게 문제).
- **DNS**: 클러스터 전용 DNS(Domain Name System)가 하나 있고 Service를 만들 때마다 기록이 추가된다. 이름:IP는 1:1이 아니다 — 이름 하나에 IP 여러 개(헤드리스 `db`, 인터넷의 `google.com`), IP 하나에 이름 여러 개(`db`와 `db-0.db`가 같은 IP를 가리킴). 짧은 이름 `db`의 전체 이름은 `db.default.svc.cluster.local`(같은 네임스페이스면 앞부분만 써도 됨). 여러 IP가 오면 호출하는 쪽이 하나를 고른다(보통 첫 번째, DNS가 순서를 섞기도 함).
- **누가 무엇을 갖나**: 파드는 레이블과 자기 IP만 갖고 Service의 존재도 모른다(파드에 설치·설정할 것 없음). Service가 이름·고정 IP·셀렉터 조건·파드 명단을 갖고, DNS는 이름→IP 기록만 갖는다. "찾는 일"이 두 번: Service가 레이블로 파드를 찾고, 호출자가 이름으로 Service를 찾는다.
- **Service 개수·셀렉터**: 보통 **앱 하나당 Service 하나**(frontend·backend·db), Service 하나 뒤에 같은 앱의 파드 여러 개. 셀렉터에 레이블을 여러 개 쓰면 **전부 가진 파드만**(AND) 선택 — 예: `app: backend`만이면 v1·v2 모두, `app: backend` + `version: v2`면 v2만.
- 포트를 여러 개 열면 포트마다 `name`을 붙인다. Service 이름은 DNS 이름이 되므로 소문자·숫자·`-`만(63자 이하).
- **Service는 요청 내용을 읽지 않는다.** 호스트(`admin.shop.com`)·경로(`/api`)별 분기는 **Ingress / Gateway API**가 하고(Gateway API는 Ingress의 후속, 기능 더 많음), 뒤에서 각 Service가 파드로 나눈다. Ingress 앞에 LB 하나만 두면 Service마다 LB를 만드는 비용도 줄인다.

## 관련
- [[파드-개념과-kind-실험-결과]] — 파드·레이블·템플릿 기초
- [[워크로드-리소스-정리와-공식문서-대조-정정]] — 같은 Notion 정리의 공식 문서 대조

## 출처
- Claude Code 세션 자동 캡처 (/home/yunho) — 2026-11-12~13 세션 (설명 내용은 Claude의 개념 설명이며 공식 문서 대조는 위 대조 페이지 참조)
