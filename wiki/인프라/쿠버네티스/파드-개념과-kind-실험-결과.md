---
tags: [쿠버네티스, Pod, kind, 사이드카, init컨테이너, 라이프사이클]
updated: 2026-11-12
---

# 파드 개념과 kind 실험 결과

파드는 네트워크(IP·포트)를 공유하고 필요하면 볼륨을 같이 마운트하는 컨테이너 묶음이며, 쿠버네티스가 스케줄하는 가장 작은 단위다. 로컬 kind 클러스터(노드 1개)에서 직접 돌려 확인한 동작과, 헷갈리기 쉬운 구분(컨테이너 재시작 vs 파드 교체, STATUS vs phase 등)을 정리한다.

## 파드 = 논리 호스트
- 컨테이너들이 **IP 하나·포트 공간 하나**를 공유 → 서로 `localhost`로 통신. 항상 **같은 노드에 같이 스케줄**된다(스케줄러는 파드 통째로 노드를 고름). 요청 자원 합이 들어갈 노드가 없으면 일부만 뜨지 않고 파드 전체가 `Pending`.
- **네트워크 공유는 자동, 스토리지 공유는 자동이 아니다.** 파일시스템은 컨테이너마다 따로이고, `spec.volumes`에 볼륨을 선언한 뒤 컨테이너마다 `volumeMounts`로 붙여야 공유된다(마운트 경로는 컨테이너마다 달라도 됨).
- 실험(`pod-two`: nginx + busybox): 두 컨테이너 IP 동일, busybox에서 `localhost:80`이 nginx 응답, 하지만 busybox에는 `/usr/share/nginx/html`이 없음. `emptyDir` 공유 실험에서는 writer가 5초마다 쓴 `index.html`이 nginx에 바로 반영됨.
- `emptyDir`은 파드가 지워질 때 같이 지워진다. 파드보다 오래 남길 데이터는 PersistentVolume.
- "애플리케이션 하나를 위한" 논리 호스트: 프론트+백+DB를 한 파드에 넣지 않는다(각자 따로 스케일). 웹서버+전용 로그 수집기처럼 떨어지면 의미 없는 것만 묶는다. 복제(수평 확장)는 **컨테이너가 아니라 파드를 늘려서** 한다 — 한 파드에 nginx 3개는 포트 충돌 + 한 노드에 몰림.

## 같은 포트를 쓰는 컨테이너 2개 (실험 `pod-clash`)
- 파드는 **정상 생성**된다(쿠버네티스는 컨테이너가 열 포트를 미리 모름). 먼저 뜬 컨테이너가 80을 잡고, 두 번째는 `bind() to 0.0.0.0:80 failed (98: Address already in use)`로 exit 1 → 그 컨테이너만 재시작 반복(재시작 간격이 점점 늘어남, `CrashLoopBackOff`로 보이는 게 보통).
- 이때 **파드 phase는 `Running`** (하나라도 실행 중이면 Running). 건강 여부는 phase가 아니라 `READY 1/2`와 RESTARTS로 본다.

## 컨테이너 재시작 ≠ 파드 교체
| 한 일 | RESTARTS | IP | UID |
|---|---|---|---|
| 파드 생성 | 0 | 10.244.0.6 | 1f9a… |
| 컨테이너 프로세스 kill (`kill 1`) | 1 | 그대로 | 그대로 |
| 파드 삭제 후 같은 이름으로 재생성 | 0 | 10.244.0.7 | 099d… (다른 파드) |
- 컨테이너가 죽으면 kubelet이 **같은 파드 안에서** 그 컨테이너만 다시 띄운다. 파드 자체를 재시작하는 기능은 없고, "파드 재시작"은 실제로 **삭제 후 새 파드 생성**이다.
- 파드가 노드에 배정되면 이사하지 않고, 실행 끝남·삭제·축출(eviction)·노드 사망 중 하나로 **거기서 끝**난다. 대체 파드는 컨트롤러가 새로 만든다(UID·IP 다름). 직접 만든 파드는 아무도 다시 만들어 주지 않는다.
- 규칙: 파드는 컨트롤러(Deployment 등)에 맡기고, IP를 코드에 박지 말고 Service를 쓴다. (한 파드의 IP는 살아 있는 동안 안 바뀐다. 바뀌는 건 **교체된 새 파드**의 IP.)
- `restartPolicy`(파드 전체에 적용): `Always`(기본, 항상 재시작 → 계속 Running) / `OnFailure`(실패 때만, 성공하면 Succeeded) / `Never`. Deployment는 Always만, Job은 OnFailure·Never만 쓸 수 있다.

## phase 5가지와 kubectl STATUS
- phase: `Pending`(노드 배정 대기·이미지 pull·init 컨테이너 실행 중 포함) / `Running` / `Succeeded` / `Failed` / `Unknown`(노드와 통신 불가).
- `Unknown`은 중간 상태: 노드가 돌아오면 복귀, 끝내 안 돌아오면 `Failed` 처리 후 삭제. (실례: 클러스터가 꺼져 있던 동안 `Unknown`이던 파드가 켜자 `Running`으로 복귀, AGE 유지·RESTARTS만 +1.)
- `kubectl get pods`의 **STATUS는 phase가 아니라 kubectl이 phase+컨테이너 상태를 합친 표시용 문구**: `ContainerCreating`·`ImagePullBackOff`(→ Pending), `CrashLoopBackOff`(→ 보통 Running), `Terminating`(삭제 전 phase 유지), `Completed`(→ Succeeded). 진짜 phase는 `kubectl describe pod`의 `Status:`.

## init 컨테이너 · 사이드카 · 임시 컨테이너
- **init 컨테이너**(`spec.initContainers`): 파드가 **시작되는 동안** 앱 컨테이너보다 먼저, 순서대로 하나씩 실행돼 성공해야 다음/앱이 시작. 실행 중엔 `STATUS Init:0/2`, phase는 `Pending`. 실패하면 앱은 시작되지 않고 재시도.
- init 컨테이너에 `restartPolicy: Always`를 붙이면 **사이드카**(v1.33 Stable): 순서대로 시작하되 "끝나야"가 아니라 "시작만 하면" 다음으로 넘어가고, 앱이 도는 동안 계속 돌며 죽으면 재시작, 종료 때는 앱이 먼저 끝나고 사이드카가 마지막. `containers`에 같이 넣으면 기동 순서 보장이 없지만 initContainers에 두면 **앱보다 먼저 뜨는 것이 보장**. 이 `restartPolicy`는 컨테이너 단위이고 `spec.restartPolicy`는 파드 단위.
- 사이드카 용도: 로그 파일을 읽어 전송, git에서 최신 파일을 받아 웹서버 폴더 갱신, 통신 프록시(암호화·기록). 앱 코드를 안 건드리고 기능을 붙이고 여러 앱에 재사용하기 위함.
- **임시 컨테이너**(ephemeral container): 셸 없는 distroless 이미지 등을 디버깅하려고 실행 중인 파드에 끼워 넣음(`kubectl debug -it 파드 --image=busybox --target=앱`). 포트 불가·자원 비보장·재시작 없음·한 번 넣으면 변경/제거 불가·`kubectl edit`로는 못 넣음. 실행 중 파드에 컨테이너를 추가하는 **유일한 허용 방법**. (파드 특징의 "일회성 ephemeral"과는 다른 뜻.)

## 주요 필드 메모
- 이름: 필수 규칙은 DNS 서브도메인(소문자·숫자·`-`·`.`, 253자), **권장은 DNS 레이블(`.` 불가, 63자)** — 파드 이름이 컨테이너 hostname으로 쓰이기 때문.
- 레이블은 `키: 값` 태그. 이름·IP는 파드가 바뀔 때마다 달라지므로 Service·ReplicaSet/Deployment는 파드를 **레이블 셀렉터**로 찾는다.
- `resources.requests`는 스케줄링 기준(자리 부족 → Pending), `limits`는 상한(CPU는 throttling, 메모리 초과는 `OOMKilled`). `250m` = CPU 0.25, `64Mi` = 64MiB.
- `ports.containerPort`는 안내용 표시 — 안 적어도 접속된다.
- `spec.os.name`은 스케줄링에 영향이 없다. 맞는 OS 노드로 보내려면 `nodeSelector: kubernetes.io/os: ...`를 함께 쓴다(안 맞으면 kubelet이 실행 거부).
- **실행 중인 파드는 거의 못 고친다**: 변경 가능 = 이미지, `activeDeadlineSeconds`, `terminationGracePeriodSeconds`, `tolerations`(추가만), `schedulingGates`; 리소스는 resize 서브리소스. 환경변수·포트·명령은 파드를 새로 만들어야 한다.
- **파드 템플릿**(`spec.template`): 컨트롤러가 파드를 찍어 내는 틀(파드의 `metadata`·`spec`, `apiVersion`·`kind` 제외). 템플릿을 고쳐도 **이미 떠 있는 파드는 안 바뀐다.** 새 파드로 언제·어떻게 교체할지는 리소스마다 다르다(ReplicaSet은 기존 파드를 안 바꿈, OnDelete는 수동 삭제). 템플릿 레이블은 컨트롤러 `selector`와 일치해야 한다.

## ReplicaSet과 Deployment의 역할 분담
- ReplicaSet: 레이블 셀렉터로 센 파드 수를 `replicas`로 유지(모자라면 템플릿으로 생성, 넘치면 삭제). **버전 교체는 못 한다.**
- Deployment: 버전마다 ReplicaSet을 하나씩 만들어 업데이트·롤백을 관리. "선언적"(원하는 상태만 적음), "스테이트리스"(파드가 중요한 데이터를 안 가짐).

## 관련
- [[워크로드-리소스-정리와-공식문서-대조-정정]] — 같은 Notion 정리를 공식 문서와 대조해 고친 내용
- [[Docker-Desktop-쿠버네티스-이미지-손상과-기본-Docker-kind-전환]] — 실습에 쓴 kind 환경

## 출처
- Claude Code 세션 자동 캡처 (/home/yunho) — 2026-11-12 세션
