# ArgoCD 트러블슈팅 가이드 🚀

## 목차
1. [ArgoCD 기본 개념](#1-argocd-기본-개념)
2. [설치 및 초기 설정 문제](#2-설치-및-초기-설정-문제)
3. [Application 동기화 문제](#3-application-동기화-문제)
4. [Git Repository 연결 문제](#4-git-repository-연결-문제)
5. [Health Check 및 상태 문제](#5-health-check-및-상태-문제)
6. [RBAC 및 권한 문제](#6-rbac-및-권한-문제)
7. [성능 및 리소스 문제](#7-성능-및-리소스-문제)
8. [고급 트러블슈팅](#8-고급-트러블슈팅)
9. [실전 예제 시나리오](#9-실전-예제-시나리오)

---

## 1. ArgoCD 기본 개념

### 1.1 ArgoCD란?

ArgoCD는 Kubernetes를 위한 선언적 GitOps 지속적 배포 도구입니다.

**핵심 개념:**
- **Application**: ArgoCD가 관리하는 배포 단위
- **Project**: Application을 그룹화하는 논리적 단위
- **Sync**: Git 저장소와 클러스터 상태를 일치시키는 작업
- **Health Status**: 리소스의 건강 상태 (Healthy, Progressing, Degraded, etc.)
- **Sync Status**: Git과 클러스터 간 동기화 상태 (Synced, OutOfSync)

### 1.2 ArgoCD 아키텍처

```
┌─────────────────────────────────────────┐
│           ArgoCD Components             │
├─────────────────────────────────────────┤
│ API Server    : REST/gRPC API           │
│ Repo Server   : Git 저장소 접근         │
│ App Controller: 상태 모니터링 및 동기화  │
│ Redis         : 캐시                     │
│ Dex           : SSO/OIDC 인증            │
└─────────────────────────────────────────┘
```

### 1.3 기본 명령어

```bash
# ArgoCD CLI 설치
curl -sSL -o argocd https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
chmod +x argocd
sudo mv argocd /usr/local/bin/

# 버전 확인
argocd version

# 로그인
argocd login <ARGOCD_SERVER>

# Application 목록 조회
argocd app list
argocd app list -o wide

# Application 상세 정보
argocd app get <APP_NAME>

# 동기화
argocd app sync <APP_NAME>

# 로그 확인
argocd app logs <APP_NAME>
```

---

## 2. 설치 및 초기 설정 문제

### 2.1 ArgoCD 설치 문제

#### 기본 설치

```bash
# Namespace 생성
kubectl create namespace argocd

# ArgoCD 설치
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# 설치 확인
kubectl get all -n argocd
```

#### 문제: Pod이 시작되지 않음

```bash
# Pod 상태 확인
kubectl get pods -n argocd

# 출력 예시:
# NAME                                  READY   STATUS             RESTARTS   AGE
# argocd-application-controller-0       0/1     CrashLoopBackOff   5          5m
# argocd-redis-7d8d46cc7f-xxxxx         1/1     Running            0          5m
# argocd-repo-server-56b8d9f7d4-xxxxx   0/1     ImagePullBackOff   0          5m
# argocd-server-6d5f8c7dc4-xxxxx        1/1     Running            0          5m
```

**문제 진단:**

```bash
# CrashLoopBackOff 분석
kubectl describe pod argocd-application-controller-0 -n argocd
kubectl logs argocd-application-controller-0 -n argocd --previous

# 일반적인 원인:
# 1. 리소스 부족
# 2. RBAC 권한 문제
# 3. Redis 연결 실패
```

**해결 방법:**

```bash
# 1. 리소스 확인
kubectl top nodes
kubectl describe node <node-name> | grep -A 5 "Allocated resources"

# 2. 리소스 제한 조정
kubectl edit deployment argocd-repo-server -n argocd

# resources 섹션 수정:
# resources:
#   limits:
#     cpu: "1"
#     memory: "1Gi"
#   requests:
#     cpu: "250m"
#     memory: "256Mi"

# 3. Redis 연결 확인
kubectl logs argocd-application-controller-0 -n argocd | grep redis
kubectl get svc argocd-redis -n argocd
```

### 2.2 초기 비밀번호 문제

#### 문제: Admin 비밀번호를 찾을 수 없음

```bash
# 초기 비밀번호는 argocd-initial-admin-secret에 저장
kubectl get secret argocd-initial-admin-secret -n argocd

# Secret이 없는 경우:
# Error from server (NotFound): secrets "argocd-initial-admin-secret" not found
```

**해결 방법:**

```bash
# 방법 1: Secret에서 비밀번호 추출
kubectl get secret argocd-initial-admin-secret -n argocd -o jsonpath="{.data.password}" | base64 -d
echo

# 방법 2: 비밀번호 재설정
kubectl patch secret argocd-secret -n argocd \
  -p '{"stringData": {"admin.password": "'$(htpasswd -nbBC 10 admin <NEW_PASSWORD> | grep -o ":.*" | sed 's/://g')'", "admin.passwordMtime": "'$(date +%FT%T%Z)'"}}'

# 방법 3: bcrypt 사용
argocd account update-password \
  --account admin \
  --current-password <CURRENT> \
  --new-password <NEW>
```

### 2.3 UI 접근 문제

#### 문제: ArgoCD UI에 접근할 수 없음

```bash
# Service 타입 확인
kubectl get svc argocd-server -n argocd

# 출력:
# NAME            TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)
# argocd-server   ClusterIP   10.96.100.100   <none>        80/TCP,443/TCP
```

**해결 방법 1: Port Forward**

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443

# 브라우저에서 접속
# https://localhost:8080
```

**해결 방법 2: LoadBalancer로 변경**

```bash
kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "LoadBalancer"}}'

# External IP 확인
kubectl get svc argocd-server -n argocd -w
```

**해결 방법 3: Ingress 설정**

```yaml
# argocd-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: argocd-server-ingress
  namespace: argocd
  annotations:
    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
    nginx.ingress.kubernetes.io/ssl-passthrough: "true"
    nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"
spec:
  ingressClassName: nginx
  rules:
  - host: argocd.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: argocd-server
            port:
              number: 443
  tls:
  - hosts:
    - argocd.example.com
    secretName: argocd-server-tls
```

```bash
kubectl apply -f argocd-ingress.yaml

# Ingress 상태 확인
kubectl get ingress -n argocd
kubectl describe ingress argocd-server-ingress -n argocd
```

---

## 3. Application 동기화 문제

### 3.1 OutOfSync 상태

#### 문제: Application이 계속 OutOfSync 상태

```bash
# Application 상태 확인
argocd app get myapp

# 출력:
# Name:               myapp
# Project:            default
# Server:             https://kubernetes.default.svc
# Namespace:          production
# URL:                https://argocd.example.com/applications/myapp
# Repo:               https://github.com/myorg/myapp
# Target:             main
# Path:               k8s/overlays/production
# SyncWindow:         Sync Allowed
# Sync Policy:        Manual
# Sync Status:        OutOfSync from main (abc1234)
# Health Status:      Healthy
```

**원인 분석:**

```bash
# Diff 확인
argocd app diff myapp

# 상세한 차이점 확인
argocd app diff myapp --local-repo-root=/path/to/repo

# 클러스터의 실제 리소스 확인
kubectl get deployment myapp -n production -o yaml > cluster-state.yaml
# Git 저장소의 매니페스트와 비교
```

**일반적인 OutOfSync 원인:**

1. **수동으로 클러스터 리소스 변경**
```bash
# 누군가 kubectl로 직접 수정
kubectl scale deployment myapp --replicas=5 -n production

# 해결: Git으로 되돌리기
argocd app sync myapp --force
```

2. **Kustomize/Helm 빌드 결과 변경**
```bash
# ConfigMapGenerator의 해시가 변경됨
# 이전: app-config-abc123
# 현재: app-config-def456

# 해결: 정상 동작, Auto-Sync 활성화 권장
```

3. **Resource Tracking Method 불일치**
```bash
# Application의 tracking method 확인
argocd app get myapp -o yaml | grep -A 5 "syncPolicy"

# Annotation 기반으로 변경
argocd app set myapp --tracking-method annotation
```

### 3.2 Sync 실패

#### 문제: Sync가 실패함

```bash
# Sync 시도
argocd app sync myapp

# 에러:
# FATA[0002] rpc error: code = Unknown desc = 
# ComparisonError: Failed to load target state: 
# `kustomize build` failed exit status 1
```

**진단:**

```bash
# Sync 상태 상세 확인
argocd app get myapp --refresh

# Application 이벤트 확인
kubectl get events -n argocd --sort-by='.lastTimestamp' | grep myapp

# Repo Server 로그 확인
kubectl logs -n argocd -l app.kubernetes.io/name=argocd-repo-server --tail=100

# Application Controller 로그
kubectl logs -n argocd -l app.kubernetes.io/name=argocd-application-controller --tail=100
```

**일반적인 Sync 실패 원인:**

**1. Kustomize 빌드 실패**

```bash
# 로그에서 확인:
# Error: accumulating resources: unable to find 'deployment.yaml'

# Git 저장소 확인
git clone <repo-url>
cd myapp
kubectl kustomize k8s/overlays/production/

# 해결: 경로 수정
```

**2. Helm Values 오류**

```bash
# 로그:
# Error: values don't meet the specifications of the schema

# Application YAML 확인
argocd app get myapp -o yaml

# 해결: values 수정
argocd app set myapp --helm-set-string image.tag=v2.0.0
```

**3. RBAC 권한 부족**

```bash
# 로그:
# Error: deployments.apps is forbidden: User "system:serviceaccount:argocd:argocd-application-controller" 
# cannot create resource "deployments" in API group "apps" in the namespace "production"

# ServiceAccount 권한 확인
kubectl describe sa argocd-application-controller -n argocd
kubectl get clusterrolebinding | grep argocd

# 해결: 권한 부여
kubectl create rolebinding argocd-deploy-production \
  --clusterrole=admin \
  --serviceaccount=argocd:argocd-application-controller \
  -n production
```

### 3.3 Auto-Sync 문제

#### 문제: Auto-Sync가 작동하지 않음

```yaml
# Application 정의
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp
spec:
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
  # ...
```

**진단:**

```bash
# Auto-Sync 설정 확인
argocd app get myapp -o yaml | grep -A 10 "syncPolicy"

# Application Controller 로그
kubectl logs -n argocd -l app.kubernetes.io/name=argocd-application-controller -f | grep myapp

# Git webhook 설정 확인 (GitHub 예시)
# Repository Settings → Webhooks
# Payload URL: https://argocd.example.com/api/webhook
# Content type: application/json
# Events: Push
```

**해결 방법:**

```bash
# 1. Auto-Sync 명시적 활성화
argocd app set myapp --sync-policy automated

# 2. Self-Heal 활성화
argocd app set myapp --self-heal

# 3. Prune 활성화
argocd app set myapp --auto-prune

# 4. Sync 간격 확인 (기본 3분)
kubectl get configmap argocd-cm -n argocd -o yaml | grep timeout

# 5. 수동 refresh
argocd app get myapp --refresh --hard-refresh
```

### 3.4 Sync Waves 문제

#### 예제: 데이터베이스를 먼저 배포하고 애플리케이션 배포

```yaml
# database.yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
  annotations:
    argocd.argoproj.io/sync-wave: "1"  # 첫 번째 wave
spec:
  # ...
---
# application.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  annotations:
    argocd.argoproj.io/sync-wave: "2"  # 두 번째 wave
spec:
  # ...
```

#### 문제: Sync Wave 순서가 지켜지지 않음

```bash
# Application 상태 확인
argocd app get myapp --show-operation

# 로그 확인
kubectl logs -n argocd -l app.kubernetes.io/name=argocd-application-controller | grep wave
```

**해결 방법:**

```yaml
# Sync Hook 사용
apiVersion: batch/v1
kind: Job
metadata:
  name: db-migration
  annotations:
    argocd.argoproj.io/hook: PreSync
    argocd.argoproj.io/hook-delete-policy: BeforeHookCreation
spec:
  template:
    spec:
      containers:
      - name: migrate
        image: migrate-tool:latest
        command: ["migrate", "up"]
      restartPolicy: Never
```

---

## 4. Git Repository 연결 문제

### 4.1 Private Repository 접근

#### 문제: Private Git 저장소에 접근할 수 없음

```bash
# Repository 추가 시도
argocd repo add https://github.com/myorg/private-repo

# 에러:
# FATA[0002] rpc error: code = Unknown desc = 
# authentication required
```

**해결 방법 1: HTTPS with Token**

```bash
# GitHub Personal Access Token 사용
argocd repo add https://github.com/myorg/private-repo \
  --username <github-username> \
  --password <github-token>

# 확인
argocd repo list
```

**해결 방법 2: SSH Key**

```bash
# SSH 키 생성
ssh-keygen -t ed25519 -C "argocd@example.com" -f ~/.ssh/argocd_ed25519 -N ""

# Public 키를 GitHub에 Deploy Key로 등록
cat ~/.ssh/argocd_ed25519.pub

# ArgoCD에 Repository 추가
argocd repo add git@github.com:myorg/private-repo.git \
  --ssh-private-key-path ~/.ssh/argocd_ed25519

# 또는 kubectl로 직접 추가
kubectl create secret generic repo-private \
  -n argocd \
  --from-file=sshPrivateKey=~/.ssh/argocd_ed25519

kubectl label secret repo-private \
  -n argocd \
  argocd.argoproj.io/secret-type=repository
```

**해결 방법 3: GitHub App**

```yaml
# argocd-cm ConfigMap에 추가
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-cm
  namespace: argocd
data:
  repositories: |
    - url: https://github.com/myorg/private-repo
      githubAppID: 12345
      githubAppInstallationID: 67890
      githubAppPrivateKey: |
        -----BEGIN RSA PRIVATE KEY-----
        ...
        -----END RSA PRIVATE KEY-----
```

### 4.2 Repository Credential 문제

#### 문제: Repository 자격 증명이 만료됨

```bash
# Application이 Failed 상태
argocd app get myapp

# 에러:
# ComparisonError: Failed to fetch manifests: 
# authentication failed
```

**진단:**

```bash
# Repository 상태 확인
argocd repo list

# CONNECTION STATUS가 Failed인 경우
# NAME                                 TYPE  URL                                   CONNECTION STATUS
# https://github.com/myorg/repo        git   https://github.com/myorg/repo         Failed

# Repository 상세 정보
kubectl get secret -n argocd -l argocd.argoproj.io/secret-type=repository -o yaml
```

**해결 방법:**

```bash
# 자격 증명 업데이트
argocd repo rm https://github.com/myorg/repo
argocd repo add https://github.com/myorg/repo \
  --username <username> \
  --password <new-token>

# 또는 kubectl로 Secret 업데이트
kubectl edit secret <repo-secret-name> -n argocd
# password 필드를 base64 인코딩된 새 토큰으로 변경
```

### 4.3 Git Submodules 문제

#### 문제: Submodule이 포함된 저장소

```bash
# 에러:
# Failed to fetch submodule: authentication failed
```

**해결 방법:**

```yaml
# Application에서 submodule 비활성화
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp
spec:
  source:
    repoURL: https://github.com/myorg/myapp
    targetRevision: main
    path: k8s
    # Submodule 지원
    submodules:
      - recursive: true

# 또는 ConfigMap에서 전역 설정
# argocd-cm
data:
  resource.customizations: |
    git.submodules.enabled: "true"
```

---

## 5. Health Check 및 상태 문제

### 5.1 Progressing 상태에서 멈춤

#### 문제: Application이 계속 Progressing 상태

```bash
argocd app get myapp

# Health Status: Progressing
# 30분 이상 Progressing 상태 유지
```

**진단:**

```bash
# 리소스별 상태 확인
argocd app get myapp --show-operation

# Kubernetes 이벤트 확인
kubectl get events -n production --sort-by='.lastTimestamp' | tail -20

# Pod 상태 확인
kubectl get pods -n production -l app=myapp
kubectl describe pod <pod-name> -n production
```

**일반적인 원인:**

**1. Readiness Probe 실패**

```bash
# Pod 로그 확인
kubectl logs <pod-name> -n production

# Readiness probe 설정 확인
kubectl get pod <pod-name> -n production -o yaml | grep -A 10 readinessProbe
```

**해결:**

```yaml
# readinessProbe 조정
readinessProbe:
  httpGet:
    path: /health
    port: 8080
  initialDelaySeconds: 30  # 증가
  periodSeconds: 10
  timeoutSeconds: 5
  failureThreshold: 5      # 증가
```

**2. 이미지 Pull 실패**

```bash
# ImagePullBackOff 확인
kubectl describe pod <pod-name> -n production | grep -A 5 Events

# 이미지 레지스트리 인증
kubectl get secret -n production | grep docker

# Secret 생성
kubectl create secret docker-registry regcred \
  --docker-server=registry.example.com \
  --docker-username=<username> \
  --docker-password=<password> \
  -n production
```

### 5.2 Degraded 상태

#### 문제: Application이 Degraded 상태

```bash
argocd app get myapp

# Health Status: Degraded
# Resources: 5 Healthy, 1 Degraded
```

**진단:**

```bash
# Degraded 리소스 확인
argocd app get myapp -o yaml | grep -A 10 "status: Degraded"

# 또는
kubectl get all -n production -l app=myapp
```

**일반적인 원인:**

**1. Deployment의 Replica 불일치**

```bash
kubectl get deployment myapp -n production

# READY: 2/3  ← 1개 Pod이 시작되지 않음
```

**해결:**

```bash
# Pod 상태 확인
kubectl get pods -n production -l app=myapp

# 실패한 Pod 로그 확인
kubectl logs <failed-pod> -n production

# 리소스 부족 확인
kubectl describe pod <failed-pod> -n production | grep -A 5 Events
```

**2. StatefulSet Pod 재시작**

```bash
kubectl get statefulset -n production

# CrashLoopBackOff 확인
kubectl describe pod <statefulset-pod> -n production
```

### 5.3 Custom Health Check

#### 예제: Custom CRD의 Health 상태 정의

```yaml
# argocd-cm ConfigMap에 추가
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-cm
  namespace: argocd
data:
  resource.customizations.health.argoproj.io_Rollout: |
    hs = {}
    if obj.status ~= nil then
      if obj.status.phase ~= nil then
        if obj.status.phase == "Healthy" then
          hs.status = "Healthy"
          hs.message = "Rollout is healthy"
          return hs
        end
        if obj.status.phase == "Progressing" then
          hs.status = "Progressing"
          hs.message = "Rollout is progressing"
          return hs
        end
      end
    end
    hs.status = "Progressing"
    hs.message = "Waiting for rollout"
    return hs
```

**적용:**

```bash
# ConfigMap 수정
kubectl edit configmap argocd-cm -n argocd

# ArgoCD 재시작
kubectl rollout restart deployment argocd-repo-server -n argocd
kubectl rollout restart deployment argocd-server -n argocd
```

---

## 6. RBAC 및 권한 문제

### 6.1 사용자 권한 문제

#### 문제: 사용자가 Application을 생성/수정할 수 없음

```bash
# 에러:
# FATA[0000] rpc error: code = PermissionDenied desc = 
# permission denied
```

**ArgoCD RBAC 구조:**

```
User/Group → Role → Policy → Project/Application
```

**진단:**

```bash
# 현재 사용자 권한 확인
argocd account get-user-info

# RBAC 정책 확인
kubectl get configmap argocd-rbac-cm -n argocd -o yaml
```

**해결 방법:**

```yaml
# argocd-rbac-cm ConfigMap 수정
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-rbac-cm
  namespace: argocd
data:
  policy.default: role:readonly
  policy.csv: |
    # Developer 역할 정의
    p, role:developer, applications, get, */*, allow
    p, role:developer, applications, create, */*, allow
    p, role:developer, applications, update, */*, allow
    p, role:developer, applications, sync, */*, allow
    
    # 특정 프로젝트만 접근
    p, role:team-a-developer, applications, *, team-a/*, allow
    
    # 그룹 매핑
    g, developers, role:developer
    g, team-a-members, role:team-a-developer
    
    # 특정 사용자 권한
    p, alice@example.com, applications, *, */*, allow
```

```bash
# 적용
kubectl apply -f argocd-rbac-cm.yaml

# RBAC 정책 테스트
argocd account can-i sync applications '*/*'
argocd account can-i create applications 'team-a/*'
```

### 6.2 ServiceAccount 권한 문제

#### 문제: ArgoCD가 대상 클러스터에 배포할 수 없음

```bash
# 에러:
# Failed to sync application: User "system:serviceaccount:argocd:argocd-application-controller" 
# cannot create resource "deployments" in namespace "production"
```

**해결 방법 1: ClusterRole 사용**

```yaml
# argocd-application-controller-clusterrole.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: argocd-application-controller
rules:
- apiGroups:
  - '*'
  resources:
  - '*'
  verbs:
  - '*'
- nonResourceURLs:
  - '*'
  verbs:
  - '*'
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: argocd-application-controller
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: argocd-application-controller
subjects:
- kind: ServiceAccount
  name: argocd-application-controller
  namespace: argocd
```

**해결 방법 2: Namespace별 Role (최소 권한)**

```yaml
# argocd-role-production.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: argocd-deploy
  namespace: production
rules:
- apiGroups:
  - ""
  - apps
  - batch
  resources:
  - configmaps
  - secrets
  - services
  - deployments
  - statefulsets
  - jobs
  - cronjobs
  verbs:
  - get
  - list
  - watch
  - create
  - update
  - patch
  - delete
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: argocd-deploy
  namespace: production
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: argocd-deploy
subjects:
- kind: ServiceAccount
  name: argocd-application-controller
  namespace: argocd
```

```bash
kubectl apply -f argocd-role-production.yaml
```

### 6.3 Multi-Cluster 권한 문제

#### 문제: 원격 클러스터에 배포할 수 없음

```bash
# 클러스터 추가
argocd cluster add <context-name>

# 에러:
# FATA[0000] rpc error: code = Unknown desc = 
# authentication failed
```

**해결 방법:**

```bash
# 1. kubeconfig 확인
kubectl config get-contexts

# 2. 클러스터 추가 (ServiceAccount 생성)
argocd cluster add <context-name> --name production-cluster

# 3. 클러스터 목록 확인
argocd cluster list

# 4. 연결 테스트
kubectl get pods -n argocd -l app.kubernetes.io/name=argocd-application-controller
kubectl logs -n argocd -l app.kubernetes.io/name=argocd-application-controller | grep "production-cluster"

# 5. ServiceAccount Token 확인 (대상 클러스터)
kubectl get secret -n kube-system | grep argocd
kubectl describe secret argocd-manager-token-xxxxx -n kube-system
```

---

## 7. 성능 및 리소스 문제

### 7.1 Application 목록 로딩 느림

#### 문제: UI에서 Application 목록이 매우 느리게 로딩됨

**진단:**

```bash
# Application 개수 확인
argocd app list | wc -l

# API Server 로그
kubectl logs -n argocd -l app.kubernetes.io/name=argocd-server --tail=100

# 리소스 사용량
kubectl top pods -n argocd
```

**해결 방법 1: 리소스 증가**

```bash
# API Server 리소스 조정
kubectl edit deployment argocd-server -n argocd
```

```yaml
resources:
  limits:
    cpu: "2"
    memory: "2Gi"
  requests:
    cpu: "500m"
    memory: "512Mi"
```

**해결 방법 2: Redis 캐시 최적화**

```bash
# Redis 리소스 증가
kubectl edit deployment argocd-redis -n argocd

# Redis 설정 확인
kubectl get configmap argocd-cm -n argocd -o yaml | grep redis
```

**해결 방법 3: Application 분산**

```yaml
# Project로 Application 그룹화
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: team-a
  namespace: argocd
spec:
  description: Team A Applications
  sourceRepos:
  - 'https://github.com/myorg/*'
  destinations:
  - namespace: 'team-a-*'
    server: https://kubernetes.default.svc
  clusterResourceWhitelist:
  - group: ''
    kind: Namespace
```

### 7.2 Repo Server 메모리 부족

#### 문제: Repo Server OOMKilled

```bash
kubectl get pods -n argocd

# NAME                                  READY   STATUS       RESTARTS   AGE
# argocd-repo-server-56b8d9f7d4-xxxxx   0/1     OOMKilled    5          10m
```

**진단:**

```bash
# 로그 확인
kubectl logs argocd-repo-server-56b8d9f7d4-xxxxx -n argocd --previous

# 메모리 사용량
kubectl top pod -n argocd -l app.kubernetes.io/name=argocd-repo-server

# 큰 저장소가 있는지 확인
argocd repo list
```

**해결 방법:**

```bash
# Repo Server 리소스 증가
kubectl edit deployment argocd-repo-server -n argocd
```

```yaml
resources:
  limits:
    cpu: "2"
    memory: "4Gi"  # 증가
  requests:
    cpu: "500m"
    memory: "1Gi"  # 증가
```

```yaml
# 또는 ConfigMap에서 캐시 설정 조정
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-cm
  namespace: argocd
data:
  # Git 요청 타임아웃
  timeout.reconciliation: "180s"
  
  # Repository 캐시 크기 제한
  repository.credentials: |
    cache:
      default-expiration: "24h"
```

### 7.3 Application Controller 성능 문제

#### 문제: Sync 작업이 큐에 쌓임

```bash
# Application이 많을 때 sync가 지연됨
argocd app list --refresh

# Application Controller 로그
kubectl logs -n argocd -l app.kubernetes.io/name=argocd-application-controller | grep "operation is pending"
```

**해결 방법 1: 샤딩(Sharding) 활성화**

```yaml
# argocd-application-controller StatefulSet 수정
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: argocd-application-controller
  namespace: argocd
spec:
  replicas: 3  # 여러 인스턴스로 증가
  template:
    spec:
      containers:
      - name: argocd-application-controller
        env:
        - name: ARGOCD_CONTROLLER_REPLICAS
          value: "3"
        - name: ARGOCD_CONTROLLER_SHARD
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
```

**해결 방법 2: Reconciliation Timeout 조정**

```yaml
# argocd-cm ConfigMap
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-cm
  namespace: argocd
data:
  # Application 조정 간격 (기본 3분)
  timeout.reconciliation: "300s"
  
  # 동시 sync 작업 수
  application.resourceTrackingMethod: "annotation"
```

```bash
kubectl apply -f argocd-cm.yaml

# Controller 재시작
kubectl rollout restart statefulset argocd-application-controller -n argocd
```

---

## 8. 고급 트러블슈팅

### 8.1 Notification 문제

#### 문제: Slack/Email 알림이 오지 않음

**설정 예제:**

```yaml
# argocd-notifications-cm ConfigMap
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-notifications-cm
  namespace: argocd
data:
  service.slack: |
    token: $slack-token
  template.app-deployed: |
    message: |
      Application {{.app.metadata.name}} is now running.
    slack:
      attachments: |
        [{
          "title": "{{.app.metadata.name}}",
          "title_link": "{{.context.argocdUrl}}/applications/{{.app.metadata.name}}",
          "color": "#18be52",
          "fields": [{
            "title": "Sync Status",
            "value": "{{.app.status.sync.status}}",
            "short": true
          }]
        }]
  trigger.on-deployed: |
    - when: app.status.operationState.phase in ['Succeeded']
      send: [app-deployed]
```

```yaml
# argocd-notifications-secret
apiVersion: v1
kind: Secret
metadata:
  name: argocd-notifications-secret
  namespace: argocd
stringData:
  slack-token: xoxb-your-slack-token
```

**진단:**

```bash
# Notification Controller 로그
kubectl logs -n argocd -l app.kubernetes.io/name=argocd-notifications-controller

# ConfigMap 확인
kubectl get configmap argocd-notifications-cm -n argocd -o yaml

# Secret 확인
kubectl get secret argocd-notifications-secret -n argocd
```

**해결 방법:**

```bash
# 1. Notification Controller 재시작
kubectl rollout restart deployment argocd-notifications-controller -n argocd

# 2. Application에 annotation 추가
kubectl patch app myapp -n argocd --type merge -p '{"metadata":{"annotations":{"notifications.argoproj.io/subscribe.on-deployed.slack":"my-channel"}}}'

# 또는 Application YAML에 추가
```

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp
  annotations:
    notifications.argoproj.io/subscribe.on-deployed.slack: my-channel
    notifications.argoproj.io/subscribe.on-health-degraded.slack: alerts-channel
```

### 8.2 ApplicationSet 문제

#### 예제: Git Generator로 여러 Application 생성

```yaml
# applicationset.yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: cluster-apps
  namespace: argocd
spec:
  generators:
  - git:
      repoURL: https://github.com/myorg/apps
      revision: HEAD
      directories:
      - path: apps/*
  template:
    metadata:
      name: '{{path.basename}}'
    spec:
      project: default
      source:
        repoURL: https://github.com/myorg/apps
        targetRevision: HEAD
        path: '{{path}}'
      destination:
        server: https://kubernetes.default.svc
        namespace: '{{path.basename}}'
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
```

#### 문제: ApplicationSet이 Application을 생성하지 않음

**진단:**

```bash
# ApplicationSet 상태 확인
kubectl get applicationset -n argocd
kubectl describe applicationset cluster-apps -n argocd

# ApplicationSet Controller 로그
kubectl logs -n argocd -l app.kubernetes.io/name=argocd-applicationset-controller

# 생성된 Application 확인
argocd app list | grep cluster-apps
```

**일반적인 원인:**

**1. Git Generator 경로 문제**

```bash
# 저장소 구조 확인
git clone https://github.com/myorg/apps
tree apps/

# apps/ 디렉토리에 하위 디렉토리가 있는가?
# apps/app1/
# apps/app2/
```

**2. Template 변수 오류**

```yaml
# 잘못된 예
template:
  metadata:
    name: '{{.path.basename}}'  # ❌ 점(.) 추가하면 안 됨

# 올바른 예
template:
  metadata:
    name: '{{path.basename}}'   # ✅
```

**해결:**

```bash
# ApplicationSet 재생성
kubectl delete applicationset cluster-apps -n argocd
kubectl apply -f applicationset.yaml

# 강제 새로고침
kubectl annotate applicationset cluster-apps -n argocd \
  argocd.argoproj.io/refresh=now --overwrite
```

### 8.3 Diff Customization

#### 문제: 무의미한 Diff로 계속 OutOfSync

```bash
# 예: Deployment의 managedFields가 계속 변경됨
argocd app diff myapp

# 출력:
# ... managedFields 변경사항 ...
```

**해결 방법: Diff 무시 설정**

```yaml
# argocd-cm ConfigMap
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-cm
  namespace: argocd
data:
  resource.customizations: |
    all:
      ignoreDifferences: |
        jsonPointers:
        - /metadata/managedFields
        - /metadata/resourceVersion
        - /metadata/generation
        - /metadata/creationTimestamp
        - /status
    
    # Deployment 특정 필드 무시
    apps/Deployment:
      ignoreDifferences: |
        jsonPointers:
        - /spec/replicas  # HPA 사용 시
    
    # Service의 clusterIP 무시
    v1/Service:
      ignoreDifferences: |
        jsonPointers:
        - /spec/clusterIP
        - /spec/clusterIPs
```

```bash
kubectl apply -f argocd-cm.yaml

# Application 새로고침
argocd app get myapp --refresh
```

### 8.4 Server-Side Apply 문제

#### 문제: Apply 충돌 발생

```bash
# 에러:
# Failed to apply manifests: Apply failed with 1 conflict: 
# conflict with "kubectl-client-side-apply"
```

**해결 방법:**

```yaml
# Application에서 Server-Side Apply 활성화
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp
spec:
  syncPolicy:
    syncOptions:
    - ServerSideApply=true  # Server-Side Apply 사용
    - RespectIgnoreDifferences=true
```

```bash
# 또는 CLI로 설정
argocd app set myapp --sync-option ServerSideApply=true

# 재sync
argocd app sync myapp --force
```

---

## 9. 실전 예제 시나리오

### 시나리오 1: 대규모 Multi-Tenant 환경

**상황:** 50개 팀, 각 팀당 10개 Application

**구조:**

```
argocd/
├── projects/
│   ├── team-a.yaml
│   ├── team-b.yaml
│   └── ...
├── applicationsets/
│   ├── team-apps.yaml
│   └── shared-services.yaml
└── apps/
    ├── team-a/
    │   ├── app1/
    │   ├── app2/
    │   └── ...
    └── team-b/
        └── ...
```

**AppProject 설정:**

```yaml
# projects/team-a.yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: team-a
  namespace: argocd
spec:
  description: Team A Project
  
  # 소스 저장소 제한
  sourceRepos:
  - 'https://github.com/myorg/team-a-*'
  - 'https://github.com/myorg/shared-charts'
  
  # 배포 대상 제한
  destinations:
  - namespace: 'team-a-*'
    server: https://kubernetes.default.svc
  - namespace: 'team-a-prod-*'
    server: https://prod-cluster
  
  # 클러스터 리소스 화이트리스트
  clusterResourceWhitelist:
  - group: ''
    kind: Namespace
  
  # Namespace 리소스 화이트리스트
  namespaceResourceWhitelist:
  - group: 'apps'
    kind: Deployment
  - group: ''
    kind: Service
  - group: ''
    kind: ConfigMap
  - group: ''
    kind: Secret
  
  # Sync 정책
  syncWindows:
  - kind: allow
    schedule: '0 9-17 * * 1-5'  # 평일 9-17시만 sync 허용
    duration: 8h
    applications:
    - '*'
    namespaces:
    - 'team-a-prod-*'
```

**ApplicationSet으로 자동 생성:**

```yaml
# applicationsets/team-apps.yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: team-apps
  namespace: argocd
spec:
  generators:
  - matrix:
      generators:
      - git:
          repoURL: https://github.com/myorg/team-manifests
          revision: HEAD
          directories:
          - path: 'teams/*/apps/*'
      - list:
          elements:
          - env: dev
            cluster: https://kubernetes.default.svc
          - env: prod
            cluster: https://prod-cluster
  
  template:
    metadata:
      name: '{{path[1]}}-{{path[3]}}-{{env}}'
      labels:
        team: '{{path[1]}}'
        app: '{{path[3]}}'
        env: '{{env}}'
    spec:
      project: '{{path[1]}}'
      source:
        repoURL: https://github.com/myorg/team-manifests
        targetRevision: HEAD
        path: '{{path}}/{{env}}'
      destination:
        server: '{{cluster}}'
        namespace: '{{path[1]}}-{{env}}-{{path[3]}}'
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
        syncOptions:
        - CreateNamespace=true
```

**문제 발생: 일부 팀의 Application만 동기화 실패**

```bash
# 전체 Application 상태 확인
argocd app list --project team-a -o wide | grep OutOfSync

# 특정 Application 진단
argocd app get team-a-app1-prod

# 원인: Sync Window 제한
# 해결: 수동 sync 또는 Sync Window 수정
argocd app sync team-a-app1-prod --force
```

### 시나리오 2: Blue-Green Deployment with ArgoCD

**구조:**

```
myapp/
├── base/
│   ├── kustomization.yaml
│   ├── deployment.yaml
│   └── service.yaml
└── overlays/
    ├── blue/
    │   └── kustomization.yaml
    └── green/
        └── kustomization.yaml
```

**Application 정의:**

```yaml
# app-blue.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp-blue
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/myorg/myapp
    targetRevision: v1.0.0  # Blue 버전
    path: overlays/blue
  destination:
    server: https://kubernetes.default.svc
    namespace: production
  syncPolicy:
    automated:
      prune: false  # 수동 제어
      selfHeal: true
---
# app-green.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp-green
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/myorg/myapp
    targetRevision: v2.0.0  # Green 버전
    path: overlays/green
  destination:
    server: https://kubernetes.default.svc
    namespace: production
  syncPolicy:
    automated:
      prune: false
      selfHeal: true
```

**배포 프로세스:**

```bash
# 1. Blue 버전 배포 (현재 운영)
kubectl apply -f app-blue.yaml
argocd app sync myapp-blue

# 2. Green 버전 배포 (신규 버전)
kubectl apply -f app-green.yaml
argocd app sync myapp-green

# 3. Green 버전 테스트
kubectl run test --rm -it --image=busybox -- \
  wget -O- http://myapp-green.production:8080/health

# 4. Service 전환 (Rollout)
kubectl patch svc myapp -n production \
  -p '{"spec":{"selector":{"version":"green"}}}'

# 5. 모니터링
argocd app get myapp-green --refresh
kubectl get pods -n production -l version=green -w

# 6. 문제 발생 시 즉시 롤백
kubectl patch svc myapp -n production \
  -p '{"spec":{"selector":{"version":"blue"}}}'

# 7. Green 안정화 확인 후 Blue 제거
argocd app delete myapp-blue
```

**문제 발생: Green 버전이 Degraded**

```bash
# 진단
argocd app get myapp-green
kubectl get pods -n production -l version=green
kubectl logs <green-pod> -n production

# 원인: 새 버전의 환경 변수 오류
# 해결: Git에서 수정 후 재sync
git commit -am "Fix env vars"
git push
argocd app sync myapp-green --prune
```

### 시나리오 3: Disaster Recovery - ArgoCD 복구

**상황:** ArgoCD가 설치된 클러스터가 손상됨

**백업 전략:**

```bash
# 1. Application 정의 백업
kubectl get applications -n argocd -o yaml > applications-backup.yaml
kubectl get appprojects -n argocd -o yaml > projects-backup.yaml
kubectl get applicationsets -n argocd -o yaml > applicationsets-backup.yaml

# 2. Repository 자격 증명 백업
kubectl get secrets -n argocd -l argocd.argoproj.io/secret-type=repository -o yaml > repo-secrets-backup.yaml

# 3. Cluster 자격 증명 백업
kubectl get secrets -n argocd -l argocd.argoproj.io/secret-type=cluster -o yaml > cluster-secrets-backup.yaml

# 4. ConfigMap 백업
kubectl get configmap argocd-cm -n argocd -o yaml > argocd-cm-backup.yaml
kubectl get configmap argocd-rbac-cm -n argocd -o yaml > argocd-rbac-cm-backup.yaml

# 정기 백업 스크립트
cat > backup-argocd.sh <<'EOF'
#!/bin/bash
BACKUP_DIR="/backups/argocd/$(date +%Y%m%d)"
mkdir -p "$BACKUP_DIR"

kubectl get applications -n argocd -o yaml > "$BACKUP_DIR/applications.yaml"
kubectl get appprojects -n argocd -o yaml > "$BACKUP_DIR/projects.yaml"
kubectl get secrets -n argocd -o yaml > "$BACKUP_DIR/secrets.yaml"
kubectl get configmaps -n argocd -o yaml > "$BACKUP_DIR/configmaps.yaml"

echo "Backup completed: $BACKUP_DIR"
EOF

chmod +x backup-argocd.sh
```

**복구 프로세스:**

```bash
# 1. 새 클러스터에 ArgoCD 설치
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# 2. ArgoCD 준비 대기
kubectl wait --for=condition=ready pod -l app.kubernetes.io/name=argocd-server -n argocd --timeout=300s

# 3. ConfigMap 복원
kubectl apply -f argocd-cm-backup.yaml
kubectl apply -f argocd-rbac-cm-backup.yaml

# 4. Repository 자격 증명 복원
kubectl apply -f repo-secrets-backup.yaml

# 5. Cluster 자격 증명 복원
kubectl apply -f cluster-secrets-backup.yaml

# 6. Project 복원
kubectl apply -f projects-backup.yaml

# 7. Application 복원
kubectl apply -f applications-backup.yaml

# 8. 모든 Application sync
argocd app list -o name | xargs -I {} argocd app sync {}

# 9. 상태 확인
argocd app list
argocd app get <app-name>
```

**복구 검증:**

```bash
# 모든 Application 상태 확인
for app in $(argocd app list -o name); do
  echo "Checking $app..."
  argocd app get "$app" --refresh
done

# Health 상태 요약
argocd app list -o wide | awk '{print $3}' | sort | uniq -c
```

### 시나리오 4: Progressive Delivery with Argo Rollouts

**통합 구조:**

```yaml
# rollout.yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: myapp
  namespace: production
spec:
  replicas: 5
  strategy:
    canary:
      steps:
      - setWeight: 20
      - pause: {duration: 5m}
      - setWeight: 40
      - pause: {duration: 5m}
      - setWeight: 60
      - pause: {duration: 5m}
      - setWeight: 80
      - pause: {duration: 5m}
  revisionHistoryLimit: 2
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
      - name: myapp
        image: myapp:v1.0.0
        ports:
        - containerPort: 8080
```

**ArgoCD Application:**

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp-rollout
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/myorg/myapp
    targetRevision: HEAD
    path: k8s/overlays/production
  destination:
    server: https://kubernetes.default.svc
    namespace: production
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

**Health Check 커스터마이징:**

```yaml
# argocd-cm ConfigMap
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-cm
  namespace: argocd
data:
  resource.customizations.health.argoproj.io_Rollout: |
    hs = {}
    if obj.status ~= nil then
      if obj.status.phase == "Healthy" then
        hs.status = "Healthy"
        hs.message = obj.status.message
        return hs
      end
      if obj.status.phase == "Progressing" then
        hs.status = "Progressing"
        hs.message = obj.status.message
        return hs
      end
      if obj.status.phase == "Degraded" then
        hs.status = "Degraded"
        hs.message = obj.status.message
        return hs
      end
    end
    hs.status = "Progressing"
    hs.message = "Waiting for rollout"
    return hs
```

**배포 시나리오:**

```bash
# 1. 새 버전 배포 (Git에 푸시)
git commit -am "Update image to v2.0.0"
git push

# 2. ArgoCD 자동 sync
argocd app get myapp-rollout --refresh

# 3. Rollout 상태 모니터링
kubectl argo rollouts get rollout myapp -n production --watch

# 4. Rollout 진행 중 분석
kubectl argo rollouts status myapp -n production

# 5. 문제 발견 시 중단
kubectl argo rollouts abort myapp -n production

# 6. 롤백
kubectl argo rollouts undo myapp -n production

# 7. 또는 Git에서 이전 버전으로 revert
git revert HEAD
git push
# ArgoCD가 자동으로 이전 버전 sync
```

**문제 발생: Rollout이 멈춤**

```bash
# 상태 확인
kubectl argo rollouts get rollout myapp -n production

# 출력:
# NAME   STRATEGY   STATUS   DESIRED  CURRENT  READY  UPDATED
# myapp  Canary     Paused   5        5        4      2

# 원인: 새 Pod이 Ready 상태가 안 됨
kubectl get pods -n production -l app=myapp
kubectl logs <new-pod> -n production

# 해결: Rollout abort 후 수정
kubectl argo rollouts abort myapp -n production
# Git에서 문제 수정 후 재배포
```

### 시나리오 5: 복잡한 Dependency 관리

**상황:** Application 간 의존성 (Database → Cache → App)

**Sync Wave 사용:**

```yaml
# database.yaml
apiVersion: v1
kind: Service
metadata:
  name: postgres
  annotations:
    argocd.argoproj.io/sync-wave: "1"
spec:
  ports:
  - port: 5432
  selector:
    app: postgres
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
  annotations:
    argocd.argoproj.io/sync-wave: "1"
spec:
  serviceName: postgres
  replicas: 1
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
      - name: postgres
        image: postgres:14
        ports:
        - containerPort: 5432
```

```yaml
# cache.yaml
apiVersion: v1
kind: Service
metadata:
  name: redis
  annotations:
    argocd.argoproj.io/sync-wave: "2"
spec:
  ports:
  - port: 6379
  selector:
    app: redis
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis
  annotations:
    argocd.argoproj.io/sync-wave: "2"
spec:
  replicas: 1
  selector:
    matchLabels:
      app: redis
  template:
    metadata:
      labels:
        app: redis
    spec:
      containers:
      - name: redis
        image: redis:7
        ports:
        - containerPort: 6379
```

```yaml
# app.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  annotations:
    argocd.argoproj.io/sync-wave: "3"
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      initContainers:
      - name: wait-for-db
        image: busybox:1.35
        command:
        - sh
        - -c
        - |
          until nc -z postgres 5432; do
            echo "Waiting for postgres..."
            sleep 2
          done
      - name: wait-for-redis
        image: busybox:1.35
        command:
        - sh
        - -c
        - |
          until nc -z redis 6379; do
            echo "Waiting for redis..."
            sleep 2
          done
      containers:
      - name: myapp
        image: myapp:v1.0.0
        ports:
        - containerPort: 8080
```

**Sync Hook 사용:**

```yaml
# db-migration-job.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: db-migration
  annotations:
    argocd.argoproj.io/hook: PreSync
    argocd.argoproj.io/hook-delete-policy: BeforeHookCreation
    argocd.argoproj.io/sync-wave: "2"
spec:
  template:
    spec:
      containers:
      - name: migrate
        image: flyway:latest
        command:
        - flyway
        - migrate
        env:
        - name: FLYWAY_URL
          value: jdbc:postgresql://postgres:5432/mydb
      restartPolicy: Never
  backoffLimit: 2
```

**배포:**

```bash
# 전체 스택 배포
argocd app sync myapp-stack --async

# 각 Wave별 진행 상황 모니터링
watch -n 2 'kubectl get pods -n production'

# Sync 상태 확인
argocd app get myapp-stack --show-operation

# 출력:
# SYNC-WAVE  KIND          NAME         STATUS     MESSAGE
# 1          Service       postgres     Synced     service/postgres created
# 1          StatefulSet   postgres     Synced     statefulset.apps/postgres created
# 2          Job           db-migration Synced     job.batch/db-migration created
# 2          Service       redis        Synced     service/redis created
# 2          Deployment    redis        Synced     deployment.apps/redis created
# 3          Deployment    myapp        Synced     deployment.apps/myapp created
```

**문제 발생: Migration Job 실패**

```bash
# Job 상태 확인
kubectl get jobs -n production

# Job 로그 확인
kubectl logs job/db-migration -n production

# 원인: Database가 아직 준비 안 됨
# 해결: Wave 순서 조정 또는 initContainer 추가

# Hook 재실행
argocd app sync myapp-stack --force --async
```

---

## 유용한 ArgoCD 명령어 모음

### 빠른 진단 명령어

```bash
# 모든 Application 상태 한눈에
argocd app list -o wide

# OutOfSync Application만 필터링
argocd app list | grep OutOfSync

# Degraded Application만 필터링
argocd app list | grep Degraded

# 특정 Project의 Application
argocd app list --project <project-name>

# Application 상세 정보 (YAML)
argocd app get <app-name> -o yaml

# Diff 확인
argocd app diff <app-name>

# 실시간 로그
argocd app logs <app-name> -f

# 특정 리소스 로그
argocd app logs <app-name> --kind Deployment --name myapp

# Repository 연결 상태
argocd repo list

# Cluster 연결 상태
argocd cluster list

# Project 목록
argocd proj list

# 사용자 권한 확인
argocd account can-i sync applications '*/*'
```

### 대량 작업

```bash
# 모든 Application sync
argocd app list -o name | xargs -I {} argocd app sync {}

# 특정 Project의 모든 Application sync
argocd app list --project myproject -o name | xargs -I {} argocd app sync {}

# OutOfSync Application만 sync
argocd app list | grep OutOfSync | awk '{print $1}' | xargs -I {} argocd app sync {}

# 모든 Application refresh
argocd app list -o name | xargs -I {} argocd app get {} --refresh

# Hard refresh (캐시 무시)
argocd app list -o name | xargs -I {} argocd app get {} --hard-refresh
```

### 디버깅 스크립트

```bash
#!/bin/bash
# argocd-health-check.sh

echo "=== ArgoCD Health Check ==="

# ArgoCD 컴포넌트 상태
echo ""
echo "=== ArgoCD Components ==="
kubectl get pods -n argocd

# Application 상태 요약
echo ""
echo "=== Application Status Summary ==="
argocd app list -o wide | awk 'NR>1 {print $3}' | sort | uniq -c

# Health 상태 요약
echo ""
echo "=== Health Status Summary ==="
argocd app list -o wide | awk 'NR>1 {print $4}' | sort | uniq -c

# Sync 상태 요약
echo ""
echo "=== Sync Status Summary ==="
argocd app list -o wide | awk 'NR>1 {print $5}' | sort | uniq -c

# 문제가 있는 Application
echo ""
echo "=== Applications with Issues ==="
argocd app list | grep -E "OutOfSync|Degraded|Unknown"

# Repository 상태
echo ""
echo "=== Repository Status ==="
argocd repo list

# 리소스 사용량
echo ""
echo "=== Resource Usage ==="
kubectl top pods -n argocd

echo ""
echo "=== Health Check Complete ==="
```

---

## ArgoCD 트러블슈팅 체크리스트 ✅

### 설치 및 초기 설정

- [ ] ArgoCD Pod이 모두 Running 상태인가?
- [ ] 초기 admin 비밀번호를 확인했는가?
- [ ] UI에 접근 가능한가? (Port-forward/LoadBalancer/Ingress)
- [ ] CLI 로그인이 정상 작동하는가?

### Application 문제

- [ ] Git Repository 연결이 정상인가?
- [ ] Repository 자격 증명이 유효한가?
- [ ] Application의 path가 올바른가?
- [ ] Kustomize/Helm 빌드가 성공하는가?
- [ ] Target namespace가 존재하는가?
- [ ] RBAC 권한이 충분한가?

### Sync 문제

- [ ] Sync Policy가 올바르게 설정되었는가?
- [ ] Diff 결과를 확인했는가?
- [ ] Resource Tracking Method가 적절한가?
- [ ] Sync Wave/Hook이 올바르게 작동하는가?
- [ ] Auto-Sync가 필요한 경우 활성화했는가?

### Health 문제

- [ ] Pod이 정상적으로 시작되는가?
- [ ] Readiness/Liveness Probe가 성공하는가?
- [ ] 리소스(CPU/Memory)가 충분한가?
- [ ] Custom Health Check가 필요한가?

### 성능 문제

- [ ] ArgoCD 컴포넌트의 리소스가 충분한가?
- [ ] Redis 캐시가 정상 작동하는가?
- [ ] Application 수가 너무 많지 않은가?
- [ ] Sharding이 필요한가?

---

## 추가 팁 💡

### 1. ArgoCD CLI alias

```bash
# ~/.bashrc or ~/.zshrc
alias arcd='argocd'
alias arcda='argocd app'
alias arcdl='argocd app list'
alias arcds='argocd app sync'
alias arcdg='argocd app get'
alias arcdd='argocd app diff'
alias arcdr='argocd repo'
alias arcdc='argocd cluster'
```

### 2. Best Practices

```yaml
# 1. Application을 Git에서 관리 (App of Apps 패턴)
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: applications
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/myorg/argocd-apps
    targetRevision: HEAD
    path: apps
  destination:
    server: https://kubernetes.default.svc
    namespace: argocd
  syncPolicy:
    automated:
      prune: true
      selfHeal: true

# 2. Project로 권한 분리

# 3. Notification 설정으로 가시성 확보

# 4. 정기 백업 자동화

# 5. Metrics 모니터링 (Prometheus/Grafana)
```

### 3. Monitoring

```bash
# Prometheus metrics 노출
kubectl port-forward svc/argocd-metrics -n argocd 8082:8082

# 주요 metrics:
# - argocd_app_info: Application 정보
# - argocd_app_sync_total: Sync 횟수
# - argocd_app_reconcile: Reconcile 시간
# - argocd_git_request_duration_seconds: Git 요청 시간
```

### 4. 유용한 도구

```bash
# argocd-autopilot: ArgoCD 설치 자동화
# argocd-notifications: 알림 통합
# argocd-image-updater: 이미지 자동 업데이트
# argocd-vault-plugin: Vault 통합
```

---

## 일반적인 에러 메시지와 해결 방법

| 에러 메시지 | 원인 | 해결 방법 |
|------------|------|----------|
| `ComparisonError: rpc error` | Git 저장소 접근 실패 | Repository 자격 증명 확인 |
| `PermissionDenied` | RBAC 권한 부족 | ServiceAccount 권한 부여 |
| `OutOfSync` | Git과 클러스터 상태 불일치 | Diff 확인 후 sync |
| `Progressing` 지속 | Pod이 Ready 안 됨 | Pod 로그 및 이벤트 확인 |
| `health assessment failed` | Health check 실패 | Custom health check 정의 |
| `sync failed: one or more` | 일부 리소스 적용 실패 | 실패한 리소스 개별 확인 |
| `cluster XXX has not been configured` | Cluster 미등록 | argocd cluster add |

---

## 결론

ArgoCD 트러블슈팅의 핵심:

1. **로그가 답이다**: Application Controller, Repo Server 로그 확인
2. **Git이 소스**: 모든 변경은 Git을 통해
3. **권한 확인**: RBAC와 ServiceAccount 권한 검증
4. **단계별 접근**: Sync Wave로 의존성 관리
5. **모니터링**: Notification과 Metrics로 가시성 확보

GitOps를 통한 선언적 배포로 안정적인 운영을 실현하세요! 🚀
