# Enterprise Progressive Delivery for Kubernetes — Canary, Blue/Green, A/B, Shadow Deployments for DevOps & MLOps



> A complete, production-oriented guide decsribing **progressive delivery** of applications and ML models on Kubernetes.  
> Covers Canary rollouts, Blue/Green swaps, A/B testing, shadow testing, monitoring-driven promotion/rollback, and automated CI/CD + GitOps workflows. Includes **detailed, well-commented code examples** (Kubernetes, Istio, Argo Rollouts, Prometheus rules, GitHub Actions, and example model validation scripts).

---

## Table of contents

1. [Introduction & Goals](#introduction--goals)  
2. [Key Concepts and Patterns](#key-concepts-and-patterns)  
3. [Architecture & Components](#architecture--components)  
4. [End-to-End Workflows — High Level](#end-to-end-workflows--high-level)  
5. [Prerequisites & Tooling](#prerequisites--tooling)  
6. [Detailed Examples (Code + Explanation)](#detailed-examples-code--explanation)  
   - [A. Canary Rollout using Argo Rollouts + Istio + Prometheus Analysis](#a-canary-rollout-using-argo-rollouts--istio--prometheus-analysis)  
   - [B. Blue/Green Deployment (Zero-downtime switch)](#b-bluegreen-deployment-zero-downtime-switch)  
   - [C. A/B Testing (Business-metric experiments)](#c-ab-testing-business-metric-experiments)  
   - [D. Shadow Testing (Realtime validation without impacting users)](#d-shadow-testing-realtime-validation-without-impact)  
7. [Monitoring, Alerting & Automated Rollback](#monitoring-alerting--automated-rollback)  
8. [CI/CD & GitOps Integration Examples](#cicd--gitops-integration-examples)  
9. [ML-specific Validation & Metrics (examples)](#ml-specific-validation--metrics-examples)  
10. [Best Practices & Operational Checklist](#best-practices--operational-checklist)  
11. [Appendices: Useful Snippets & Tools](#appendices-useful-snippets--tools)  

---

## Introduction & Goals

Progressive delivery allows teams to deploy new application versions or ML models **gradually** to production, observe their behavior, and either promote or roll back based on run-time signals (latency, errors, and model-specific metrics like accuracy or data drift). This approach reduces risk and enables faster iteration.

This README explains *how* progressive delivery is implemented in Kubernetes-based enterprise environments and provides concrete examples you can copy into your repos.

---

## Key Concepts and Patterns

- **Canary Deployment** — deploy new version to a small percentage of traffic, observe metrics, incrementally increase traffic until 100% or rollback.  
- **Blue/Green Deployment** — run two full environments (Blue = current, Green = new). Switch traffic from Blue → Green atomically (DNS or load balancer).  
- **A/B Testing** — route distinct cohorts of real users to different versions to measure business KPIs (CTR, conversion).  
- **Shadow Testing** — mirror real production traffic to the new version without serving the new version's responses to users; used to validate behavior under real load.  
- **Automated Rollback** — configured alerts or analysis results trigger automatic rollback to the previous safe release.  
- **Analysis-driven Promotion** — use observability (Prometheus metrics, model quality logs) to decide whether to proceed with rollout steps.

---

## Architecture & Components

### Typical components in a progressive delivery pipeline:

- **Kubernetes cluster(s)** — runtime for pods
- **Ingress / Service Mesh** — Istio, Linkerd, or Kuma for traffic shaping/splitting
- **Deployment controller** — Argo Rollouts (for Canary/Blue-Green orchestration) or native k8s + custom controllers
- **Observability** — Prometheus (metrics), Grafana (dashboards), Loki (logs)
- **Model monitoring** — custom metrics endpoints, Prometheus exporters, or model-monitoring platforms (e.g., WhyLabs, Evidently)
- **CI/CD** — GitHub Actions / Jenkins / GitLab CI for building images and updating manifests
- **GitOps** — ArgoCD or Flux for auto-syncing Git → cluster
- **Alerting** — Alertmanager for Prometheus to notify and optionally trigger webhook-based rollbacks




```python

flowchart LR
  A[Developer Push] -->|CI Build| B[Container Registry]
  B --> C[Git (Helm / manifests)]
  C -->|ArgoCD sync| D(Kubernetes cluster)
  C -->|Triggers| E[Argo Rollouts]
  E -->|Traffic split| F[Istio/Service Mesh]
  F --> G{Pods: v1 & v2}
  G --> H[User Traffic]
  G --> I[Metrics Prometheus]
  I --> J[Alertmanager / Analysis]
  J -->|Rollback| E
```

---

### End-to-End Workflows — High Level

- Code & Model push → CI builds image + stores in registry.

- CI updates Helm/manifest or pushes new chart / image tag to Git.

- ArgoCD (or other GitOps) deploys manifests (Rollouts, Services).

- Argo Rollouts + Istio handle traffic shifting (e.g., 10% → 30% → 100%).

- Prometheus & Analysis evaluate metrics at each step.

- If analysis passes → promote; if fails → rollback.

- Observability & auditing record the rollout.

---

### Prerequisites & Tooling

- Kubernetes cluster (minikube, k3s, EKS, GKE, AKS)

- kubectl configured to the cluster

- Istio (or alternative service mesh) installed (for traffic splitting) — optional but recommended for fine-grained traffic control

- Argo Rollouts controller installed: kubectl apply -f https://github.com/argoproj/argo-rollouts/releases/latest/download/install.yaml

- Prometheus (and Alertmanager) for metrics & alerts

- ArgoCD (optional) for GitOps sync

- Container registry (Docker Hub, ECR, GCR)

- Optional ML model registry (MLflow, S3, Artifact Registry)

---

## Detailed Examples (Code + Explanation)

### A. Canary Rollout using Argo Rollouts + Istio + Prometheus Analysis

- Overview: Use Argo Rollouts to orchesrate canary steps and Istio to split traffic. Use a Prometheus-based AnalysisTemplate to decide whether to continue.

### 1) <ins>Deploy v1 (current) Deployment + Service</ins>

**deployment-v1.yaml**

```python
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app-v1
  labels:
    app: my-app
    version: v1
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-app
      version: v1
  template:
    metadata:
      labels:
        app: my-app
        version: v1
    spec:
      containers:
      - name: my-app
        image: your-registry/my-app:1.0.0
        ports:
        - containerPort: 8080
        # Expose metrics endpoint for Prometheus (e.g., /metrics)

```

**service.yaml**

```python

apiVersion: v1
kind: Service
metadata:
  name: my-app-svc
spec:
  selector:
    app: my-app
  ports:
  - name: http
    port: 80
    targetPort: 8080
```

### Apply:

```python
kubectl apply -f deployment-v1.yaml
kubectl apply -f service.yaml
```


### 2) <ins>Install Argo Rollouts and configure Istio virtualservice (if using Istio) </ins>


(Assume Istio is installed and a Gateway exists.)

**istio-destinationrule.yaml — ensure subsets**

```python
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: my-app-destination
spec:
  host: my-app-svc
  subsets:
  - name: v1
    labels:
      version: v1
  - name: v2
    labels:
      version: v2
```

#### istio-virtualservice.yaml — virtual service that Argo Rollouts updates via TrafficSplit (Argo will set weight)

```python
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: my-app-virtualservice
spec:
  hosts:
  - "*"
  gateways:
  - my-gateway
  http:
  - route:
    - destination:
        host: my-app-svc
        subset: v1
      weight: 100
    - destination:
        host: my-app-svc
        subset: v2
      weight: 0
```

> [!NOTE]
> Argo Rollouts can be configured to update Istio VirtualService to change weights.
























































### Thank you for reading
---

### **AUTHOR'S BACKGROUND**
### Author's Name:  Emmanuel Oyekanlu
```
Skillset:   I have experience spanning several years in data science, developing scalable enterprise data pipelines,
enterprise solution architecture, architecting enterprise systems data and AI applications,
software and AI solution design and deployments, data engineering, high performance computing (GPU, CUDA), machine learning,
NLP, Agentic-AI and LLM applications as well as deploying scalable solutions (apps) on-prem and in the cloud.

I can be reached through: manuelbomi@yahoo.com

Website:  http://emmanueloyekanlu.com/
Publications:  https://scholar.google.com/citations?user=S-jTMfkAAAAJ&hl=en
LinkedIn:  https://www.linkedin.com/in/emmanuel-oyekanlu-6ba98616
Github:  https://github.com/manuelbomi

```
[![Icons](https://skillicons.dev/icons?i=aws,azure,gcp,scala,mongodb,redis,cassandra,kafka,anaconda,matlab,nodejs,django,py,c,anaconda,git,github,mysql,docker,kubernetes&theme=dark)](https://skillicons.dev)
