---
tags: [쿠버네티스, Helm, ArgoCD, kind, 포트포워딩, Secret]
updated: 2026-11-24
---

# Helm으로 kind에 Argo CD 설치하기

Helm(쿠버네티스용 패키지 관리자)으로 Argo 차트 저장소를 등록하고, 로컬 kind 클러스터에 Argo CD를 올려 웹 UI 로그인까지 확인하는 흐름이다. 각 명령이 실제로 하는 일과 출력 읽는 법을 함께 정리한다.

## 전체 순서
1. 클러스터 켜기: `docker start kind-control-plane` → `kubectl get nodes`가 `Ready`
2. 저장소 등록: `helm repo add argo https://argoproj.github.io/argo-helm` → `helm repo update`
3. 설치: `helm install argocd argo/argo-cd -n argocd --create-namespace`
4. 파드 확인: `kubectl get pods -n argocd -w`
5. 초기 비밀번호: `kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo`
6. UI 접속: `kubectl port-forward svc/argocd-server -n argocd 8080:443` → 브라우저 `https://localhost:8080`(인증서 경고는 "고급 → 계속"), 아이디 `admin`
- 정리: `helm uninstall argocd -n argocd`(Argo CD만 삭제), `docker stop kind-control-plane`(클러스터 정지, 설치한 것은 컨테이너 안에 남음)

## 1. kind 클러스터 켜기
- **kind** = Kubernetes IN Docker. Docker 컨테이너 하나를 노드 한 대처럼 써서 API 서버·스케줄러·etcd까지 그 안에 넣는다. 그래서 `docker ps -a`에 `kind-control-plane` 컨테이너가 보인다.
- 이미 만들어 둔 클러스터는 `kind create cluster`가 아니라 **`docker start`**로 깨운다. 새로 만들면 기존 내용이 사라지지만, 다시 켜면 그대로 남는다. 클러스터가 아예 없을 때만 `kind create cluster`.
- `kubectl get nodes`: `STATUS Ready`면 파드를 받을 준비 완료(`NotReady`는 부팅 중이거나 네트워크 설정이 덜 된 것, 수십 초 대기). `AGE`는 클러스터 생성 후 경과 시간. kubectl이 어느 클러스터를 보는지는 `~/.kube/config`.

## 2. Helm 저장소
- 비유: Helm=npm, **차트**=패키지, 차트 저장소=레지스트리. 차트는 Deployment·Service·ConfigMap·Secret·RBAC 등 YAML 묶음이다.
- `helm repo add <별명> <URL>`: 저장소를 별명으로 등록(이후 `argo/argo-cd`처럼 `별명/차트`로 호출). 정보는 로컬(`~/.config/helm/repositories.yaml`)에만 저장되고 **클러스터엔 변화 없음**. `already exists with the same configuration, skipping`은 오류가 아니라 이미 등록돼 있다는 뜻.
- `helm repo update`: 저장소의 최신 차트 목록(index.yaml)을 받아 캐시. 안 하면 옛 목록으로 옛 버전이 설치될 수 있다.
- 이미 설치돼 있는지는 `helm list -n argocd`로 확인.

## 3. helm install 해부
- `argocd`(첫 번째 인자) = **릴리스 이름**. 같은 차트를 이름만 바꿔 여러 번 설치할 수 있고, 삭제·업그레이드 때 이 이름으로 부른다.
- `argo/argo-cd` = 설치할 차트, `-n argocd` = 네임스페이스(클러스터 안의 폴더 같은 분리 단위), `--create-namespace` = 없으면 만들기(없이 쓰면 네임스페이스 없음 오류).
- 동작: 차트 다운로드 → `values.yaml` 기본값을 템플릿에 채워 YAML 완성 → API 서버에 일괄 전송(Deployment/StatefulSet, Service, Secret, ConfigMap, **CRD**(`Application` 등 쿠버네티스에 원래 없는 리소스 종류), RBAC). 릴리스 기록은 클러스터 안 Secret으로 남아서 나중에 빠짐없이 삭제 가능.
- 출력: `STATUS: deployed`는 **리소스를 보냈다**는 뜻이지 파드가 다 떴다는 뜻이 아님(→ 4단계 확인). `REVISION`은 `helm upgrade`마다 증가하고 `helm rollback`으로 되돌릴 수 있다. `NOTES:`는 차트 작성자의 사용 안내(port-forward·비밀번호 확인법).

## 4. 파드 읽는 법
- `-n argocd` 없으면 `default` 네임스페이스를 봐서 아무것도 안 나옴. `-w`(watch)는 상태가 바뀔 때마다 새 줄을 찍어 Ctrl+C로 직접 끝내야 한다.
- `READY 0/1`인데 `Running`일 수 있다: 프로세스는 떴지만 readiness 검사 통과 전. `PodInitializing`=init 컨테이너 실행 중. `Completed`=일회성 작업이 정상 종료(`0/1`이어도 정상). `RESTARTS`가 계속 오르면 문제.
- 구성 요소:
  - `argocd-server`: 웹 UI·API
  - `repo-server`: Git에서 받아 YAML 생성
  - `application-controller`: Git 상태와 클러스터 실제 상태를 비교·동기화(핵심)
  - `applicationset-controller`: Application 대량 생성
  - `dex-server`: 외부 로그인(SSO)
  - `notifications-controller`: Slack 등 알림
  - `redis`: 캐시
  - `redis-secret-init`: 설치 때 redis 비밀번호를 만드는 일회성 Job

## 5. 초기 admin 비밀번호
- Argo CD가 설치 시 랜덤 비밀번호를 `argocd-initial-admin-secret` Secret에 저장. 명령 분해: `-o jsonpath="{.data.password}"`로 값 칸만 추출 → `base64 -d`로 디코드 → `; echo`로 줄바꿈(없으면 프롬프트가 붙어 읽기 불편).
- 쿠버네티스 Secret은 값을 **base64로 저장**한다(이진 데이터도 담으려는 인코딩). **암호화가 아니라 누구나 풀 수 있는 변환**이므로 Secret 읽기 권한 통제가 중요.
- 안내문이 이 Secret 삭제를 권하는 이유: 남아 있으면 읽을 수 있는 사람은 누구나 admin으로 로그인 가능. 운영에서는 로그인 후 비밀번호 변경 + Secret 삭제, 연습용 로컬 클러스터는 두어도 무방.

## 6. 포트 포워딩
- Argo CD 서버는 클러스터 안에만 열려 있어서 내 PC 8080 포트를 `svc/argocd-server`의 443으로 임시 연결한다. 명령이 계속 떠 있어야 접속되므로 **그 터미널은 켜 둔다**.
- `Handling connection for 8080`은 브라우저가 화면을 불러올 때마다 찍히는 정상 로그. Ctrl+C로 끄면 접속만 끊기고 Argo CD는 클러스터에서 계속 돈다.
- 대안(차트 NOTES): `server.ingress.enabled`로 Ingress를 켜고 SSL passthrough 또는 `configs.params."server.insecure"`로 TLS를 Ingress에서 종료.

## 관련
- [[파드-개념과-kind-실험-결과]] — 같은 kind 클러스터에서의 파드 동작 실험
- [[Deployment-롤아웃과-Service-타입-개념-정리]]
- [[Docker-Desktop-쿠버네티스-이미지-손상과-기본-Docker-kind-전환]] — 이 kind 클러스터를 만든 배경

## 출처
- Claude Code 세션 자동 캡처 (2026-11-24, scratch-2026-11-23-407843). 설치 안내문 NOTES 내용은 실제 `helm install` 출력 기준.
- 비밀번호 값은 기록하지 않음.
