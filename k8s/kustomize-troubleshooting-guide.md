# Kustomize 트러블슈팅 가이드 🔧

## 목차
1. [Kustomize 기본 개념](#1-kustomize-기본-개념)
2. [기본 진단 및 검증](#2-기본-진단-및-검증)
3. [구조 및 경로 문제](#3-구조-및-경로-문제)
4. [Overlay 및 Patch 문제](#4-overlay-및-patch-문제)
5. [리소스 관리 문제](#5-리소스-관리-문제)
6. [변수 및 ConfigMap/Secret 생성](#6-변수-및-configmapsecret-생성)
7. [고급 기능 트러블슈팅](#7-고급-기능-트러블슈팅)
8. [실전 예제 시나리오](#8-실전-예제-시나리오)

---

## 1. Kustomize 기본 개념

### 1.1 Kustomize란?

Kustomize는 템플릿 없이 Kubernetes YAML을 커스터마이징하는 도구입니다.

**핵심 개념:**
- **Base**: 기본 리소스 정의
- **Overlay**: 환경별 커스터마이징 (dev, staging, prod)
- **Patch**: 기존 리소스 수정
- **kustomization.yaml**: Kustomize 설정 파일

### 1.2 기본 디렉토리 구조

```
my-app/
├── base/
│   ├── kustomization.yaml
│   ├── deployment.yaml
│   ├── service.yaml
│   └── configmap.yaml
└── overlays/
    ├── dev/
    │   ├── kustomization.yaml
    │   └── replica-patch.yaml
    ├── staging/
    │   ├── kustomization.yaml
    │   └── replica-patch.yaml
    └── production/
        ├── kustomization.yaml
        ├── replica-patch.yaml
        └── resource-limits.yaml
```

### 1.3 버전 확인

```bash
# kubectl에 내장된 kustomize 버전
kubectl version --client

# 독립 실행형 kustomize 버전
kustomize version

# 버전별 차이 확인
kubectl kustomize --help
kustomize build --help
```

**중요:** kubectl의 kustomize와 독립 실행형 kustomize는 버전이 다를 수 있으며, 일부 기능 차이가 있습니다.

---

## 2. 기본 진단 및 검증

### 2.1 빌드 테스트

```bash
# Base 빌드
kubectl kustomize base/

# 또는 독립 실행형
kustomize build base/

# Overlay 빌드
kubectl kustomize overlays/production/

# 출력을 파일로 저장
kubectl kustomize overlays/production/ > output.yaml

# 빌드 + 검증
kubectl kustomize overlays/production/ | kubectl apply --dry-run=client -f -
```

### 2.2 에러 확인 방법

```bash
# 상세한 에러 메시지 확인
kubectl kustomize overlays/production/ 2>&1 | tee kustomize-error.log

# YAML 문법 검증
kubectl kustomize overlays/production/ | yamllint -

# 리소스별로 확인
kubectl kustomize overlays/production/ | kubectl apply --dry-run=client -f - -o yaml
```

### 2.3 일반적인 에러 패턴

```bash
# 에러 1: kustomization.yaml을 찾을 수 없음
# Error: unable to find one of 'kustomization.yaml', 'kustomization.yml' or 'Kustomization'

# 원인: 잘못된 디렉토리 또는 파일명 오타
# 해결:
ls -la  # kustomization.yaml 파일 존재 확인
pwd     # 현재 경로 확인

# 에러 2: 리소스를 찾을 수 없음
# Error: accumulating resources: accumulation err='accumulating resources from 'deployment.yaml': 
# evalsymlink failure on '/path/deployment.yaml'

# 원인: 경로가 잘못되었거나 파일이 없음
# 해결:
cat kustomization.yaml  # resources 경로 확인
ls -la deployment.yaml  # 파일 존재 확인
```

---

## 3. 구조 및 경로 문제

### 3.1 Base 구조 문제

#### 문제: Base 리소스를 찾을 수 없음

```bash
# 잘못된 kustomization.yaml
# overlays/production/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

bases:
  - ../../base  # 상대 경로가 잘못됨

# 에러:
# Error: unable to find one of 'kustomization.yaml'...
```

**진단 과정:**

```bash
# 현재 디렉토리 확인
pwd
# /path/to/my-app/overlays/production

# 상대 경로 확인
ls -la ../../base/
# 디렉토리가 존재하는가?

# Base에 kustomization.yaml이 있는가?
ls -la ../../base/kustomization.yaml

# 빌드 테스트
cd ../../base
kubectl kustomize .
cd -
```

**해결 방법:**

```yaml
# overlays/production/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:  # Kustomize v3+ 에서는 resources 사용
  - ../../base

patchesStrategicMerge:
  - replica-patch.yaml
```

### 3.2 리소스 경로 문제

#### 예제: 복잡한 디렉토리 구조

```
my-app/
├── base/
│   ├── kustomization.yaml
│   ├── app/
│   │   ├── deployment.yaml
│   │   └── service.yaml
│   └── database/
│       ├── statefulset.yaml
│       └── service.yaml
└── overlays/
    └── production/
        └── kustomization.yaml
```

**올바른 base/kustomization.yaml:**

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - app/deployment.yaml
  - app/service.yaml
  - database/statefulset.yaml
  - database/service.yaml

# 또는 디렉토리 단위로
# resources:
#   - app
#   - database
# (각 디렉토리에 kustomization.yaml이 있어야 함)
```

**검증:**

```bash
# Base 빌드 테스트
kubectl kustomize base/

# 각 리소스가 포함되는지 확인
kubectl kustomize base/ | grep -E "kind:|name:"
```

### 3.3 순환 참조 문제

```bash
# 에러: cyclic dependency detected

# 잘못된 예제:
# base/kustomization.yaml
resources:
  - ../overlays/dev  # ❌ Overlay를 Base에서 참조

# overlays/dev/kustomization.yaml
resources:
  - ../../base  # Base를 Overlay에서 참조
```

**해결 원칙:**
- Overlay는 Base를 참조할 수 있음
- Base는 Overlay를 참조하면 안 됨
- 같은 레벨의 Overlay끼리는 참조하지 않음

---

## 4. Overlay 및 Patch 문제

### 4.1 Strategic Merge Patch 문제

#### 예제: Deployment에 리소스 제한 추가

```yaml
# base/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 1
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
        image: myapp:1.0.0
        ports:
        - containerPort: 8080
```

#### 문제: Patch가 적용되지 않음

```yaml
# overlays/production/resource-patch.yaml (잘못된 예)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  template:
    spec:
      containers:
      - resources:  # ❌ 컨테이너 이름이 없어서 매칭 안 됨
          limits:
            cpu: "1000m"
            memory: "512Mi"
```

**에러 확인:**

```bash
kubectl kustomize overlays/production/

# 빌드는 성공하지만 resources가 적용되지 않음
# 또는 에러:
# Error: merging from generator &{...}: name not found
```

**올바른 Patch:**

```yaml
# overlays/production/resource-patch.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  template:
    spec:
      containers:
      - name: myapp  # ✅ 컨테이너 이름 매칭 필수
        resources:
          limits:
            cpu: "1000m"
            memory: "512Mi"
          requests:
            cpu: "500m"
            memory: "256Mi"
```

```yaml
# overlays/production/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - ../../base

patchesStrategicMerge:
  - resource-patch.yaml
```

**검증:**

```bash
# Patch 적용 확인
kubectl kustomize overlays/production/ | grep -A 10 "resources:"

# 또는 yq 사용
kubectl kustomize overlays/production/ | yq e 'select(.kind == "Deployment") | .spec.template.spec.containers[0].resources' -
```

### 4.2 JSON Patch 문제

#### 예제: 특정 필드 변경

```yaml
# overlays/production/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - ../../base

patchesJson6902:
  - target:
      group: apps
      version: v1
      kind: Deployment
      name: myapp
    patch: |-
      - op: replace
        path: /spec/replicas
        value: 5
      - op: add
        path: /spec/template/spec/containers/0/env
        value:
          - name: ENVIRONMENT
            value: production
```

#### 문제: 경로가 잘못됨

```bash
# 에러:
# Error: unable to apply JSON patch: remove operation does not apply: doc is missing path

# 원인: 배열 인덱스 오류 또는 존재하지 않는 경로
```

**디버깅:**

```bash
# Base의 구조 확인
kubectl kustomize base/ | yq e 'select(.kind == "Deployment")' - > deployment.yaml
cat deployment.yaml

# 경로 검증
# /spec/template/spec/containers/0/env 경로가 존재하는가?

# JSON Patch 경로 규칙:
# - 배열은 0부터 시작하는 인덱스
# - 존재하지 않는 경로에 replace는 불가 (add 사용)
# - 특수문자(/, ~)는 이스케이프 필요
```

**올바른 Patch:**

```yaml
patchesJson6902:
  - target:
      group: apps
      version: v1
      kind: Deployment
      name: myapp
    patch: |-
      - op: replace
        path: /spec/replicas
        value: 5
      - op: test  # 먼저 테스트
        path: /spec/template/spec/containers/0/env
      - op: add
        path: /spec/template/spec/containers/0/env/-  # 배열 끝에 추가
        value:
          name: ENVIRONMENT
          value: production
```

### 4.3 Patch 우선순위 및 충돌

#### 문제: 여러 Patch가 충돌함

```yaml
# overlays/production/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - ../../base

patchesStrategicMerge:
  - replica-patch.yaml      # replicas: 5
  - another-patch.yaml      # replicas: 3  ❌ 충돌!
```

**진단:**

```bash
# 최종 결과 확인
kubectl kustomize overlays/production/ | yq e 'select(.kind == "Deployment") | .spec.replicas' -

# 어떤 값이 적용되었는가?
# Kustomize는 마지막 패치를 우선 적용
```

**해결 방법:**

```yaml
# 방법 1: Patch 통합
# overlays/production/unified-patch.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 5  # 하나의 값으로 통일
  template:
    spec:
      containers:
      - name: myapp
        resources:
          limits:
            cpu: "1000m"

# overlays/production/kustomization.yaml
patchesStrategicMerge:
  - unified-patch.yaml  # 하나의 패치만 사용
```

---

## 5. 리소스 관리 문제

### 5.1 Name Prefix/Suffix 문제

#### 예제: 리소스 이름에 환경별 접두사 추가

```yaml
# overlays/production/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - ../../base

namePrefix: prod-
nameSuffix: -v1
```

**예상 결과:**
```
myapp -> prod-myapp-v1
myapp-service -> prod-myapp-service-v1
```

#### 문제: Selector가 맞지 않음

```bash
# 빌드 후 확인
kubectl kustomize overlays/production/

# Service의 selector가 변경되지 않아서 Pod을 찾지 못함
# Service selector: app: myapp
# Pod labels: app: prod-myapp-v1  ❌ 불일치!
```

**원인 분석:**

```bash
# base/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: myapp
spec:
  selector:
    app: myapp  # 이 값이 자동으로 변경되지 않음!
  ports:
  - port: 80
    targetPort: 8080
```

**해결 방법 1: commonLabels 사용**

```yaml
# overlays/production/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - ../../base

namePrefix: prod-

commonLabels:
  environment: production
  version: v1

# commonLabels는 모든 리소스의 labels와 selector에 자동 추가
```

**해결 방법 2: namePrefix 사용 제한**

```yaml
# namePrefix/Suffix는 신중하게 사용
# 대신 namespace로 환경 분리 권장

# overlays/production/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: production

resources:
  - ../../base

commonLabels:
  environment: production
```

### 5.2 Namespace 문제

#### 문제: Namespace가 중복 적용됨

```yaml
# base/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  namespace: default  # ❌ Base에 하드코딩
```

```yaml
# overlays/production/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: production  # Overlay에서 변경 시도

resources:
  - ../../base
```

**결과:**

```bash
kubectl kustomize overlays/production/

# Warning: namespace가 중복되거나 변경되지 않을 수 있음
```

**올바른 방법:**

```yaml
# base/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  # namespace 제거! Kustomize가 자동으로 추가
spec:
  # ...
```

```yaml
# overlays/production/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: production  # ✅ 여기서만 지정

resources:
  - ../../base
```

**검증:**

```bash
# Namespace 확인
kubectl kustomize overlays/production/ | grep "namespace:"

# 적용 테스트
kubectl kustomize overlays/production/ | kubectl apply --dry-run=client -f -
```

### 5.3 리소스 참조 문제

#### 문제: ConfigMap/Secret 이름이 변경되어 참조 실패

```yaml
# base/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

configMapGenerator:
  - name: app-config
    files:
      - config.properties

# 생성되는 이름: app-config-h8f7dk2g5m (해시 추가됨)
```

```yaml
# base/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  template:
    spec:
      containers:
      - name: myapp
        envFrom:
        - configMapRef:
            name: app-config  # ❌ 해시가 없어서 찾지 못함!
```

**해결 방법:**

```yaml
# base/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - deployment.yaml

configMapGenerator:
  - name: app-config
    files:
      - config.properties

# Kustomize가 자동으로 참조를 업데이트함
# BUT! configMapGenerator 이후에 resources를 나열하면 안됨
```

**올바른 순서:**

```yaml
# base/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

# 1. 먼저 리소스 정의
resources:
  - deployment.yaml
  - service.yaml

# 2. 그 다음 Generator
configMapGenerator:
  - name: app-config
    files:
      - config.properties

secretGenerator:
  - name: app-secret
    literals:
      - password=secret123
```

**검증:**

```bash
# ConfigMap 이름 확인
kubectl kustomize base/ | grep -A 3 "kind: ConfigMap"

# Deployment의 참조 확인
kubectl kustomize base/ | grep -A 5 "envFrom:"

# 이름이 자동으로 업데이트 되었는지 확인
# configMapRef:
#   name: app-config-h8f7dk2g5m  ✅
```

---

## 6. 변수 및 ConfigMap/Secret 생성

### 6.1 ConfigMapGenerator 문제

#### 예제: 파일에서 ConfigMap 생성

```yaml
# base/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

configMapGenerator:
  - name: app-config
    files:
      - config.properties
      - application.yaml
```

#### 문제: 파일을 찾을 수 없음

```bash
# 에러:
# Error: couldn't make configmap: unable to add file 'config.properties' 
# to the configmap: evalsymlink failure

# 원인: 상대 경로 문제
```

**디버깅:**

```bash
# kustomization.yaml 위치 확인
pwd
# /path/to/my-app/base

# 파일 존재 확인
ls -la config.properties
ls -la application.yaml

# 파일이 다른 디렉토리에 있는 경우
tree .
```

**해결 방법:**

```yaml
# base/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

configMapGenerator:
  - name: app-config
    files:
      - configs/config.properties  # 상대 경로 지정
      - configs/application.yaml
    # 또는
    # files:
    #   - config.properties=configs/config.properties  # 키=경로 형식
```

### 6.2 SecretGenerator 문제

#### 예제: 민감한 정보를 Secret으로 생성

```yaml
# overlays/production/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - ../../base

secretGenerator:
  - name: db-secret
    literals:
      - username=admin
      - password=prod-password-123
    type: Opaque
```

#### 문제: Secret이 Git에 노출됨

**보안 이슈:**
- literals로 직접 입력한 값이 Git에 커밋됨
- Base64 인코딩만으로는 보안 불충분

**해결 방법 1: 환경 변수 사용**

```bash
# .env 파일 생성 (Git에서 제외)
cat > .env <<EOF
DB_USERNAME=admin
DB_PASSWORD=prod-password-123
EOF

# .gitignore에 추가
echo ".env" >> .gitignore
```

```yaml
# overlays/production/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - ../../base

secretGenerator:
  - name: db-secret
    envs:
      - .env  # 환경 변수 파일에서 읽기
```

**해결 방법 2: SOPS (Secrets OPerationS) 사용**

```bash
# SOPS 설치
# brew install sops (macOS)
# or download from https://github.com/mozilla/sops

# Secret 파일 암호화
cat > secret.yaml <<EOF
apiVersion: v1
kind: Secret
metadata:
  name: db-secret
type: Opaque
stringData:
  username: admin
  password: prod-password-123
EOF

# 암호화 (Age 또는 GPG 사용)
sops -e secret.yaml > secret.enc.yaml

# Git에는 암호화된 파일만 커밋
git add secret.enc.yaml
```

**해결 방법 3: External Secrets Operator**

```yaml
# Kubernetes에서 External Secrets Operator 사용
# AWS Secrets Manager, HashiCorp Vault 등과 연동
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: db-secret
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: aws-secrets-manager
    kind: SecretStore
  target:
    name: db-secret
  data:
    - secretKey: password
      remoteRef:
        key: prod/db/password
```

### 6.3 ConfigMap/Secret 업데이트 문제

#### 문제: ConfigMap 변경 시 Pod이 자동 재시작되지 않음

```yaml
# overlays/production/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - ../../base

configMapGenerator:
  - name: app-config
    behavior: create  # 또는 replace, merge
    files:
      - config.properties
```

**ConfigMapGenerator behavior 옵션:**
- `create`: 새로운 ConfigMap 생성 (기본값, 해시 추가)
- `replace`: 기존 ConfigMap 교체 (해시 없음)
- `merge`: 기존 ConfigMap과 병합

#### 문제: replace 사용 시 롤링 업데이트 안 됨

```yaml
configMapGenerator:
  - name: app-config
    behavior: replace  # 해시가 없어서 이름이 안 바뀜
    files:
      - config.properties

# ConfigMap은 업데이트되지만
# Deployment의 spec.template이 변경되지 않아
# Pod이 재시작되지 않음!
```

**해결 방법 1: 기본 동작 사용 (create)**

```yaml
# overlays/production/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - ../../base

configMapGenerator:
  - name: app-config
    # behavior: create (기본값)
    files:
      - config.properties

# ConfigMap 이름이 매번 달라져서 자동으로 롤링 업데이트
```

**해결 방법 2: Annotation 추가**

```yaml
# overlays/production/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - ../../base

configMapGenerator:
  - name: app-config
    behavior: replace
    files:
      - config.properties

# ConfigMap 변경 시 Deployment에 annotation 추가
patchesStrategicMerge:
  - |-
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: myapp
    spec:
      template:
        metadata:
          annotations:
            configmap/checksum: ${CONFIG_CHECKSUM}  # 수동 업데이트 필요
```

**해결 방법 3: Reloader 사용**

```bash
# Reloader 설치 (자동으로 ConfigMap/Secret 변경 감지)
kubectl apply -f https://raw.githubusercontent.com/stakater/Reloader/master/deployments/kubernetes/reloader.yaml

# Deployment에 annotation 추가
```

```yaml
# base/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  annotations:
    reloader.stakater.com/auto: "true"  # 자동 재시작
spec:
  # ...
```

---

## 7. 고급 기능 트러블슈팅

### 7.1 Components 사용 문제

#### Components란?
재사용 가능한 구성 요소 (선택적으로 활성화)

```
my-app/
├── base/
│   └── kustomization.yaml
├── components/
│   ├── monitoring/
│   │   ├── kustomization.yaml
│   │   └── servicemonitor.yaml
│   └── ingress/
│       ├── kustomization.yaml
│       └── ingress.yaml
└── overlays/
    └── production/
        └── kustomization.yaml
```

#### 예제: Monitoring Component

```yaml
# components/monitoring/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1alpha1
kind: Component

resources:
  - servicemonitor.yaml

patches:
  - target:
      kind: Deployment
    patch: |-
      - op: add
        path: /spec/template/metadata/annotations
        value:
          prometheus.io/scrape: "true"
          prometheus.io/port: "8080"
```

```yaml
# overlays/production/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - ../../base

components:
  - ../../components/monitoring
  - ../../components/ingress
```

#### 문제: Component를 찾을 수 없음

```bash
# 에러:
# Error: accumulating components: accumulation err='accumulating resources from 
# '../../components/monitoring': must build at directory

# 원인: Component 디렉토리에 kustomization.yaml이 없거나 kind가 잘못됨
```

**해결:**

```yaml
# components/monitoring/kustomization.yaml
# kind를 Component로 설정해야 함!
apiVersion: kustomize.config.k8s.io/v1alpha1
kind: Component  # ✅ Kustomization이 아님!

resources:
  - servicemonitor.yaml
```

### 7.2 Transformers 문제

#### 예제: 이미지 태그 변경

```yaml
# overlays/production/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - ../../base

images:
  - name: myapp
    newName: registry.example.com/myapp
    newTag: v2.0.0
```

#### 문제: 이미지가 변경되지 않음

```bash
# 빌드 확인
kubectl kustomize overlays/production/ | grep "image:"
# image: myapp:1.0.0  # ❌ 변경 안 됨!
```

**원인 분석:**

```yaml
# base/deployment.yaml 확인
spec:
  containers:
  - name: myapp
    image: my-app:1.0.0  # ❌ 이름이 myapp이 아님!
```

**해결:**

```yaml
# overlays/production/kustomization.yaml
images:
  - name: my-app  # ✅ Base의 이미지 이름과 정확히 일치해야 함
    newName: registry.example.com/myapp
    newTag: v2.0.0

# 또는 digest 사용
images:
  - name: my-app
    newName: registry.example.com/myapp
    digest: sha256:abc123...
```

### 7.3 Vars (변수) 사용 문제

#### 예제: Service 이름을 환경 변수로 주입

```yaml
# base/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - deployment.yaml
  - service.yaml

vars:
  - name: SERVICE_NAME
    objref:
      kind: Service
      name: myapp
      apiVersion: v1
    fieldref:
      fieldpath: metadata.name
```

```yaml
# base/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  template:
    spec:
      containers:
      - name: myapp
        env:
        - name: SERVICE_NAME
          value: $(SERVICE_NAME)  # 변수 참조
```

#### 문제: 변수가 치환되지 않음

```bash
kubectl kustomize base/

# 결과:
# env:
# - name: SERVICE_NAME
#   value: $(SERVICE_NAME)  # ❌ 변수 그대로 남음
```

**원인:**
- Kustomize v4+에서 vars는 deprecated
- 대신 replacements 사용 권장

**해결 (Kustomize v4+):**

```yaml
# base/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - deployment.yaml
  - service.yaml

replacements:
  - source:
      kind: Service
      name: myapp
      fieldPath: metadata.name
    targets:
      - select:
          kind: Deployment
          name: myapp
        fieldPaths:
          - spec.template.spec.containers.[name=myapp].env.[name=SERVICE_NAME].value
```

---

## 8. 실전 예제 시나리오

### 시나리오 1: Multi-Environment 설정

**목표:** Dev, Staging, Production 환경별 설정 관리

```
my-app/
├── base/
│   ├── kustomization.yaml
│   ├── deployment.yaml
│   ├── service.yaml
│   └── configmap.yaml
└── overlays/
    ├── dev/
    │   ├── kustomization.yaml
    │   └── patches/
    ├── staging/
    │   ├── kustomization.yaml
    │   └── patches/
    └── production/
        ├── kustomization.yaml
        └── patches/
```

**Base 설정:**

```yaml
# base/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - deployment.yaml
  - service.yaml

configMapGenerator:
  - name: app-config
    literals:
      - LOG_LEVEL=info
      - MAX_CONNECTIONS=100
```

```yaml
# base/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 1
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
        image: myapp:1.0.0
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 200m
            memory: 256Mi
        envFrom:
        - configMapRef:
            name: app-config
```

**Development Overlay:**

```yaml
# overlays/dev/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: dev

resources:
  - ../../base

namePrefix: dev-

commonLabels:
  environment: dev

configMapGenerator:
  - name: app-config
    behavior: merge
    literals:
      - LOG_LEVEL=debug
      - MAX_CONNECTIONS=10

patchesStrategicMerge:
  - patches/replica-patch.yaml
```

```yaml
# overlays/dev/patches/replica-patch.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 1
```

**Production Overlay:**

```yaml
# overlays/production/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: production

resources:
  - ../../base

commonLabels:
  environment: production

commonAnnotations:
  managed-by: kustomize

configMapGenerator:
  - name: app-config
    behavior: merge
    literals:
      - LOG_LEVEL=warn
      - MAX_CONNECTIONS=1000

patchesStrategicMerge:
  - patches/replica-patch.yaml
  - patches/resource-patch.yaml

images:
  - name: myapp
    newTag: 2.0.0
```

```yaml
# overlays/production/patches/replica-patch.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 5
```

```yaml
# overlays/production/patches/resource-patch.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  template:
    spec:
      containers:
      - name: myapp
        resources:
          requests:
            cpu: 500m
            memory: 512Mi
          limits:
            cpu: 1000m
            memory: 1Gi
```

**빌드 및 배포:**

```bash
# Dev 환경 배포
kubectl apply -k overlays/dev/

# Staging 환경 배포
kubectl apply -k overlays/staging/

# Production 환경 배포 (Dry-run 먼저)
kubectl apply -k overlays/production/ --dry-run=client

# 실제 배포
kubectl apply -k overlays/production/

# 각 환경 확인
kubectl get all -n dev
kubectl get all -n staging
kubectl get all -n production
```

### 시나리오 2: Blue-Green Deployment

**목표:** Blue-Green 배포 전략 구현

```
my-app/
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

**Blue Overlay:**

```yaml
# overlays/blue/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: production

resources:
  - ../../base

nameSuffix: -blue

commonLabels:
  version: blue
  environment: production

images:
  - name: myapp
    newTag: v1.0.0

patchesStrategicMerge:
  - |-
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: myapp
    spec:
      replicas: 3
```

**Green Overlay:**

```yaml
# overlays/green/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: production

resources:
  - ../../base

nameSuffix: -green

commonLabels:
  version: green
  environment: production

images:
  - name: myapp
    newTag: v2.0.0

patchesStrategicMerge:
  - |-
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: myapp
    spec:
      replicas: 3
```

**Service 전환:**

```yaml
# base/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: myapp
spec:
  selector:
    app: myapp
    version: blue  # 또는 green으로 변경하여 트래픽 전환
  ports:
  - port: 80
    targetPort: 8080
```

**배포 프로세스:**

```bash
# 1. Blue 버전 배포
kubectl apply -k overlays/blue/

# 2. Green 버전 배포 (트래픽은 아직 Blue)
kubectl apply -k overlays/green/

# 3. Green 버전 테스트
kubectl run test-pod --rm -it --image=busybox -- \
  wget -O- http://myapp-green:8080

# 4. Service를 Green으로 전환
kubectl patch service myapp -n production \
  -p '{"spec":{"selector":{"version":"green"}}}'

# 5. 확인 후 Blue 제거
kubectl delete deployment myapp-blue -n production
```

### 시나리오 3: 복잡한 Patch 적용 실패

**증상:** 여러 필드를 수정하는 복잡한 Patch가 제대로 작동하지 않음

```yaml
# overlays/production/complex-patch.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 5
  template:
    spec:
      containers:
      - name: myapp
        resources:
          limits:
            cpu: "2000m"
            memory: "2Gi"
        env:
        - name: NEW_ENV
          value: "production"
      initContainers:  # 새로운 initContainer 추가
      - name: init
        image: busybox
        command: ['sh', '-c', 'echo init']
```

**문제:** Patch 적용 후 일부만 반영되거나 에러 발생

**디버깅:**

```bash
# 1. Base 구조 확인
kubectl kustomize base/ > base-output.yaml
cat base-output.yaml

# 2. Patch 적용 결과 확인
kubectl kustomize overlays/production/ > prod-output.yaml
cat prod-output.yaml

# 3. Diff 확인
diff base-output.yaml prod-output.yaml

# 4. Strategic Merge 동작 이해
# - containers: 이름으로 매칭
# - env: 이름으로 매칭
# - initContainers: 병합됨
```

**해결: 명확한 Patch 분리**

```yaml
# overlays/production/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - ../../base

patchesStrategicMerge:
  - patches/replica-patch.yaml
  - patches/resource-patch.yaml
  - patches/env-patch.yaml
  - patches/init-container-patch.yaml
```

```yaml
# overlays/production/patches/replica-patch.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 5
```

```yaml
# overlays/production/patches/resource-patch.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  template:
    spec:
      containers:
      - name: myapp
        resources:
          limits:
            cpu: "2000m"
            memory: "2Gi"
          requests:
            cpu: "1000m"
            memory: "1Gi"
```

```yaml
# overlays/production/patches/env-patch.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  template:
    spec:
      containers:
      - name: myapp
        env:
        - name: NEW_ENV
          value: "production"
```

```yaml
# overlays/production/patches/init-container-patch.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  template:
    spec:
      initContainers:
      - name: init
        image: busybox:1.35
        command: ['sh', '-c', 'echo Initializing...']
```

### 시나리오 4: Helm Chart를 Kustomize로 변환

**목표:** 기존 Helm Chart를 Kustomize로 마이그레이션

```bash
# Helm으로 템플릿 렌더링
helm template my-release ./my-helm-chart > base-from-helm.yaml

# Base 디렉토리 생성
mkdir -p my-app/base
cd my-app/base

# 리소스별로 파일 분리
cat base-from-helm.yaml | yq e 'select(.kind == "Deployment")' - > deployment.yaml
cat base-from-helm.yaml | yq e 'select(.kind == "Service")' - > service.yaml
cat base-from-helm.yaml | yq e 'select(.kind == "ConfigMap")' - > configmap.yaml
```

**kustomization.yaml 생성:**

```yaml
# base/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - deployment.yaml
  - service.yaml
  - configmap.yaml

# Helm의 Values를 ConfigMapGenerator로 변환
configMapGenerator:
  - name: app-config
    literals:
      - APP_NAME=myapp
      - REPLICAS=3
```

**Helm Values를 Overlay로 변환:**

```yaml
# overlays/production/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - ../../base

# Helm values.production.yaml의 내용을 patch로 변환
patchesStrategicMerge:
  - |-
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: myapp
    spec:
      replicas: 5  # values.production.yaml의 replicaCount
```

### 시나리오 5: GitOps with Kustomize

**목표:** ArgoCD/Flux와 Kustomize 통합

```
gitops-repo/
├── apps/
│   └── myapp/
│       ├── base/
│       │   ├── kustomization.yaml
│       │   └── ...
│       └── overlays/
│           ├── dev/
│           ├── staging/
│           └── production/
└── clusters/
    ├── dev-cluster/
    │   └── kustomization.yaml
    ├── staging-cluster/
    │   └── kustomization.yaml
    └── production-cluster/
        └── kustomization.yaml
```

**Cluster-level Kustomization:**

```yaml
# clusters/production-cluster/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - ../../apps/myapp/overlays/production
  - ../../apps/database/overlays/production
  - ../../apps/cache/overlays/production

# 클러스터 전체 설정
commonLabels:
  cluster: production
  managed-by: argocd

commonAnnotations:
  argocd.argoproj.io/sync-wave: "1"
```

**ArgoCD Application:**

```yaml
# argocd/myapp-production.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp-production
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/myorg/gitops-repo
    targetRevision: main
    path: apps/myapp/overlays/production
  destination:
    server: https://kubernetes.default.svc
    namespace: production
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

**배포 및 검증:**

```bash
# ArgoCD CLI로 배포
argocd app create -f argocd/myapp-production.yaml
argocd app sync myapp-production

# 또는 kubectl로 직접 배포
kubectl apply -k clusters/production-cluster/

# 상태 확인
argocd app get myapp-production
kubectl get all -n production
```

---

## 유용한 Kustomize 명령어 모음

### 빠른 진단 명령어

```bash
# 빌드 및 YAML 검증
kubectl kustomize . | kubectl apply --dry-run=client -f -

# 특정 리소스만 확인
kubectl kustomize . | yq e 'select(.kind == "Deployment")' -

# 리소스 개수 확인
kubectl kustomize . | yq e '.kind' - | sort | uniq -c

# ConfigMap/Secret 확인
kubectl kustomize . | grep -A 5 "kind: ConfigMap"
kubectl kustomize . | grep -A 5 "kind: Secret"

# 이미지 태그 확인
kubectl kustomize . | grep "image:"

# Namespace 확인
kubectl kustomize . | grep "namespace:"

# Labels 확인
kubectl kustomize . | yq e '.metadata.labels' -
```

### 디버깅 스크립트

```bash
#!/bin/bash
# kustomize-debug.sh

echo "=== Kustomize Validation ==="

# kustomization.yaml 존재 확인
if [ ! -f "kustomization.yaml" ]; then
    echo "❌ kustomization.yaml not found!"
    exit 1
fi
echo "✅ kustomization.yaml found"

# 빌드 테스트
echo ""
echo "=== Building Kustomization ==="
if kubectl kustomize . > /tmp/kustomize-output.yaml 2>&1; then
    echo "✅ Build successful"
else
    echo "❌ Build failed"
    kubectl kustomize . 2>&1
    exit 1
fi

# YAML 검증
echo ""
echo "=== Validating YAML ==="
if kubectl apply --dry-run=client -f /tmp/kustomize-output.yaml > /dev/null 2>&1; then
    echo "✅ YAML validation passed"
else
    echo "❌ YAML validation failed"
    kubectl apply --dry-run=client -f /tmp/kustomize-output.yaml 2>&1
    exit 1
fi

# 리소스 통계
echo ""
echo "=== Resource Statistics ==="
cat /tmp/kustomize-output.yaml | yq e '.kind' - | sort | uniq -c

echo ""
echo "✅ All checks passed!"
```

### CI/CD 통합 예제

```yaml
# .github/workflows/kustomize-validate.yml
name: Kustomize Validation

on:
  pull_request:
    paths:
      - 'k8s/**'

jobs:
  validate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Setup Kustomize
        run: |
          curl -s "https://raw.githubusercontent.com/kubernetes-sigs/kustomize/master/hack/install_kustomize.sh" | bash
          sudo mv kustomize /usr/local/bin/
      
      - name: Validate Base
        run: |
          kustomize build k8s/base | kubectl apply --dry-run=client -f -
      
      - name: Validate Overlays
        run: |
          for overlay in k8s/overlays/*/; do
            echo "Validating $overlay"
            kustomize build "$overlay" | kubectl apply --dry-run=client -f -
          done
```

---

## Kustomize 트러블슈팅 체크리스트 ✅

### 빌드 실패 시

- [ ] `kustomization.yaml` 파일 존재 확인
- [ ] 파일명 정확히 확인 (kustomization.yaml, 대소문자 구분)
- [ ] `resources` 경로 확인
- [ ] 상대 경로 검증
- [ ] YAML 문법 오류 확인
- [ ] API 버전 및 Kind 확인

### Patch 문제 시

- [ ] Patch 대상 리소스 이름 일치 확인
- [ ] Strategic Merge: 컨테이너 이름 매칭 확인
- [ ] JSON Patch: 경로 정확성 확인
- [ ] Patch 순서 확인
- [ ] 여러 Patch 충돌 확인

### ConfigMap/Secret 문제 시

- [ ] Generator에서 참조하는 파일 존재 확인
- [ ] 상대 경로 검증
- [ ] behavior 설정 확인 (create/replace/merge)
- [ ] 리소스에서 정확한 이름 참조 확인
- [ ] Git에 민감한 정보 커밋 여부 확인

### 배포 실패 시

- [ ] Namespace 존재 확인
- [ ] RBAC 권한 확인
- [ ] 리소스 이름 충돌 확인
- [ ] Selector와 Label 매칭 확인
- [ ] 이미지 태그 정확성 확인

---

## 추가 팁 💡

### 1. Kustomize와 Helm 비교

| 기능 | Kustomize | Helm |
|------|-----------|------|
| 템플릿 | ❌ 템플릿 없음 | ✅ Go 템플릿 |
| 학습 곡선 | 쉬움 | 중간 |
| 버전 관리 | Git | Chart 버전 |
| Rollback | Git revert | helm rollback |
| 종속성 | ❌ | ✅ |
| kubectl 통합 | ✅ 내장 | ❌ 별도 도구 |

### 2. Best Practices

```bash
# 1. Base는 최소한으로 유지
# - 환경 특정 설정은 Overlay에
# - 공통 설정만 Base에

# 2. 명확한 디렉토리 구조
# - base/, overlays/, components/
# - 환경별 overlay (dev, staging, production)

# 3. ConfigMap/Secret Generator 활용
# - 하드코딩 대신 Generator 사용
# - 민감 정보는 외부 관리

# 4. Patch는 작고 명확하게
# - 큰 Patch 대신 여러 작은 Patch
# - 용도별로 파일 분리

# 5. 버전 관리
# - Git으로 모든 변경 추적
# - 의미 있는 커밋 메시지

# 6. 검증 자동화
# - CI/CD에 kustomize build 통합
# - Dry-run으로 사전 검증
```

### 3. 유용한 도구

```bash
# yq - YAML 프로세서
brew install yq

# kustomize (최신 버전)
curl -s "https://raw.githubusercontent.com/kubernetes-sigs/kustomize/master/hack/install_kustomize.sh" | bash

# kubeval - Kubernetes YAML 검증
brew install kubeval

# kustomize-diff - 변경사항 비교
go install github.com/weaveworks/kustomize-diff@latest
```

### 4. 환경 변수 활용

```bash
# 환경별 빌드
export ENVIRONMENT=production
kubectl kustomize overlays/$ENVIRONMENT/

# 동적 이미지 태그
export IMAGE_TAG=v2.0.0
kubectl kustomize overlays/production/ | \
  sed "s|image: myapp:.*|image: myapp:$IMAGE_TAG|" | \
  kubectl apply -f -
```

---

## 일반적인 에러 메시지와 해결 방법

| 에러 메시지 | 원인 | 해결 방법 |
|------------|------|----------|
| `unable to find one of 'kustomization.yaml'` | 파일이 없거나 경로가 잘못됨 | 파일 존재 및 경로 확인 |
| `no matches for kind "XXX"` | API 버전 불일치 | apiVersion 확인 |
| `accumulating resources: accumulation err` | 리소스 파일을 찾을 수 없음 | resources 경로 확인 |
| `name not found` | Patch 대상 리소스 불일치 | 리소스 이름 매칭 확인 |
| `failed to find unique target for patch` | 여러 리소스가 매칭됨 | 더 구체적인 selector 사용 |
| `invalid path` | JSON Patch 경로 오류 | 경로 구조 확인 |
| `cyclic dependency` | 순환 참조 발생 | 의존성 구조 재검토 |

---

## 결론

Kustomize 트러블슈팅의 핵심:

1. **단순함 유지**: 템플릿 없이 순수 YAML 사용
2. **계층 구조**: Base와 Overlay의 명확한 분리
3. **점진적 접근**: 작은 변경부터 시작
4. **검증 우선**: 항상 dry-run으로 테스트
5. **Git 활용**: 버전 관리로 변경 추적

Kustomize는 간단하지만 강력한 도구입니다. 복잡한 템플릿 없이도 효과적으로 Kubernetes 리소스를 관리할 수 있습니다! 🔧
