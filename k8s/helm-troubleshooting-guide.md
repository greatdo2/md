# Helm 트러블슈팅 가이드 ⎈

## 목차
1. [Helm 기본 진단](#1-helm-기본-진단)
2. [설치(Install) 문제 해결](#2-설치install-문제-해결)
3. [업그레이드(Upgrade) 문제 해결](#3-업그레이드upgrade-문제-해결)
4. [Release 관리 문제](#4-release-관리-문제)
5. [Chart 개발 및 디버깅](#5-chart-개발-및-디버깅)
6. [Values 및 템플릿 문제](#6-values-및-템플릿-문제)
7. [Repository 문제 해결](#7-repository-문제-해결)
8. [실전 예제 시나리오](#8-실전-예제-시나리오)

---

## 1. Helm 기본 진단

### 1.1 Helm 버전 및 환경 확인

```bash
# Helm 버전 확인
helm version

# Helm 환경 정보
helm env

# Kubernetes 연결 확인
helm list
helm list --all-namespaces
```

**예제 출력:**
```
version.BuildInfo{Version:"v3.13.0", GitCommit:"825e86f6a7a38cef1112bfa606e4127a706749b1", GitTreeState:"clean", GoVersion:"go1.20.8"}
```

### 1.2 Release 상태 확인

```bash
# 모든 release 조회
helm list
helm list -A  # 모든 네임스페이스

# 특정 release 상태 확인
helm status <release-name>
helm status <release-name> -n <namespace>

# Release 히스토리 확인
helm history <release-name>
helm history <release-name> -n <namespace>
```

**실제 예제:**
```bash
helm list -n production

# 출력:
NAME            NAMESPACE   REVISION    UPDATED                                 STATUS      CHART           APP VERSION
my-app          production  3           2024-01-28 10:30:00.123456 +0900 KST   deployed    my-app-1.2.3    1.2.3
nginx-ingress   production  5           2024-01-27 15:20:00.123456 +0900 KST   failed      nginx-3.1.0     1.5.1  ⚠️
```

---

## 2. 설치(Install) 문제 해결

### 2.1 설치 실패 진단

#### 문제: Chart를 찾을 수 없음

```bash
# 에러 예시:
# Error: failed to download "myapp" (hint: running `helm repo update` may help)

# 해결 방법:
# 1. Repository 목록 확인
helm repo list

# 2. Repository 업데이트
helm repo update

# 3. Chart 검색
helm search repo myapp

# 4. 특정 버전 검색
helm search repo myapp --versions
```

**실전 예제:**
```bash
# Repository 추가
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update

# Chart 검색 및 확인
helm search repo bitnami/mysql
helm show chart bitnami/mysql
helm show values bitnami/mysql
```

#### 문제: 필수 Values가 누락됨

```bash
# 에러 예시:
# Error: values don't meet the specifications of the schema(s)

# 해결 방법:
# 1. Chart의 values.yaml 확인
helm show values <chart-name>

# 2. 필수 값 확인
helm show values bitnami/mysql | grep -A 5 "## @param"

# 3. values.yaml 파일 생성
cat > my-values.yaml <<EOF
auth:
  rootPassword: "mySecurePassword123"
  database: "myapp"
  username: "myuser"
  password: "myPassword456"
EOF

# 4. values 파일과 함께 설치
helm install my-release bitnami/mysql -f my-values.yaml
```

#### 문제: Namespace가 존재하지 않음

```bash
# 에러 예시:
# Error: create: failed to create: namespaces "myapp" not found

# 해결 방법 1: Namespace 먼저 생성
kubectl create namespace myapp
helm install my-release bitnami/mysql -n myapp

# 해결 방법 2: --create-namespace 옵션 사용
helm install my-release bitnami/mysql -n myapp --create-namespace
```

### 2.2 설치 전 검증 (Dry-run)

```bash
# Dry-run으로 설치 시뮬레이션
helm install my-release bitnami/mysql --dry-run --debug

# 템플릿 렌더링 결과만 확인
helm template my-release bitnami/mysql

# Values와 함께 dry-run
helm install my-release bitnami/mysql -f my-values.yaml --dry-run --debug

# 특정 네임스페이스로 dry-run
helm install my-release bitnami/mysql -n production --dry-run --debug --create-namespace
```

**Dry-run 출력 분석:**
```bash
# 출력에서 확인해야 할 사항:
# 1. 모든 YAML이 올바르게 생성되는가?
# 2. Values가 제대로 적용되었는가?
# 3. 리소스 이름에 충돌이 없는가?
# 4. 에러 메시지가 없는가?

helm install my-app ./my-chart --dry-run --debug 2>&1 | grep -i error
```

### 2.3 설치 시 일반적인 에러

#### 에러: "release already exists"

```bash
# 에러 메시지:
# Error: cannot re-use a name that is still in use

# 확인
helm list -n <namespace>
helm list --all -n <namespace>  # 삭제된 것도 포함

# 해결 방법 1: 다른 이름 사용
helm install my-release-v2 bitnami/mysql

# 해결 방법 2: 기존 release 완전 삭제
helm uninstall my-release -n <namespace>

# 해결 방법 3: 강제 재설치
helm install my-release bitnami/mysql --replace
```

#### 에러: Timeout

```bash
# 에러 메시지:
# Error: timed out waiting for the condition

# 해결 방법 1: Timeout 시간 늘리기
helm install my-release bitnami/mysql --timeout 10m

# 해결 방법 2: --wait 없이 설치
helm install my-release bitnami/mysql --wait=false

# 해결 방법 3: 설치 후 상태 확인
helm install my-release bitnami/mysql --wait=false
kubectl get pods -l app.kubernetes.io/instance=my-release -w
```

**Timeout 원인 분석:**
```bash
# Pod 상태 확인
kubectl get pods -l app.kubernetes.io/instance=my-release

# Pod 이벤트 확인
kubectl describe pod <pod-name>

# 로그 확인
kubectl logs <pod-name>

# 일반적인 원인:
# - 이미지 풀링 실패
# - 리소스 부족
# - PVC 바인딩 실패
# - Liveness/Readiness probe 실패
```

---

## 3. 업그레이드(Upgrade) 문제 해결

### 3.1 업그레이드 전 검증

```bash
# 현재 설치된 Chart 정보 확인
helm get values my-release
helm get manifest my-release

# 새 버전으로 dry-run
helm upgrade my-release bitnami/mysql --version 9.5.0 --dry-run --debug

# 변경사항 비교
helm diff upgrade my-release bitnami/mysql --version 9.5.0  # helm diff 플러그인 필요
```

**helm diff 플러그인 설치:**
```bash
helm plugin install https://github.com/databus23/helm-diff

# 사용 예제
helm diff upgrade my-release bitnami/mysql -f new-values.yaml
```

### 3.2 업그레이드 실패 처리

#### 문제: 업그레이드 중 실패

```bash
# 에러 예시:
# Error: UPGRADE FAILED: another operation (install/upgrade/rollback) is in progress

# 상태 확인
helm status my-release

# Release 상태가 "pending-upgrade"인 경우
helm list -n <namespace> | grep pending

# 해결 방법 1: Rollback
helm rollback my-release

# 해결 방법 2: 특정 revision으로 rollback
helm history my-release
helm rollback my-release 2

# 해결 방법 3: 강제 업그레이드 (주의!)
helm upgrade my-release bitnami/mysql --force
```

**실전 Rollback 예제:**
```bash
# 업그레이드 실패 시나리오
helm upgrade my-app ./my-chart --set image.tag=v2.0.0

# 문제 발생 확인
kubectl get pods -l app=my-app
# NAME           READY   STATUS             RESTARTS   AGE
# my-app-xxxxx   0/1     ImagePullBackOff   0          2m

# 이전 버전으로 즉시 롤백
helm rollback my-app

# 특정 revision으로 롤백
helm history my-app
# REVISION  UPDATED                   STATUS          CHART       DESCRIPTION
# 1         Mon Jan 25 10:00:00 2024  superseded      my-app-1.0  Install complete
# 2         Mon Jan 28 11:00:00 2024  failed          my-app-2.0  Upgrade failed

helm rollback my-app 1
```

### 3.3 업그레이드 중 상태 확인

```bash
# 업그레이드 진행 중 모니터링
helm upgrade my-release bitnami/mysql --wait --timeout 5m

# 다른 터미널에서 실시간 모니터링
watch -n 2 kubectl get pods -l app.kubernetes.io/instance=my-release

# 업그레이드 후 상태 확인
helm status my-release
helm get values my-release
```

---

## 4. Release 관리 문제

### 4.1 Release가 "pending" 상태로 멈춤

```bash
# Release 상태 확인
helm list -a -n <namespace>

# Secret 확인 (Helm 3는 Secret에 release 정보 저장)
kubectl get secrets -n <namespace> -l owner=helm

# 특정 release의 secret 확인
kubectl get secret sh.helm.release.v1.my-release.v1 -n <namespace> -o yaml

# 해결 방법 1: Pending release 삭제
kubectl delete secret -l name=my-release,owner=helm -n <namespace>

# 해결 방법 2: Helm uninstall
helm uninstall my-release -n <namespace>

# 해결 방법 3: 강제 재설치
helm install my-release bitnami/mysql -n <namespace> --replace
```

### 4.2 Release 완전 삭제

```bash
# 일반 삭제
helm uninstall my-release

# 히스토리까지 완전 삭제 (Helm 3는 기본적으로 히스토리 삭제)
helm uninstall my-release --keep-history  # 히스토리 유지하려면

# Namespace와 함께 삭제
helm uninstall my-release -n <namespace>
kubectl delete namespace <namespace>

# 삭제 확인
helm list -a -n <namespace>
kubectl get all -n <namespace>
```

### 4.3 여러 Release 관리

```bash
# 모든 namespace의 release 조회
helm list -A

# Failed 상태의 release 찾기
helm list -A --failed

# 특정 패턴의 release 찾기
helm list -A | grep "my-app"

# 오래된 release 정리
for release in $(helm list -q -n old-namespace); do
  echo "Uninstalling $release"
  helm uninstall $release -n old-namespace
done
```

---

## 5. Chart 개발 및 디버깅

### 5.1 Chart 구조 검증

```bash
# Chart 생성
helm create my-chart

# Chart 구조 확인
tree my-chart/
```

**Chart 디렉토리 구조:**
```
my-chart/
├── Chart.yaml          # Chart 메타데이터
├── values.yaml         # 기본 values
├── charts/             # 의존성 charts
├── templates/          # Kubernetes 템플릿
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── ingress.yaml
│   ├── _helpers.tpl    # 템플릿 헬퍼
│   └── NOTES.txt       # 설치 후 안내문
└── .helmignore         # 무시할 파일
```

```bash
# Chart 유효성 검증
helm lint my-chart/

# 더 엄격한 검증
helm lint my-chart/ --strict

# Chart 패키징
helm package my-chart/
```

**Lint 출력 예제:**
```bash
helm lint ./my-chart

# 성공 출력:
==> Linting ./my-chart
[INFO] Chart.yaml: icon is recommended
1 chart(s) linted, 0 chart(s) failed

# 실패 출력:
==> Linting ./my-chart
[ERROR] templates/deployment.yaml: unable to parse YAML
    error converting YAML to JSON: yaml: line 15: mapping values are not allowed in this context
[ERROR] Chart.yaml: version is required
Error: 1 chart(s) linted, 1 chart(s) failed
```

### 5.2 템플릿 디버깅

```bash
# 템플릿 렌더링 결과 확인
helm template my-release ./my-chart

# 특정 values로 템플릿 렌더링
helm template my-release ./my-chart -f my-values.yaml

# 특정 파일만 렌더링
helm template my-release ./my-chart --show-only templates/deployment.yaml

# 디버그 모드로 자세한 정보 출력
helm template my-release ./my-chart --debug
```

**템플릿 함수 테스트:**
```bash
# _helpers.tpl 테스트
helm template my-release ./my-chart --debug 2>&1 | grep -A 5 "fullname"

# Values 적용 확인
helm template my-release ./my-chart --set image.tag=v2.0.0 | grep "image:"
```

### 5.3 Chart 의존성 관리

```bash
# Chart.yaml에 의존성 정의 예제:
# dependencies:
#   - name: postgresql
#     version: "12.1.0"
#     repository: "https://charts.bitnami.com/bitnami"

# 의존성 다운로드
helm dependency update ./my-chart

# 의존성 목록 확인
helm dependency list ./my-chart

# 의존성 빌드
helm dependency build ./my-chart
```

**의존성 문제 해결:**
```bash
# 에러: dependency update failed

# charts/ 디렉토리 확인
ls -la ./my-chart/charts/

# Lock 파일 확인
cat ./my-chart/Chart.lock

# 의존성 재다운로드
rm -rf ./my-chart/charts/*
rm ./my-chart/Chart.lock
helm dependency update ./my-chart

# 특정 의존성 버전 문제
helm dependency update ./my-chart --verify
```

---

## 6. Values 및 템플릿 문제

### 6.1 Values 덮어쓰기 우선순위

```bash
# Values 우선순위 (높은 순):
# 1. --set 파라미터
# 2. -f values.yaml 파일 (나중에 지정한 것이 우선)
# 3. Chart의 기본 values.yaml

# 예제: 우선순위 테스트
cat > values1.yaml <<EOF
replicaCount: 2
image:
  tag: v1.0.0
EOF

cat > values2.yaml <<EOF
replicaCount: 3
EOF

# 실행
helm template my-release ./my-chart \
  -f values1.yaml \
  -f values2.yaml \
  --set image.tag=v2.0.0 | grep -E "replicas:|image:"

# 결과:
# replicas: 3          # values2.yaml에서
# image: myapp:v2.0.0  # --set에서
```

### 6.2 Values 검증

```bash
# 현재 적용된 values 확인
helm get values my-release

# 기본값 포함하여 모든 values 확인
helm get values my-release --all

# 특정 키만 확인
helm get values my-release -o json | jq '.image'

# Values 구조 확인
helm show values bitnami/mysql | less
```

**Values 검증 스크립트:**
```bash
# values-test.sh
#!/bin/bash

CHART="./my-chart"
VALUES="my-values.yaml"

echo "Testing values..."
helm template test $CHART -f $VALUES --debug > /tmp/rendered.yaml

# 필수 값 검증
if ! grep -q "image: " /tmp/rendered.yaml; then
    echo "ERROR: Image not found!"
    exit 1
fi

if ! grep -q "replicas: " /tmp/rendered.yaml; then
    echo "ERROR: Replicas not found!"
    exit 1
fi

echo "Values validation passed!"
```

### 6.3 템플릿 함수 문제 해결

#### 문제: 템플릿 렌더링 에러

```bash
# 에러 예시:
# Error: template: my-chart/templates/deployment.yaml:15:20: executing "my-chart/templates/deployment.yaml" 
# at <.Values.image.tag>: nil pointer evaluating interface {}.tag

# 원인: Values에 필요한 키가 없음

# 해결 방법 1: 기본값 설정
{{- .Values.image.tag | default "latest" }}

# 해결 방법 2: 조건부 렌더링
{{- if .Values.image.tag }}
image: {{ .Values.image.repository }}:{{ .Values.image.tag }}
{{- else }}
image: {{ .Values.image.repository }}:latest
{{- end }}

# 해결 방법 3: required 함수 사용
image: {{ .Values.image.repository }}:{{ required "image.tag is required" .Values.image.tag }}
```

**자주 사용하는 템플릿 함수:**
```yaml
# 문자열 처리
{{ .Values.name | upper }}          # 대문자로
{{ .Values.name | lower }}          # 소문자로
{{ .Values.name | quote }}          # 따옴표로 감싸기
{{ .Values.name | trim }}           # 공백 제거

# 기본값
{{ .Values.port | default 8080 }}

# 필수값
{{ required "A valid .Values.license is required!" .Values.license }}

# 조건문
{{- if .Values.enabled }}
enabled: true
{{- end }}

# 반복문
{{- range .Values.items }}
- {{ . }}
{{- end }}

# 헬퍼 함수 호출
{{ include "my-chart.fullname" . }}

# 들여쓰기
{{ .Values.config | indent 4 }}
{{ .Values.config | nindent 4 }}    # 줄바꿈 + 들여쓰기
```

### 6.4 YAML 문법 오류

```bash
# YAML 검증
helm template my-release ./my-chart > output.yaml
kubectl apply --dry-run=client -f output.yaml

# yamllint로 검증 (별도 설치 필요)
pip install yamllint
yamllint output.yaml

# 일반적인 YAML 에러:
# 1. 들여쓰기 불일치
# 2. 탭 문자 사용 (스페이스만 사용해야 함)
# 3. 따옴표 불일치
# 4. 콜론(:) 뒤 공백 누락
```

---

## 7. Repository 문제 해결

### 7.1 Repository 관리

```bash
# Repository 목록 확인
helm repo list

# Repository 추가
helm repo add stable https://charts.helm.sh/stable
helm repo add bitnami https://charts.bitnami.com/bitnami

# Repository 업데이트
helm repo update

# 특정 repository만 업데이트
helm repo update bitnami

# Repository 제거
helm repo remove stable
```

### 7.2 Repository 연결 문제

```bash
# 에러: Repository 접근 불가
# Error: looks like "https://example.com/charts" is not a valid chart repository

# 해결 방법 1: URL 확인
curl -I https://example.com/charts/index.yaml

# 해결 방법 2: Repository 재추가
helm repo remove myrepo
helm repo add myrepo https://correct-url.com/charts
helm repo update

# 해결 방법 3: 인증 필요한 경우
helm repo add myrepo https://charts.example.com \
  --username myuser \
  --password mypassword

# 해결 방법 4: 프록시 설정
export HTTP_PROXY=http://proxy.example.com:8080
export HTTPS_PROXY=http://proxy.example.com:8080
helm repo update
```

### 7.3 Private Repository 설정

```bash
# Username/Password 인증
helm repo add private-repo https://charts.example.com/private \
  --username admin \
  --password secret123

# Token 인증
helm repo add private-repo https://charts.example.com/private \
  --username token \
  --password ghp_xxxxxxxxxxxxxxxxxxxx

# Certificate 인증
helm repo add private-repo https://charts.example.com/private \
  --ca-file ca.crt \
  --cert-file client.crt \
  --key-file client.key

# Repository 설정 확인
cat ~/.config/helm/repositories.yaml
```

### 7.4 로컬 Chart 사용

```bash
# 로컬 디렉토리에서 설치
helm install my-release ./my-chart

# 로컬 패키지 파일에서 설치
helm package ./my-chart
helm install my-release my-chart-0.1.0.tgz

# URL에서 직접 설치
helm install my-release https://example.com/charts/my-chart-0.1.0.tgz

# Git repository에서 설치
git clone https://github.com/example/charts.git
helm install my-release ./charts/my-chart
```

---

## 8. 실전 예제 시나리오

### 시나리오 1: WordPress 설치 실패 - PVC 문제

**증상:** WordPress Chart 설치 후 Pod이 Pending 상태

```bash
# 설치 시도
helm install my-wordpress bitnami/wordpress -n wordpress --create-namespace

# 문제 확인
helm status my-wordpress -n wordpress
# STATUS: deployed
# 하지만 Pod은 Pending...

kubectl get pods -n wordpress
# NAME                          READY   STATUS    RESTARTS   AGE
# my-wordpress-7c8f6d4b-xxxxx   0/1     Pending   0          5m

# 상세 정보 확인
kubectl describe pod my-wordpress-7c8f6d4b-xxxxx -n wordpress
# Events:
#   Warning  FailedScheduling  5m  default-scheduler  0/3 nodes available: 
#   persistentvolumeclaim "data-my-wordpress-mariadb-0" not found
```

**해결 과정:**

```bash
# PVC 상태 확인
kubectl get pvc -n wordpress
# NAME                            STATUS    VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS   AGE
# data-my-wordpress-mariadb-0     Pending                                      standard       5m

# PVC 상세 정보
kubectl describe pvc data-my-wordpress-mariadb-0 -n wordpress
# Events:
#   Warning  ProvisioningFailed  1m  persistentvolume-controller  
#   storageclass.storage.k8s.io "standard" not found

# StorageClass 확인
kubectl get storageclass
# No resources found  ⚠️ 문제 발견!

# 해결 방법 1: StorageClass 생성 (로컬 환경)
cat <<EOF | kubectl apply -f -
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: standard
provisioner: kubernetes.io/no-provisioner
volumeBindingMode: WaitForFirstConsumer
EOF

# 해결 방법 2: persistence 비활성화로 재설치
helm uninstall my-wordpress -n wordpress

helm install my-wordpress bitnami/wordpress -n wordpress \
  --set persistence.enabled=false \
  --set mariadb.primary.persistence.enabled=false

# 설치 확인
kubectl get pods -n wordpress -w
```

### 시나리오 2: Nginx Ingress 업그레이드 실패

**증상:** Nginx Ingress Controller 업그레이드 후 서비스 중단

```bash
# 기존 버전 확인
helm list -n ingress-nginx
# NAME            NAMESPACE       REVISION  CHART                    APP VERSION
# ingress-nginx   ingress-nginx   1         ingress-nginx-4.0.0      1.1.0

# 업그레이드 시도
helm upgrade ingress-nginx ingress-nginx/ingress-nginx \
  -n ingress-nginx \
  --version 4.8.0

# 업그레이드 후 문제 발생
kubectl get pods -n ingress-nginx
# NAME                                        READY   STATUS             RESTARTS   AGE
# ingress-nginx-controller-xxxxx              0/1     CrashLoopBackOff   5          3m
```

**문제 분석:**

```bash
# Pod 로그 확인
kubectl logs ingress-nginx-controller-xxxxx -n ingress-nginx
# Error: Configuration file contains invalid syntax

# Release 상태 확인
helm status ingress-nginx -n ingress-nginx
# STATUS: deployed  (하지만 실제로는 문제 있음)

# 히스토리 확인
helm history ingress-nginx -n ingress-nginx
# REVISION  UPDATED                   STATUS        CHART                  DESCRIPTION
# 1         Mon Jan 15 10:00:00 2024  superseded    ingress-nginx-4.0.0    Install complete
# 2         Mon Jan 28 11:00:00 2024  deployed      ingress-nginx-4.8.0    Upgrade complete

# 설정 변경 사항 확인
helm get values ingress-nginx -n ingress-nginx --revision 1 > old-values.yaml
helm get values ingress-nginx -n ingress-nginx --revision 2 > new-values.yaml
diff old-values.yaml new-values.yaml
```

**해결 과정:**

```bash
# 즉시 롤백
helm rollback ingress-nginx 1 -n ingress-nginx

# 롤백 확인
kubectl get pods -n ingress-nginx -w

# 정상 작동 확인 후 올바른 설정으로 재업그레이드
cat > correct-values.yaml <<EOF
controller:
  service:
    type: LoadBalancer
  config:
    use-forwarded-headers: "true"
    compute-full-forwarded-for: "true"
EOF

# Dry-run으로 먼저 테스트
helm upgrade ingress-nginx ingress-nginx/ingress-nginx \
  -n ingress-nginx \
  --version 4.8.0 \
  -f correct-values.yaml \
  --dry-run --debug

# 실제 업그레이드
helm upgrade ingress-nginx ingress-nginx/ingress-nginx \
  -n ingress-nginx \
  --version 4.8.0 \
  -f correct-values.yaml
```

### 시나리오 3: Custom Chart 템플릿 오류

**증상:** 자체 개발한 Chart 설치 시 템플릿 에러

```bash
# Chart 설치 시도
helm install myapp ./myapp-chart -n production --create-namespace

# 에러 발생:
# Error: template: myapp-chart/templates/deployment.yaml:25:38: 
# executing "myapp-chart/templates/deployment.yaml" at <.Values.database.port>: 
# nil pointer evaluating interface {}.port
```

**문제 분석:**

```bash
# 템플릿 렌더링 테스트
helm template myapp ./myapp-chart --debug

# 문제가 있는 라인 확인
cat myapp-chart/templates/deployment.yaml | sed -n '20,30p'
# 23       - name: DATABASE_HOST
# 24         value: {{ .Values.database.host }}
# 25       - name: DATABASE_PORT
# 26         value: {{ .Values.database.port }}  ⚠️ 문제 라인!

# values.yaml 확인
cat myapp-chart/values.yaml | grep -A 5 database
# database:
#   host: postgres
#   # port 키가 없음!  ⚠️ 원인 발견
```

**해결 과정:**

```bash
# 방법 1: values.yaml에 누락된 값 추가
cat >> myapp-chart/values.yaml <<EOF

database:
  host: postgres
  port: 5432
  name: myapp
EOF

# 방법 2: 템플릿에 기본값 추가
cat > myapp-chart/templates/deployment.yaml <<EOF
...
        - name: DATABASE_PORT
          value: {{ .Values.database.port | default "5432" | quote }}
...
EOF

# 방법 3: required 함수로 필수값 명시
cat > myapp-chart/templates/deployment.yaml <<EOF
...
        - name: DATABASE_PORT
          value: {{ required ".Values.database.port is required!" .Values.database.port | quote }}
...
EOF

# 수정 후 테스트
helm template myapp ./myapp-chart --debug

# 설치
helm install myapp ./myapp-chart -n production
```

### 시나리오 4: Values 파일 우선순위 혼란

**증상:** 여러 values 파일 사용 시 예상과 다른 값 적용

```bash
# 설정 파일들
cat values-base.yaml
# replicaCount: 3
# image:
#   repository: myapp
#   tag: v1.0.0

cat values-production.yaml
# replicaCount: 5
# resources:
#   limits:
#     memory: 2Gi

cat values-override.yaml
# image:
#   tag: v2.0.0

# 설치
helm install myapp ./myapp-chart \
  -f values-base.yaml \
  -f values-production.yaml \
  -f values-override.yaml \
  --set replicaCount=10

# 예상: replicaCount=10, tag=v2.0.0
# 실제 적용된 값 확인
helm get values myapp
```

**검증 및 확인:**

```bash
# 템플릿 렌더링으로 최종 값 확인
helm template myapp ./myapp-chart \
  -f values-base.yaml \
  -f values-production.yaml \
  -f values-override.yaml \
  --set replicaCount=10 | grep -E "replicas:|image:"

# 출력:
# replicas: 10              # --set이 최우선
# image: myapp:v2.0.0       # values-override.yaml이 적용

# JSON으로 확인
helm get values myapp -o json | jq '.'

# 모든 기본값 포함하여 확인
helm get values myapp --all -o json | jq '.'
```

### 시나리오 5: Helm Release Secret 손상

**증상:** Helm 명령어 실행 시 이상한 에러 발생

```bash
# 상태 확인 시도
helm list -n production
# Error: decode: yaml: unmarshal errors

# 또는
helm status myapp -n production
# Error: release: not found
```

**문제 분석:**

```bash
# Secret 확인
kubectl get secrets -n production -l owner=helm

# 손상된 secret 확인
kubectl get secret sh.helm.release.v1.myapp.v3 -n production -o yaml

# Secret 디코딩 시도
kubectl get secret sh.helm.release.v1.myapp.v3 -n production \
  -o jsonpath='{.data.release}' | base64 -d | base64 -d | gunzip
# gzip: stdin: unexpected end of file  ⚠️ 손상됨!
```

**복구 과정:**

```bash
# 방법 1: 이전 revision의 secret 확인
kubectl get secrets -n production -l owner=helm,name=myapp

# 이전 버전이 정상이면 해당 버전으로 롤백
helm rollback myapp 2 -n production

# 방법 2: 손상된 secret 삭제 후 재설치
kubectl delete secret sh.helm.release.v1.myapp.v3 -n production

# Release 완전 제거
helm uninstall myapp -n production

# 재설치
helm install myapp ./myapp-chart -n production -f production-values.yaml

# 방법 3: Secret 수동 복구 (고급)
# 백업이 있는 경우
kubectl apply -f myapp-release-backup.yaml
```

### 시나리오 6: Chart 의존성 버전 충돌

**증상:** Chart 설치 시 의존성 버전 에러

```bash
# Chart.yaml 내용
cat myapp-chart/Chart.yaml
# dependencies:
#   - name: postgresql
#     version: "12.0.0"
#     repository: "https://charts.bitnami.com/bitnami"
#   - name: redis
#     version: "17.0.0"
#     repository: "https://charts.bitnami.com/bitnami"

# 의존성 업데이트 시도
helm dependency update ./myapp-chart
# Error: can't get a valid version for repositories bitnami...
# version "12.0.0" not found
```

**해결 과정:**

```bash
# 사용 가능한 버전 확인
helm search repo bitnami/postgresql --versions | head -20

# Chart.yaml 수정
cat > myapp-chart/Chart.yaml <<EOF
apiVersion: v2
name: myapp
version: 1.0.0
dependencies:
  - name: postgresql
    version: "^12.1.0"    # ^ 사용으로 유연하게
    repository: "https://charts.bitnami.com/bitnami"
  - name: redis
    version: ">=17.0.0 <18.0.0"  # 범위 지정
    repository: "https://charts.bitnami.com/bitnami"
EOF

# 의존성 재업데이트
rm -rf myapp-chart/charts/* myapp-chart/Chart.lock
helm dependency update ./myapp-chart

# 의존성 확인
helm dependency list ./myapp-chart

# 설치
helm install myapp ./myapp-chart -n production
```

---

## 유용한 Helm 디버깅 명령어 모음

### 빠른 진단 명령어

```bash
# 모든 Release 상태 한눈에 보기
helm list -A -o json | jq -r '.[] | "\(.name)\t\(.namespace)\t\(.status)\t\(.chart)"'

# Failed Release 찾기
helm list -A --failed

# Pending Release 찾기
helm list -A --pending

# 특정 Release의 모든 리소스 확인
helm get manifest myapp | kubectl get -f -

# Release의 Hooks 확인
helm get hooks myapp

# Release의 Notes 확인
helm get notes myapp

# Chart의 CRDs 확인
helm show crds bitnami/postgresql

# Chart의 README 확인
helm show readme bitnami/postgresql
```

### 고급 디버깅 기법

```bash
# 템플릿 렌더링 + YAML 검증
helm template myapp ./myapp-chart | kubectl apply --dry-run=client -f -

# 특정 템플릿만 렌더링하여 파일로 저장
helm template myapp ./myapp-chart \
  --show-only templates/deployment.yaml > deployment-rendered.yaml

# Values 병합 결과 확인
helm install myapp ./myapp-chart \
  -f values1.yaml -f values2.yaml \
  --set key=value \
  --dry-run --debug 2>&1 | grep -A 100 "USER-SUPPLIED VALUES"

# Release Secret 내용 확인
kubectl get secret -n <namespace> -l owner=helm,name=<release-name> \
  -o jsonpath='{.items[0].data.release}' | base64 -d | base64 -d | gunzip | jq '.'
```

### Helm 플러그인 활용

```bash
# helm-diff 플러그인 (변경사항 비교)
helm plugin install https://github.com/databus23/helm-diff
helm diff upgrade myapp ./myapp-chart -f new-values.yaml

# helm-secrets 플러그인 (민감정보 암호화)
helm plugin install https://github.com/jkroepke/helm-secrets
helm secrets install myapp ./myapp-chart -f secrets.yaml

# helm-unittest 플러그인 (Chart 테스트)
helm plugin install https://github.com/helm-unittest/helm-unittest
helm unittest ./myapp-chart

# helm-docs 플러그인 (문서 자동 생성)
helm plugin install https://github.com/norwoodj/helm-docs
helm-docs -c ./myapp-chart
```

---

## Helm 트러블슈팅 체크리스트 ✅

### 설치/업그레이드 전

- [ ] `helm lint` - Chart 유효성 검증
- [ ] `helm template --debug` - 템플릿 렌더링 확인
- [ ] `--dry-run` - 시뮬레이션 실행
- [ ] Values 파일 검토
- [ ] 의존성 버전 확인
- [ ] 대상 네임스페이스 확인

### 설치/업그레이드 실패 시

- [ ] `helm status` - Release 상태 확인
- [ ] `helm history` - 히스토리 조회
- [ ] `kubectl get events` - K8s 이벤트 확인
- [ ] `kubectl describe pod` - Pod 상세 정보
- [ ] `kubectl logs` - 로그 확인
- [ ] `helm get manifest` - 실제 배포된 매니페스트 확인

### Release 문제 시

- [ ] `helm list -a` - 모든 Release 상태 확인
- [ ] `helm get values` - 적용된 values 확인
- [ ] Secret 상태 확인
- [ ] `helm rollback` 고려
- [ ] 필요시 완전 재설치

### Chart 개발 시

- [ ] `helm create` - 표준 구조 사용
- [ ] `helm lint --strict` - 엄격한 검증
- [ ] `helm template` - 모든 시나리오 테스트
- [ ] `helm package` - 패키징 테스트
- [ ] `helm install --dry-run` - 설치 시뮬레이션

---

## 추가 팁 💡

### 1. Helm alias 설정

```bash
# ~/.bashrc or ~/.zshrc
alias h='helm'
alias hl='helm list'
alias hla='helm list -A'
alias hi='helm install'
alias hu='helm upgrade'
alias hd='helm uninstall'
alias hs='helm status'
alias hh='helm history'
alias hr='helm rollback'
alias ht='helm template'
alias hg='helm get'
```

### 2. Helm 환경 변수

```bash
# Helm 설정 디렉토리
export HELM_CONFIG_HOME=~/.config/helm
export HELM_DATA_HOME=~/.local/share/helm
export HELM_CACHE_HOME=~/.cache/helm

# Repository 캐시 위치
export HELM_REPOSITORY_CACHE=~/.cache/helm/repository

# 플러그인 디렉토리
export HELM_PLUGINS=~/.local/share/helm/plugins

# 기본 네임스페이스
export HELM_NAMESPACE=default

# 디버그 모드
export HELM_DEBUG=true
```

### 3. 로깅 및 모니터링

```bash
# Helm 작업 로그 저장
helm install myapp ./myapp-chart --debug 2>&1 | tee helm-install.log

# Release 변경 추적
helm history myapp --output json | jq '.'

# 정기적인 상태 확인 스크립트
#!/bin/bash
for release in $(helm list -q -A); do
    echo "Checking $release..."
    helm status $release --show-desc
done
```

### 4. CI/CD 통합

```bash
# GitLab CI 예제
deploy:
  script:
    - helm lint ./myapp-chart --strict
    - helm template ./myapp-chart --validate
    - helm upgrade --install myapp ./myapp-chart
        --namespace production
        --create-namespace
        --wait
        --timeout 10m
        --atomic  # 실패 시 자동 롤백
        --cleanup-on-fail
```

---

## 일반적인 에러 메시지와 해결 방법

| 에러 메시지 | 원인 | 해결 방법 |
|------------|------|----------|
| `release: not found` | Release가 존재하지 않음 | `helm list -a`로 확인 후 재설치 |
| `another operation (install/upgrade/rollback) is in progress` | 이전 작업이 완료되지 않음 | `helm rollback` 또는 secret 정리 |
| `timed out waiting for the condition` | Pod이 정상 시작되지 않음 | `kubectl describe pod`로 원인 파악 |
| `has no deployed releases` | 배포된 revision이 없음 | Release 삭제 후 재설치 |
| `chart requires kubeVersion` | K8s 버전 불일치 | Chart 버전 변경 또는 클러스터 업그레이드 |
| `unable to build kubernetes objects` | YAML 문법 오류 | `helm template --debug`로 확인 |

---

## 결론

Helm 트러블슈팅의 핵심:

1. **사전 검증**: `lint`, `template`, `--dry-run` 적극 활용
2. **단계별 진단**: Release → Chart → Values → 템플릿 순으로 확인
3. **로그와 이벤트**: K8s 이벤트와 Pod 로그를 항상 확인
4. **Rollback 준비**: 문제 발생 시 빠르게 이전 버전으로 복구
5. **문서화**: 설정 변경 사항과 문제 해결 과정 기록

Happy Helming! ⎈
