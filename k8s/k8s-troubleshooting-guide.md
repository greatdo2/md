# kubectl을 활용한 Kubernetes 트러블슈팅 가이드 🚀

## 목차
1. [기본 상태 확인](#1-기본-상태-확인)
2. [Pod 트러블슈팅](#2-pod-트러블슈팅)
3. [Service 트러블슈팅](#3-service-트러블슈팅)
4. [로그 및 이벤트 확인](#4-로그-및-이벤트-확인)
5. [리소스 사용량 확인](#5-리소스-사용량-확인)
6. [네트워크 트러블슈팅](#6-네트워크-트러블슈팅)
7. [실전 예제 시나리오](#7-실전-예제-시나리오)

---

## 1. 기본 상태 확인

### 1.1 클러스터 전체 상태 확인

```bash
# 클러스터 정보 확인
kubectl cluster-info

# 노드 상태 확인
kubectl get nodes
kubectl get nodes -o wide

# 모든 네임스페이스의 리소스 확인
kubectl get all --all-namespaces
```

**예제 출력:**
```
NAME           STATUS   ROLES    AGE   VERSION
master-node    Ready    master   10d   v1.28.0
worker-node-1  Ready    worker   10d   v1.28.0
worker-node-2  NotReady worker   10d   v1.28.0  ⚠️ 문제!
```

### 1.2 특정 리소스 상태 확인

```bash
# Pod 상태 확인
kubectl get pods
kubectl get pods -n <namespace>
kubectl get pods --all-namespaces

# Deployment 상태 확인
kubectl get deployments
kubectl get deployments -o wide
```

---

## 2. Pod 트러블슈팅

### 2.1 Pod 상태 이상 진단

#### 문제 상황: Pod이 Pending 상태

```bash
# Pod 상세 정보 확인
kubectl describe pod <pod-name>

# Events 섹션 확인 (가장 중요!)
kubectl describe pod <pod-name> | grep -A 10 Events
```

**일반적인 원인:**
- 리소스 부족 (CPU, Memory)
- 노드 선택자(NodeSelector) 불일치
- PersistentVolume 바인딩 실패

**해결 예제:**
```bash
# 노드의 리소스 확인
kubectl top nodes

# Pod의 리소스 요청량 확인
kubectl get pod <pod-name> -o jsonpath='{.spec.containers[*].resources}'
```

#### 문제 상황: Pod이 CrashLoopBackOff 상태

```bash
# 로그 확인 (현재 컨테이너)
kubectl logs <pod-name>

# 이전 크래시된 컨테이너의 로그 확인
kubectl logs <pod-name> --previous

# 특정 컨테이너의 로그 (멀티 컨테이너 Pod인 경우)
kubectl logs <pod-name> -c <container-name>
```

**실제 예제:**
```bash
# 웹 애플리케이션 Pod이 계속 재시작되는 경우
kubectl logs web-app-78d4f6c9b-xk2pm --previous

# 출력 예시:
# Error: ECONNREFUSED connect to database:5432
# → 데이터베이스 연결 실패가 원인!
```

#### 문제 상황: ImagePullBackOff

```bash
# Pod 상세 정보에서 이미지 관련 에러 확인
kubectl describe pod <pod-name>

# Secret 확인 (Private Registry 사용 시)
kubectl get secrets
kubectl describe secret <registry-secret>
```

**해결 방법:**
```bash
# 올바른 이미지 태그 확인
kubectl set image deployment/<deployment-name> <container-name>=<new-image>

# ImagePullSecrets 생성 (Private Registry용)
kubectl create secret docker-registry regcred \
  --docker-server=<registry-server> \
  --docker-username=<username> \
  --docker-password=<password>
```

### 2.2 Pod 내부 디버깅

```bash
# Pod 안으로 접속
kubectl exec -it <pod-name> -- /bin/bash
kubectl exec -it <pod-name> -- sh

# 멀티 컨테이너 Pod의 특정 컨테이너 접속
kubectl exec -it <pod-name> -c <container-name> -- /bin/bash

# 명령어 직접 실행
kubectl exec <pod-name> -- ls /app
kubectl exec <pod-name> -- env
```

**실전 디버깅 예제:**
```bash
# 네트워크 연결 테스트
kubectl exec <pod-name> -- curl http://api-service:8080/health

# 파일 시스템 확인
kubectl exec <pod-name> -- ls -la /var/log

# 환경 변수 확인
kubectl exec <pod-name> -- printenv | grep DATABASE
```

### 2.3 임시 디버그 Pod 생성

```bash
# 네트워크 디버깅용 임시 Pod 생성
kubectl run debug-pod --image=nicolaka/netshoot -it --rm -- /bin/bash

# 특정 네임스페이스에 디버그 Pod 생성
kubectl run debug-pod -n production --image=busybox -it --rm -- sh
```

---

## 3. Service 트러블슈팅

### 3.1 Service 연결 문제

```bash
# Service 상태 확인
kubectl get svc
kubectl describe svc <service-name>

# Endpoint 확인 (중요!)
kubectl get endpoints <service-name>
```

**문제 예제: Endpoint가 비어있는 경우**

```bash
kubectl get svc my-service
# NAME         TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)
# my-service   ClusterIP   10.96.100.50    <none>        80/TCP

kubectl get endpoints my-service
# NAME         ENDPOINTS   AGE
# my-service   <none>      5m  ⚠️ 문제 발견!

# 원인 파악: Pod Selector 확인
kubectl get svc my-service -o yaml | grep -A 5 selector
kubectl get pods --show-labels
```

**해결 방법:**
```bash
# Selector가 Pod Labels와 일치하는지 확인
kubectl describe svc my-service
kubectl get pods -l app=my-app  # Service selector와 동일한 레이블로 조회
```

### 3.2 Service 연결 테스트

```bash
# ClusterIP로 테스트
kubectl run test-pod --image=busybox -it --rm -- wget -O- http://<service-name>:<port>

# DNS 이름으로 테스트
kubectl run test-pod --image=busybox -it --rm -- nslookup <service-name>

# 다른 네임스페이스의 서비스 테스트
kubectl run test-pod --image=busybox -it --rm -- wget -O- http://<service-name>.<namespace>.svc.cluster.local
```

---

## 4. 로그 및 이벤트 확인

### 4.1 로그 확인 방법

```bash
# 실시간 로그 스트리밍
kubectl logs -f <pod-name>

# 최근 N줄만 확인
kubectl logs --tail=100 <pod-name>

# 특정 시간 이후의 로그
kubectl logs --since=1h <pod-name>

# 모든 컨테이너의 로그 (멀티 컨테이너)
kubectl logs <pod-name> --all-containers=true
```

**실전 예제:**
```bash
# 에러 로그만 필터링
kubectl logs <pod-name> | grep -i error

# 타임스탬프와 함께 보기
kubectl logs <pod-name> --timestamps=true

# 여러 Pod의 로그를 한번에 (같은 레이블)
kubectl logs -l app=nginx --all-containers=true
```

### 4.2 이벤트 확인

```bash
# 전체 이벤트 확인
kubectl get events --sort-by='.lastTimestamp'

# 특정 네임스페이스의 이벤트
kubectl get events -n <namespace> --sort-by='.lastTimestamp'

# 실시간 이벤트 감시
kubectl get events -w

# 특정 Pod 관련 이벤트만 확인
kubectl get events --field-selector involvedObject.name=<pod-name>
```

---

## 5. 리소스 사용량 확인

### 5.1 리소스 모니터링

```bash
# 노드 리소스 사용량
kubectl top nodes

# Pod 리소스 사용량
kubectl top pods
kubectl top pods --all-namespaces
kubectl top pods -n <namespace>

# 특정 Pod의 컨테이너별 리소스
kubectl top pod <pod-name> --containers
```

**예제 출력:**
```
NAME              CPU(cores)   MEMORY(bytes)
web-app-1         250m         512Mi
web-app-2         180m         420Mi
database-pod      450m         2Gi  ⚠️ 높은 사용량!
```

### 5.2 리소스 제한 확인

```bash
# Pod의 리소스 요청/제한 확인
kubectl describe pod <pod-name> | grep -A 5 "Limits\|Requests"

# Deployment의 리소스 설정 확인
kubectl get deployment <deployment-name> -o yaml | grep -A 10 resources
```

---

## 6. 네트워크 트러블슈팅

### 6.1 DNS 문제 진단

```bash
# DNS 서비스 확인
kubectl get svc -n kube-system | grep dns

# CoreDNS Pod 상태 확인
kubectl get pods -n kube-system -l k8s-app=kube-dns

# DNS 테스트
kubectl run dns-test --image=busybox -it --rm -- nslookup kubernetes.default
```

### 6.2 네트워크 정책 확인

```bash
# NetworkPolicy 확인
kubectl get networkpolicies
kubectl describe networkpolicy <policy-name>

# Pod의 IP 주소 확인
kubectl get pods -o wide
```

**연결 테스트 예제:**
```bash
# Pod to Pod 연결 테스트
kubectl exec <source-pod> -- ping <target-pod-ip>

# Pod to Service 연결 테스트
kubectl exec <source-pod> -- curl http://<service-name>:8080

# 외부 연결 테스트
kubectl exec <pod-name> -- curl -I https://www.google.com
```

---

## 7. 실전 예제 시나리오

### 시나리오 1: 웹 애플리케이션이 응답하지 않음

**증상:** 사용자가 웹사이트에 접속할 수 없음

```bash
# 1단계: Pod 상태 확인
kubectl get pods -l app=web

# 2단계: Service 및 Endpoint 확인
kubectl get svc web-service
kubectl get endpoints web-service

# 3단계: Pod 로그 확인
kubectl logs -l app=web --tail=50

# 4단계: Pod 상세 정보 확인
kubectl describe pod <web-pod-name>

# 5단계: 직접 테스트
kubectl exec -it <web-pod-name> -- curl localhost:8080
```

### 시나리오 2: 데이터베이스 연결 실패

**증상:** 애플리케이션이 "Database connection failed" 에러 발생

```bash
# 1단계: 애플리케이션 로그 확인
kubectl logs <app-pod> | grep -i database

# 2단계: 데이터베이스 Pod 상태 확인
kubectl get pods -l app=database
kubectl describe pod <db-pod-name>

# 3단계: Service 연결 확인
kubectl get svc database-service
kubectl get endpoints database-service

# 4단계: 네트워크 연결 테스트
kubectl exec <app-pod> -- nc -zv database-service 5432

# 5단계: 환경 변수 확인
kubectl exec <app-pod> -- env | grep DB_
```

**해결 과정:**
```bash
# ConfigMap 확인
kubectl get configmap app-config -o yaml

# Secret 확인 (비밀번호 등)
kubectl get secret db-credentials -o yaml

# 올바른 값으로 업데이트
kubectl edit configmap app-config

# Pod 재시작
kubectl rollout restart deployment/app-deployment
```

### 시나리오 3: Pod이 계속 재시작됨

**증상:** Pod이 CrashLoopBackOff 상태

```bash
# 1단계: 재시작 횟수 확인
kubectl get pods

# 2단계: 크래시 이유 확인
kubectl describe pod <pod-name> | grep -A 10 "Last State"

# 3단계: 이전 컨테이너 로그 확인
kubectl logs <pod-name> --previous

# 4단계: 현재 로그 확인
kubectl logs <pod-name> -f

# 5단계: Liveness/Readiness Probe 확인
kubectl describe pod <pod-name> | grep -A 5 "Liveness\|Readiness"
```

**일반적인 해결 방법:**
```bash
# Probe 설정 수정
kubectl edit deployment <deployment-name>

# 리소스 제한 늘리기
kubectl set resources deployment <deployment-name> \
  --limits=cpu=500m,memory=512Mi \
  --requests=cpu=200m,memory=256Mi
```

### 시나리오 4: 디스크 공간 부족

**증상:** Pod이 Evicted 상태

```bash
# 1단계: Evicted Pod 확인
kubectl get pods | grep Evicted

# 2단계: 노드 리소스 확인
kubectl describe node <node-name> | grep -A 5 "Allocated resources"

# 3단계: 디스크 사용량 확인
kubectl top nodes

# 4단계: Evicted Pod 정리
kubectl delete pods --field-selector status.phase=Failed
```

---

## 유용한 kubectl 트러블슈팅 명령어 모음

### 빠른 진단 명령어

```bash
# 모든 문제가 있는 Pod 찾기
kubectl get pods --all-namespaces --field-selector=status.phase!=Running,status.phase!=Succeeded

# 최근 이벤트 확인 (에러만)
kubectl get events --field-selector type=Warning --sort-by='.lastTimestamp'

# 리소스 사용량 TOP 5
kubectl top pods --all-namespaces | sort -k3 -rn | head -6

# 준비되지 않은 모든 Pod
kubectl get pods --all-namespaces -o json | jq -r '.items[] | select(.status.conditions[] | select(.type=="Ready" and .status=="False")) | .metadata.namespace + "/" + .metadata.name'
```

### 출력 포맷 활용

```bash
# JSON 형식으로 상세 정보
kubectl get pod <pod-name> -o json

# YAML 형식으로 확인
kubectl get pod <pod-name> -o yaml

# 특정 필드만 추출 (JSONPath)
kubectl get pods -o jsonpath='{.items[*].metadata.name}'

# Custom Columns
kubectl get pods -o custom-columns=NAME:.metadata.name,STATUS:.status.phase,IP:.status.podIP
```

---

## 트러블슈팅 체크리스트 ✅

**Pod 문제 시:**
- [ ] `kubectl get pods` - 상태 확인
- [ ] `kubectl describe pod` - 상세 정보 및 이벤트
- [ ] `kubectl logs` - 로그 확인
- [ ] `kubectl logs --previous` - 이전 로그 확인
- [ ] `kubectl exec` - 컨테이너 내부 진단

**Service/네트워크 문제 시:**
- [ ] `kubectl get svc` - Service 상태
- [ ] `kubectl get endpoints` - Endpoint 확인
- [ ] `kubectl describe svc` - Service 상세 정보
- [ ] Pod Labels와 Service Selector 일치 확인
- [ ] DNS 테스트

**리소스 문제 시:**
- [ ] `kubectl top nodes` - 노드 리소스
- [ ] `kubectl top pods` - Pod 리소스
- [ ] `kubectl describe node` - 노드 상세 정보
- [ ] 리소스 요청/제한 설정 확인

---

## 추가 팁 💡

1. **alias 설정으로 효율성 높이기**
```bash
alias k='kubectl'
alias kgp='kubectl get pods'
alias kgd='kubectl get deployments'
alias kgs='kubectl get svc'
alias kdp='kubectl describe pod'
alias kl='kubectl logs'
```

2. **자주 사용하는 플러그인**
```bash
# kubectl 플러그인 관리자 설치
curl -LO "https://github.com/kubernetes-sigs/krew/releases/latest/download/krew-linux_amd64.tar.gz"

# 유용한 플러그인
kubectl krew install tail      # 여러 Pod 로그 동시 확인
kubectl krew install ctx       # Context 빠르게 전환
kubectl krew install ns        # Namespace 빠르게 전환
```

3. **디버깅 모드 활성화**
```bash
# 자세한 API 요청 확인
kubectl get pods -v=8
```

---

## 결론

Kubernetes 트러블슈팅의 핵심은:
1. **체계적인 접근**: 위에서 아래로 (Node → Pod → Container)
2. **로그가 답이다**: 항상 로그와 이벤트를 먼저 확인
3. **재현 가능성**: 문제를 재현하고 단계별로 확인
4. **도구 활용**: kubectl의 다양한 옵션 활용

Happy Troubleshooting! 🎯
