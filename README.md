# Enterprise Progressive Delivery for Kubernetes — Canary, Blue/Green, A/B Shadow Deployments for MLOps & DevOps



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

- **Canary Deployment** — deploy new version to a small percentage of traffic, observe metrics, incrementally increase traffic until 100% or rollback to the old model/app version.  
- **Blue/Green Deployment** — run two full environments (Blue = current, Green = new). Switch traffic from Blue → Green atomically (DNS or load balancer).  
- **A/B Testing** — route distinct cohorts of real users to different versions to measure business KPIs (CTR, conversion).  
- **Shadow Testing** — mirror real production traffic to the new version without serving the new version's responses to users; used to validate behavior under real load.  
- **Automated Rollback** — configured alerts or analysis results trigger automatic rollback to the previous safe release.  
- **Analysis-driven Promotion** — use observability (Prometheus metrics, model quality logs) to decide whether to proceed with rollout steps.


### Canary Deployment

| Feature | Description |
|---------|-------------|
| **Goal** | Safely introduce new version gradually |
| **Traffic** | Starts small (1%, 5%, 20%, 50%...) |
| **Monitoring** | Latency, errors, throughput, accuracy for ML |
| **Rollback** | Automatic or manual to old version |
| **Tools** | Argo Rollouts, Istio, Linkerd, Seldon, KServe |


#### Example of Canary workflow:

(1). Old model v1 is running

(2). Deploy v2 but route only 10% traffic

(3). Monitor metrics (p95 latency, model accuracy, drift)

(4). If good → increase to 50% → 100%

(5). If bad → rollback instantly


###  Canary vs Blue-Green vs A/B Testing

| Method | Purpose | Traffic Split | Used When |
|--------|---------|---------------|-----------|
| **Canary Deployment** | Safe rollout | Gradual (1% → 10% → 50% → 100%) | New app/model version rollout |
| **Blue-Green Deployment** | Zero-downtime switch | 100% on Blue → 100% on Green | Infrastructure/app upgrades |
| **A/B Testing** | Experiment on real users | Parallel different versions | Optimize ML model behavior, UX |


<ins>Simple definition</ins>

- Blue-Green → Two full environments, switch all traffic at once

- Canary → Gradual rollout with monitoring

- A/B Testing → Compare models/versions for business metrics (CTR, conversion, accuracy)

### <ins>A/B Shadow Deployment</ins> 

##### It essentially is a combination of two techniques for validating new software versions or machine learning models in production: a shadow deployment, which runs a new version in parallel with the current one using real production traffic without impacting users, followed by (or in conjunction with) A/B testing, which exposes different user groups to different versions to measure real-world performance and user behavior. 

<ins>Shadow Deployment (The "Shadow" part)</ins>
##### In a shadow deployment, a new model or service (the "challenger") is deployed alongside the existing, live version (the "champion"). 

- Process: A copy of the real-time production requests is sent to both the champion and the challenger. However, only the champion's output is returned to the end-user; the challenger's output is logged for analysis.

- Purpose: This allows engineers to test the new version's performance, stability, and behavior under actual production load and conditions without any risk to the live user experience or system operations. It is ideal for "sanity checking" a new model or testing infrastructure changes.

- Advantage: Risk mitigation is the primary benefit, as potential errors, performance bottlenecks, or bugs in the new version do not affect live users

<ins>A/B Testing (The "A/B" part)</ins>

##### A/B testing is a controlled experiment where users are randomly divided into groups, each exposed to a different variation of a product or feature. 

-  Process: Users in Group A interact with the existing version (champion), while users in Group B interact with the new version (challenger). The responses from both are live and affect the user experience.
  
- Purpose: The goal is to compare specific business metrics (e.g., conversion rates, engagement, click-through rates) to determine which version performs better based on actual user behavior and feedback.
  
- Advantage: Provides statistically significant, empirical data on how a new feature or model impacts the user experience and business outcomes.

<ins>Combined A/B / Shadow deployment</ins>

##### This typically describes a comprehensive strategy where a new version first undergoes a low-risk shadow deployment for technical validation (performance, stability), and only after it proves stable is it then rolled out via A/B testing to a portion of users to measure real-world impact and user engagement. This combines the safety of shadow deployments with the empirical validation of A/B testing, making it a robust and low-risk method for continuous delivery and improvement, especially for complex AI/ML systems. 

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

## A. Canary Rollout using Argo Rollouts + Istio + Prometheus Analysis

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


### 3) <ins>Rollout resource (Argo Rollouts) — orchestrates canary and analysis </ins>


**rollout-canary.yaml**


```python
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: my-app-rollout
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
        # version label will be set by updated ReplicaSet
    spec:
      containers:
      - name: my-app
        image: your-registry/my-app:1.1.0  # new version image
        ports:
        - containerPort: 8080
  strategy:
    canary:
      canaryService: "my-app-svc"  # the service that Istio routes to
      stableService: "my-app-stable" # optional stable svc name
      trafficRouting:
        istio:
          virtualService:
            name: my-app-virtualservice
            port: 80
      steps:
      - setWeight: 10                # send 10% of traffic to new version
      - pause: {duration: 2m}        # wait for 2 minutes and run analysis
      - setWeight: 30                # increase to 30%
      - pause: {duration: 5m}
      - setWeight: 60                # 60%
      - pause: {duration: 5m}
      - setWeight: 100               # promote to 100%
```

#### Apply Rollout:

```python
kubectl apply -f rollout-canary.yaml
```

### 4) <ins>AnalysisTemplate (Argo Rollouts) with Prometheus queries </ins>

**analysistemplate-prom.yaml**

```python
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata:
  name: canary-analysis-template
spec:
  args:
  - name: service
    value: my-app-svc
  metrics:
  - name: request-error-rate
    # Query Prometheus for HTTP 5xx rate for canary vs baseline
    provider:
      prometheus:
        address: http://prometheus:9090
        query: |
          # ratio of 5xx errors in last 2 minutes over total requests
          sum(rate(http_requests_total{job="my-app",status=~"5..",instance=~".*"}[2m]))
          /
          sum(rate(http_requests_total{job="my-app",instance=~".*"}[2m]))
    threshold: 0.05  # fail if > 5% errors
    failureLimit: 1
  - name: latency-p95
    provider:
      prometheus:
        address: http://prometheus:9090
        query: |
          histogram_quantile(0.95, sum(rate(http_request_duration_seconds_bucket{job="my-app"}[2m])) by (le))
    # threshold means: fail if p95 > 1.5s
    threshold: 1.5
    failureLimit: 1
```

Now reference the analysis template in the Rollout steps by adding analysis blocks to the steps (example shortened):

```python

      steps:
      - setWeight: 10
      - pause:
          duration: 2m
          # run analysis after pause
          analysis:
            templates:
            - templateName: canary-analysis-template
```


When the analysis fails, Argo Rollouts marks the rollout as Failed and will not proceed to next step. You may configure automation to rollback.

### 5) <ins> Observability & Metrics exposition </ins>


#### Your app should export the following Prometheus-style metrics:

- http_requests_total{status="200",job="my-app",instance="<pod>"}

- http_request_duration_seconds_bucket{le="0.1",job="my-app"}

- ML-specific metrics (if model): model_prediction_latency_seconds, model_prediction_accuracy, model_input_drift_{feature}

You can implement /metrics endpoint with Prometheus client libraries (Python example below in ML metrics section).

---

## B. Blue/Green Deployment (Zero-downtime switch)

#### Goal: Deploy green environment in parallel with blue, perform final checks, then switch 100% traffic to green

### 1) <ins>  Two Deployments with distinct labels and services </ins>


**deployment-blue.yaml (current)**

```python

apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app-blue
  labels:
    app: my-app
    env: blue
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-app
      env: blue
  template:
    metadata:
      labels:
        app: my-app
        env: blue
    spec:
      containers:
      - name: my-app
        image: your-registry/my-app:1.0.0
```

deployment-green.yaml (new)

```python
# Same as above but env: green and image: 1.1.0
```

##### service-bluegreen.yaml — single service points to one label; swap label when done

```python
apiVersion: v1
kind: Service
metadata:
  name: my-app-service
spec:
  selector:
    app: my-app
    env: blue   # change to 'green' when switching
  ports:
  - port: 80
    targetPort: 8080
```

<ins>Switching strategy</ins>:

- Deploy green (not yet receiving traffic).

- Run smoke tests / integration tests against green pods (via port-forward or internal tests).

- If green healthy → update service selector to env: green (atomic switch, all traffic goes to green).

- If issues → keep blue or revert.

> [!NOTE] With a service mesh you can also perform a weighted switch with Istio for smoother transition.

---

## C. A/B Testing (Business-metric experiments)

#### Goal: Evaluate business metrics (e.g., conversion, CTR) by routing real users to different versions.

<ins>Approach</ins>:

- Define cohorts (user IDs hashed) and route by header or cookie.

- Use Istio VirtualService or specialized experimentation frameworks (e.g., Flagship, Optimizely) to route X% of traffic deterministically by user ID hashing.

#### Example Istio VirtualService snippet for A/B split by header:

```python
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: my-app-ab
spec:
  hosts:
  - "my-app.example.com"
  http:
  - match:
    - headers:
        x-user-group:
          exact: "group-a"
    route:
    - destination:
        host: my-app-svc
        subset: variantA
      weight: 100
  - match:
    - headers:
        x-user-group:
          exact: "group-b"
    route:
    - destination:
        host: my-app-svc
        subset: variantB
      weight: 100
  - route:
    - destination:
        host: my-app-svc
        subset: variantA
      weight: 50
    - destination:
        host: my-app-svc
        subset: variantB
      weight: 50
```


<ins>Measuring business metrics</ins>:

- Emit custom metrics (Prometheus counters) for events: purchase_total, signup_total, ad_click_total labeled by variant.

- Build Grafana panels and use statistical tests (e.g., chi-square) to determine significance.

---

## D. Shadow Testing (Realtime validation without impacting users)

#### Goal: Mirror production traffic to a new model or app instance but do not use the response in serving real users.

<ins>Approach</ins>:

- Service Mesh or custom proxy duplicates incoming requests and sends copies to shadow endpoint.

- The shadow instance logs predictions and returns its output to monitoring but does not alter response path.

#### Istio EnvoyFilter or custom sidecar can mirror traffic. Example VirtualService mirror spec:

```python
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: my-app-shadow
spec:
  hosts:
  - "my-app.example.com"
  http:
  - route:
    - destination:
        host: my-app-svc
        subset: stable
      weight: 100
    mirror:
      host: my-app-svc
      subset: candidate
```

> [!Important]
>  Shadow requests should be asynchronous or non-blocking to avoid latency impact.























































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
