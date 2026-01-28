# Kubeflow 전체 삭제 정리 가이드

> 이 문서는 **이미 설치된 Kubeflow 환경을 Kubernetes 클러스터에서 완전히 제거**하기 위해 실제로 수행한 과정을 정리한 것이다.  
> 단순 네임스페이스 삭제가 아니라, **CRD / Webhook / APIService / ClusterRole까지 포함한 완전 제거**를 목표로 한다.

---

## 1. 기본 상황

- Kubernetes: NKS (네이버 클라우드 Kubernetes)
- 설치 이력:
  - Kubeflow
  - KServe
  - Katib
  - Knative
  - KEDA
- 목표:
  - ML 관련 컴포넌트 **재설치 예정 없음**
  - 클러스터를 **완전히 깨끗한 상태**로 정리

---

## 2. 네임스페이스 삭제

먼저 Kubeflow 및 연관 네임스페이스를 삭제한다.

```bash
kubectl delete ns kubeflow
kubectl delete ns kserve
kubectl delete ns katib
kubectl delete ns knative-serving
kubectl delete ns keda
kubectl delete ns profiles-system
```

### 2.1 Terminating 상태에서 멈출 경우

finalizer가 남아있으면 namespace가 삭제되지 않는다.

```bash
kubectl get ns <namespace> -o json > ns.json
```

`spec.finalizers`를 제거 후 강제 반영:

```bash
kubectl replace --raw "/api/v1/namespaces/<namespace>/finalize" -f ns.json
```

> ⚠ Windows PowerShell에서는 UTF-16 인코딩 문제 발생 가능
> → 이 경우 `kubectl patch` 방식 사용 권장

```bash
kubectl patch namespace <namespace> -p '{"spec":{"finalizers":[]}}' --type=merge
```

---

## 3. CRD(CustomResourceDefinition) 정리

Kubeflow 계열 CRD 확인:

```bash
kubectl get crd | findstr -i "kubeflow kserve katib notebook pipeline spark tensorboard"
```

존재 시 삭제:

```bash
kubectl delete crd <crd-name>
```

> ⚠ CRD는 **네임스페이스와 무관한 클러스터 리소스**이므로 반드시 정리해야 한다.

---

## 4. Webhook Configuration 삭제

### 4.1 Validating Webhook

```bash
kubectl get validatingwebhookconfigurations | findstr -i "kubeflow kserve katib knative"
```

```bash
kubectl delete validatingwebhookconfiguration \
  clusterservingruntime.serving.kserve.io \
  inferencegraph.serving.kserve.io \
  inferenceservice.serving.kserve.io \
  katib.kubeflow.org \
  localmodelcache.serving.kserve.io \
  servingruntime.serving.kserve.io \
  trainedmodel.serving.kserve.io \
  validator.training-operator.kubeflow.org
```

### 4.2 Mutating Webhook

```bash
kubectl get mutatingwebhookconfigurations | findstr -i "kubeflow kserve katib knative"
```

```bash
kubectl delete mutatingwebhookconfiguration \
  cache-webhook-kubeflow \
  inferenceservice.serving.kserve.io \
  katib.kubeflow.org \
  sinkbindings.webhook.sources.knative.dev
```

---

## 5. APIService 정리 (Knative 잔존 문제)

Knative 관련 APIService는 삭제해도 **자동으로 다시 생성**될 수 있다.
이는 backing controller가 아직 남아있기 때문이며, **관련 컴포넌트가 완전히 제거되면 더 이상 재생성되지 않는다**.

확인:

```bash
kubectl get apiservice | findstr -i "knative"
```

삭제:

```bash
kubectl delete apiservice v1.serving.knative.dev
kubectl delete apiservice v1beta1.serving.knative.dev
kubectl delete apiservice v1alpha1.autoscaling.internal.knative.dev
kubectl delete apiservice v1alpha1.caching.internal.knative.dev
kubectl delete apiservice v1alpha1.networking.internal.knative.dev
```

---

## 6. ClusterRole / ClusterRoleBinding 정리

확인:

```bash
kubectl get clusterrole | findstr -i "kubeflow kserve katib"
kubectl get clusterrolebinding | findstr -i "kubeflow kserve katib"
```

삭제:

```bash
kubectl delete clusterrole kubeflow-admin kubeflow-edit kubeflow-view
kubectl delete clusterrolebinding kubeflow-user-full-access
```

---

## 7. 최종 검증 (완전 삭제 확인)

아래 명령에서 **아무 출력도 나오지 않으면 성공**:

```bash
kubectl get ns | findstr -i "kubeflow knative kserve katib profile"
kubectl get crd | findstr -i "kubeflow kserve katib"
kubectl get apiservice | findstr -i "kubeflow knative kserve katib"
kubectl get validatingwebhookconfigurations | findstr -i "kubeflow knative kserve"
kubectl get mutatingwebhookconfigurations | findstr -i "kubeflow knative kserve"
```

---

## 8. 최종 상태 요약

- Kubeflow 및 ML 스택 **완전 제거 완료**
- 클러스터는 일반 워크로드 전용으로 사용 가능
- 향후 Kubeflow 재설치 시 충돌 위험 없음

> 이 상태는 **신규 클러스터와 거의 동일한 수준**이다.

---

## 9. 운영자 메모

- Kubeflow는 "삭제"보다 "해체"에 가깝다
- Namespace 삭제만으로는 절대 끝나지 않는다
- CRD / Webhook / APIService를 남기면 반드시 문제를 만든다

---

✅ **정리 완료**

