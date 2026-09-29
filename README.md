# Kubernetes Prometheus & Grafana Monitoring

Kubernetes 환경에서 **Prometheus와 Grafana를 기반으로 모니터링 및 Alerting 환경을 구축한 프로젝트**입니다.

Kubernetes의 CPU / Memory 및 Pod 상태를 Prometheus로 수집하고 Grafana를 통해 시각화했습니다.

또한 Kubernetes `PrometheusRule`을 YAML Manifest로 작성하여 GitHub에서 Alert 설정을 코드로 관리하고, Pod CPU 사용량이 임계치를 초과하면 **Grafana 및 Telegram을 통해 알림을 확인**할 수 있도록 구성했습니다.

---

## 1. Project Overview

### 목표

* Kubernetes 클러스터 리소스 모니터링
* Prometheus 기반 Metric 수집
* Grafana 기반 Kubernetes Dashboard 구성
* PrometheusRule을 이용한 Alert Rule 구성
* Alert 설정을 YAML로 관리하고 GitHub에서 버전 관리
* Pod CPU 임계치 초과 감지
* Telegram Notification 연동
* CPU 부하 테스트를 통한 Alert Firing / Recovery 검증

---

## 2. Architecture

```text
                         GitHub
                           │
                           │ PrometheusRule
                           ▼
                    ┌───────────────┐
                    │   Kubernetes  │
                    │               │
                    │ PrometheusRule│
                    └───────┬───────┘
                            │
                            ▼
                    ┌───────────────┐
                    │  Prometheus   │
                    │               │
                    │ Metric 수집    │
                    │ Alert 평가     │
                    └───────┬───────┘
                            │
                            ▼
                    ┌───────────────┐
                    │    Grafana    │
                    │               │
                    │ Dashboard     │
                    │ Monitoring    │
                    └───────┬───────┘
                            │
                            ▼
                    ┌───────────────┐
                    │   Telegram    │
                    │ Notification  │
                    └───────────────┘
```

### Monitoring Flow

```text
Kubernetes
    ↓
Prometheus
    ↓
Grafana
    ↓
Alert Rule
    ↓
Telegram
```

### Configuration Management

```text
GitHub
    ↓
cpu-alert.yaml
    ↓
kubectl apply
    ↓
PrometheusRule
    ↓
Prometheus
```

---

## 3. Environment

| Component               | Version / Environment      |
| ----------------------- | -------------------------- |
| OS                      | Windows                    |
| Kubernetes              | v1.37.0                    |
| Kubernetes Distribution | Kind                       |
| Cluster                 | devops-cluster             |
| Nodes                   | 1 Control Plane + 2 Worker |
| Helm                    | v4.3.0                     |
| Monitoring Stack        | kube-prometheus-stack      |
| Prometheus              | v0.94.1                    |
| Grafana                 | kube-prometheus-stack      |
| Alertmanager            | kube-prometheus-stack      |
| Notification            | Telegram                   |

---

## 4. Kubernetes Cluster

Kind를 이용하여 Kubernetes 테스트 클러스터를 구성했습니다.

```text
devops-cluster
├── control-plane
├── worker
└── worker2
```

Node 상태 확인:

```bash
kubectl get nodes
```

모든 Node가 `Ready` 상태인 것을 확인했습니다.

### Kubernetes Cluster

📸 **Screenshot**

> <img width="578" height="83" alt="image" src="https://github.com/user-attachments/assets/86b7f25a-cb86-46b1-843f-27f3992e075b" />


---

## 5. Monitoring Stack

`kube-prometheus-stack` Helm Chart를 이용하여 Prometheus와 Grafana를 포함한 Kubernetes Monitoring Stack을 구성했습니다.

### Helm Repository

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
```

### Namespace

```bash
kubectl create namespace monitoring
```

### Installation

```bash
helm install monitoring prometheus-community/kube-prometheus-stack \
  --namespace monitoring
```

설치된 주요 구성 요소:

```text
kube-prometheus-stack
├── Prometheus
├── Grafana
├── Alertmanager
├── kube-prometheus-operator
├── kube-state-metrics
└── node-exporter
```

Pod 상태 확인:

```bash
kubectl get pods -n monitoring
```

### Monitoring Stack

📸 **Screenshot**

> `kubectl get pods -n monitoring` 결과
> <img width="726" height="174" alt="image" src="https://github.com/user-attachments/assets/9deb2419-4ca4-4e17-bd4b-e625c95faeca" />


---

## 6. Grafana Dashboard

Grafana를 Prometheus와 연동하여 Kubernetes 리소스를 시각화했습니다.

Grafana 접속:

```bash
kubectl port-forward -n monitoring svc/monitoring-grafana 3000:80
```

```text
http://localhost:3000
```

확인한 주요 Metric:

* Cluster CPU
* Cluster Memory
* Pod CPU
* Pod Memory
* Node Resource
* Kubernetes Object 상태

### Grafana Kubernetes Dashboard

📸 **Screenshot**

> Grafana `Kubernetes → Compute Resources → Cluster` 화면
> CPU / Memory 그래프가 보이는 화면

---

## 7. Prometheus Metric

Prometheus에서 Kubernetes Metric을 직접 조회하여 정상적으로 데이터가 수집되는 것을 확인했습니다.

Prometheus 접속:

```bash
kubectl port-forward -n monitoring \
  svc/monitoring-kube-prometheus-prometheus 9090:9090
```

```text
http://localhost:9090
```

### Pod 상태 Metric

```promql
kube_pod_status_phase
```

### Running Pod Count

```promql
count(
  kube_pod_status_phase{
    namespace="default",
    phase="Running",
    pod=~"my-devops-app-.*"
  }
)
```

Prometheus Query를 통해 Kubernetes의 실제 Pod 상태가 Metric으로 수집되는 것을 확인했습니다.

### Prometheus Query

📸 **Screenshot**

> Prometheus에서 PromQL Query를 실행한 화면

---

# 8. Alert Rule as Code

이번 프로젝트에서는 Grafana UI에서만 Alert를 설정하는 것이 아니라, Kubernetes `PrometheusRule` 리소스를 YAML로 작성하여 **Alert 설정을 코드로 관리**했습니다.

프로젝트 구조:

```text
kubernetes-prometheus-grafana-monitoring/
│
├── README.md
│
├── helm/
│   └── values.yaml
│
├── manifests/
│   └── cpu-alert.yaml
│
└── docs/
    └── screenshots/
```

### PrometheusRule

`manifests/cpu-alert.yaml`

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: kubernetes-cpu-alert
  namespace: monitoring
  labels:
    release: monitoring
spec:
  groups:
    - name: kubernetes-monitoring
      rules:
        - alert: KubernetesCPUHigh
          expr: |
            100 * (
              sum by (pod) (
                rate(
                  container_cpu_usage_seconds_total{
                    namespace="default",
                    container!="POD",
                    container!=""
                  }[5m]
                )
              )
            ) > 30
          for: 0m
          labels:
            severity: warning
          annotations:
            summary: "Kubernetes Pod CPU usage is high"
            description: "Pod {{ $labels.pod }} CPU usage is above 30%."
```

### 적용

```bash
kubectl apply -f manifests/cpu-alert.yaml
```

### PrometheusRule 확인

```bash
kubectl get prometheusrule -n monitoring
```

결과:

```text
kubernetes-cpu-alert
```

### Prometheus Alert 확인

Prometheus:

```text
http://localhost:9090/alerts
```

에서 `KubernetesCPUHigh` Rule이 정상적으로 등록된 것을 확인했습니다.

### Alert Rule

📸 **Screenshot**

> Prometheus `/alerts` 화면에서 `KubernetesCPUHigh`가 `Inactive` 상태로 표시된 화면

---

# 9. CPU Load Test

Alert 동작을 검증하기 위해 CPU 부하 테스트용 Pod를 생성했습니다.

```bash
kubectl run cpu-test \
  --image=busybox \
  --restart=Never \
  -- /bin/sh -c "while true; do echo test > /dev/null; done"
```

Pod 상태 확인:

```bash
kubectl get pod cpu-test
```

CPU 부하가 발생하면 Prometheus가 Metric을 수집하고 `KubernetesCPUHigh` Alert Rule이 조건을 평가합니다.

---

# 10. Alert Firing Test

CPU 부하 테스트 결과 `KubernetesCPUHigh` Alert가 실제로 `Firing` 상태로 변경되는 것을 확인했습니다.

테스트 과정:

```text
CPU Load Pod
     ↓
CPU Usage 증가
     ↓
Prometheus Metric 수집
     ↓
PrometheusRule 평가
     ↓
CPU > 30%
     ↓
KubernetesCPUHigh
     ↓
Firing
```

실제 테스트에서는 `cpu-test` Pod의 CPU 사용량이 약 **65%**까지 상승하면서 Alert가 발생했습니다.

### Prometheus Firing

📸 **Screenshot**

> Prometheus `/alerts`에서 `KubernetesCPUHigh`가 `Firing` 상태인 화면

### Grafana Alert

📸 **Screenshot**

> Grafana Alerting 화면에서 Alert가 `Firing` 상태인 화면

---

# 11. Telegram Notification

Grafana Contact Point에 Telegram Bot을 연결하여 Alert Notification을 구성했습니다.

```text
Prometheus Alert
       ↓
Grafana Alerting
       ↓
Contact Point
       ↓
Telegram Bot
       ↓
Telegram
```

CPU Alert 발생 시 Telegram으로 알림이 전달되는 것을 확인했습니다.

### Telegram Alert

📸 **Screenshot**

> 실제 Telegram에서 수신한 CPU Alert 화면

⚠️ **주의**

Bot Token과 같은 인증 정보는 GitHub Repository에 저장하지 않습니다.

---

# 12. Alert Recovery

CPU 부하 테스트 종료 후 테스트 Pod를 삭제했습니다.

```bash
kubectl delete pod cpu-test
```

CPU 사용량이 정상 상태로 돌아오면서 Alert도 정상 상태로 복구되는 것을 확인했습니다.

```text
Firing
   ↓
CPU Load 종료
   ↓
CPU Usage 감소
   ↓
Normal / Inactive
```

이를 통해 다음 과정을 검증했습니다.

```text
장애 상황 발생
    ↓
Metric 수집
    ↓
Threshold 초과 감지
    ↓
Alert Firing
    ↓
Telegram Notification
    ↓
장애 상황 해소
    ↓
Alert Recovery
```

### Alert Recovery

📸 **Screenshot**

> Prometheus 또는 Grafana에서 Alert가 `Inactive / Normal` 상태로 복구된 화면

---

# 13. GitHub Repository

본 프로젝트에서는 GitHub를 이용하여 Monitoring 설정 파일과 Alert Rule을 버전 관리했습니다.

```text
GitHub
│
├── helm/
│   └── values.yaml
│
├── manifests/
│   └── cpu-alert.yaml
│
└── README.md
```

GitHub Actions를 통한 CI/CD는 본 프로젝트의 범위에 포함하지 않았으며, **Monitoring Configuration과 Alert Rule의 코드 관리 및 버전 관리**에 GitHub를 사용했습니다.

---

# 14. Project Test Result

| Test                         | Result |
| ---------------------------- | ------ |
| Kubernetes Node Monitoring   | ✅      |
| Prometheus Metric Collection | ✅      |
| Grafana Dashboard            | ✅      |
| PrometheusRule 등록            | ✅      |
| CPU Threshold Alert          | ✅      |
| Alert Firing                 | ✅      |
| Telegram Notification        | ✅      |
| Alert Recovery               | ✅      |
| Alert Configuration Git 관리   | ✅      |

---

# 15. What I Learned

### Kubernetes

* Kubernetes Node / Pod 상태 확인
* Pod Resource 상태 확인
* Kubernetes Metric 구조 이해
* Prometheus Operator 기반 Monitoring 구조 이해

### Prometheus

* Kubernetes Metric 수집 구조
* PromQL 작성
* Pod CPU Metric 조회
* PrometheusRule을 이용한 Alert 구성
* Alert 상태 확인

### Grafana

* Prometheus Data Source 연동
* Kubernetes Dashboard 활용
* Metric Visualization
* Alerting 구성
* Telegram Contact Point 연동

### Git / GitHub

* Monitoring 설정 파일의 Git 관리
* PrometheusRule YAML 버전 관리
* Kubernetes Manifest 관리
* Infrastructure Configuration을 코드로 관리하는 방식 이해

---

# 16. Project Structure

```text
kubernetes-prometheus-grafana-monitoring/
│
├── README.md
│
├── helm/
│   └── values.yaml
│
├── manifests/
│   └── cpu-alert.yaml
│
└── docs/
    └── screenshots/
```

---

# 17. Related DevOps Projects

## Project 1 — Docker CI/CD Pipeline

```text
GitHub
   ↓
GitHub Actions
   ↓
Docker Build
   ↓
Docker Hub
   ↓
Remote Server Deployment
```

**Skills**

```text
Docker
GitHub Actions
CI/CD
Docker Hub
SSH
Linux
```

---

## Project 2 — Kubernetes GitOps

```text
GitHub
   ↓
ArgoCD
   ↓
Kubernetes
   ↓
Application Deployment
```

**Skills**

```text
Kubernetes
ArgoCD
GitOps
Deployment
Service
Replica Management
```

---

## Project 3 — Kubernetes Monitoring

```text
Kubernetes
   ↓
Prometheus
   ↓
Grafana
   ↓
Alerting
   ↓
Telegram
```

**Skills**

```text
Kubernetes
Helm
Prometheus
PromQL
Grafana
PrometheusRule
Alerting
Telegram
Git
GitHub
```

---

# 18. DevOps Portfolio Architecture

세 프로젝트를 연결하면 다음과 같은 전체 DevOps 환경을 구성했습니다.

```text
                         GitHub
                           │
            ┌──────────────┼──────────────┐
            │              │              │
            ▼              ▼              ▼
      GitHub Actions     ArgoCD       Monitoring
            │              │          Configuration
            ▼              ▼              │
       Docker Hub      Kubernetes ◀──────┘
                           │
                           ▼
                      Prometheus
                           │
                           ▼
                        Grafana
                           │
                           ▼
                       Telegram
```

### End-to-End

```text
Source Code
    ↓
GitHub
    ↓
CI/CD
    ↓
Docker Image
    ↓
Kubernetes
    ↓
ArgoCD GitOps
    ↓
Prometheus Monitoring
    ↓
Grafana Visualization
    ↓
Alert Rule
    ↓
Telegram Notification
```

---

# 19. Key Skills

```text
Kubernetes
Docker
Helm
Prometheus
PromQL
Grafana
PrometheusRule
Alerting
Telegram
Git
GitHub
GitOps
ArgoCD
GitHub Actions
CI/CD
Linux
Infrastructure Monitoring
```
