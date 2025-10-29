# End‑to‑End DevOps Data Pipeline (Kafka → Spark → Postgres → API → Dashboard)

A production‑grade project you can ship to GitHub to showcase 5+ years DevOps experience. It covers IaC, Kubernetes, CI/CD (GitHub Actions + ArgoCD), observability (Prometheus/Grafana/ELK), security, and scalable data streaming.

---

## 0) Prerequisites

* Cloud: **AWS** account with admin on a sandbox.
* Local tools: `awscli`, `kubectl`, `helm`, `terraform` ≥ 1.5, `docker`, `kind` (optional), `jq`, `kubectx`, `k9s` (optional).
* GitHub: a private repo; GitHub Environments (staging/prod) with required secrets.
* DNS: Route53 hosted zone (optional) for ingress hosts.

---

## 1) Repository structure

```
devops-data-pipeline/
├─ app/
│  ├─ api/                 # FastAPI (or Node.js) service
│  │  └─ app.py
│  ├─ frontend/            # React/Next.js dashboard (optional)
│  └─ jobs/                # Spark/Flink jobs
│     └─ job1.py
├─ docker/
│  ├─ api.Dockerfile
│  ├─ spark.Dockerfile
│  └─ kafka-connect.Dockerfile (optional)
├─ k8s/
│  ├─ namespaces.yaml
│  ├─ api/
│  │  ├─ deployment.yaml
│  │  ├─ service.yaml
│  │  └─ hpa.yaml
│  ├─ spark/
│  │  └─ deployment.yaml
│  ├─ kafka/
│  │  ├─ values.yaml
│  │  └─ kustomization.yaml
│  ├─ ingress/
│  │  └─ alb-ingress.yaml
│  └─ argocd/
│     ├─ project.yaml
│     └─ application-set.yaml
├─ monitoring/
│  ├─ kube-stack/          # kube-prometheus-stack values
│  │  └─ values.yaml
│  ├─ grafana/
│  │  └─ dashboards/
│  └─ alerting/
│     └─ alertmanager-config.yaml
├─ logging/
│  ├─ loki/values.yaml
│  └─ promtail/values.yaml
├─ terraform/
│  ├─ envs/
│  │  ├─ staging/
│  │  │  ├─ backend.tf
│  │  │  └─ terragrunt.hcl (optional)
│  │  └─ prod/
│  ├─ modules/
│  │  ├─ vpc/
│  │  ├─ eks/
│  │  ├─ rds/
│  │  └─ msk/              # Managed Kafka (or self‑managed on EC2)
│  ├─ main.tf
│  ├─ variables.tf
│  └─ outputs.tf
├─ ci-cd/
│  ├─ github/
│  │  └─ pipeline.yml
│  └─ argocd/
│     └─ bootstrap.sh
├─ scripts/
│  ├─ build_push.sh
│  ├─ port_forward.sh
│  └─ load_test.sh
└─ README.md
```

---

## 2) Minimum viable app

### FastAPI API (`app/api/app.py`)

```python
from fastapi import FastAPI
import os, psycopg2
app = FastAPI()

@app.get("/healthz")
def health():
    return {"status": "ok"}

@app.get("/metrics-sample")
def metrics():
    return {"records_processed": 123, "lag": 0}
```

### API Dockerfile (`docker/api.Dockerfile`)

```dockerfile
FROM python:3.11-slim
WORKDIR /app
COPY app/api/requirements.txt ./
RUN pip install -r requirements.txt
COPY app/api/ .
EXPOSE 8080
CMD ["uvicorn", "app:app", "--host", "0.0.0.0", "--port", "8080"]
```

### Spark job (`app/jobs/job1.py`)

```python
from pyspark.sql import SparkSession
spark = SparkSession.builder.appName("stream-job").getOrCreate()
# TODO: read from Kafka, transform, write to Postgres
print("Job ran")
```

---

## 3) Local dev (optional)

Use `docker-compose` for Kafka + Zookeeper + Postgres and run API locally.

`docker-compose.yml` (root):

```yaml
version: "3.9"
services:
  zookeeper:
    image: confluentinc/cp-zookeeper:7.6.1
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181
  kafka:
    image: confluentinc/cp-kafka:7.6.1
    depends_on: [zookeeper]
    environment:
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka:9092
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
    ports: ["9092:9092"]
  postgres:
    image: postgres:15
    environment:
      POSTGRES_PASSWORD: postgres
    ports: ["5432:5432"]
```

---

## 4) Terraform (AWS)

### Scope

* **VPC** (public/private subnets, NAT)
* **EKS** (managed node groups or Fargate)
* **RDS Postgres** (or Aurora)
* **MSK** (Managed Kafka) or self‑managed Kafka on EC2 (for cost, start with MSK)
* **ECR** repos
* **IAM**: IRSA roles for controllers (ALB Ingress, ExternalDNS, Cluster Autoscaler, etc.)

### Example module usage (`terraform/main.tf`)

```hcl
module "vpc" {
  source = "terraform-aws-modules/vpc/aws"
  name   = var.name
  cidr   = var.vpc_cidr
  azs    = var.azs
  public_subnets  = var.public_subnets
  private_subnets = var.private_subnets
}

module "eks" {
  source          = "terraform-aws-modules/eks/aws"
  cluster_name    = "${var.name}-eks"
  cluster_version = "1.29"
  vpc_id          = module.vpc.vpc_id
  subnet_ids      = module.vpc.private_subnets
}

module "rds" {
  source  = "terraform-aws-modules/rds/aws"
  engine  = "postgres"
  version = "15"
  # ... subnet group, sg, creds from Secrets Manager
}

module "msk" {
  source = "terraform-aws-modules/msk/aws"
  cluster_name = "${var.name}-msk"
  # ... brokers, subnets, security groups
}
```

**Commands**

```bash
cd terraform
terraform init
terraform apply -auto-approve
aws eks update-kubeconfig --name <cluster> --region <region>
```

---

## 5) Cluster add‑ons (Helm)

Install ALB Ingress Controller, ExternalDNS, Metrics Server, Prometheus, Grafana, Loki, Promtail:

```bash
helm repo add eks https://aws.github.io/eks-charts
helm upgrade --install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system --set clusterName=<cluster> --set serviceAccount.create=false \
  --set region=<region> --set vpcId=<vpc-id>

helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm upgrade --install kube-prom-stack prometheus-community/kube-prometheus-stack -n monitoring --create-namespace -f monitoring/kube-stack/values.yaml

helm repo add grafana https://grafana.github.io/helm-charts
helm upgrade --install loki grafana/loki -n logging --create-namespace -f logging/loki/values.yaml
helm upgrade --install promtail grafana/promtail -n logging -f logging/promtail/values.yaml
```

---

## 6) Build & push images (ECR)

```bash
aws ecr create-repository --repository-name data-pipeline/api || true
aws ecr get-login-password | docker login --username AWS --password-stdin <acct>.dkr.ecr.<region>.amazonaws.com
IMAGE_TAG=$(git rev-parse --short HEAD)
docker build -t <acct>.dkr.ecr.<region>.amazonaws.com/data-pipeline/api:$IMAGE_TAG -f docker/api.Dockerfile .
docker push <acct>.dkr.ecr.<region>.amazonaws.com/data-pipeline/api:$IMAGE_TAG
```

---

## 7) Kubernetes manifests

### Namespace

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: data
```

### API Deployment `k8s/api/deployment.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api
  namespace: data
spec:
  replicas: 2
  selector:
    matchLabels: {app: api}
  template:
    metadata:
      labels: {app: api}
    spec:
      containers:
      - name: api
        image: <acct>.dkr.ecr.<region>.amazonaws.com/data-pipeline/api:__TAG__
        ports: [{containerPort: 8080}]
        env:
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: app-secrets
              key: database_url
        readinessProbe:
          httpGet: {path: /healthz, port: 8080}
          initialDelaySeconds: 5
        livenessProbe:
          httpGet: {path: /healthz, port: 8080}
          initialDelaySeconds: 15
```

### Service `k8s/api/service.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: api
  namespace: data
spec:
  type: ClusterIP
  selector: {app: api}
  ports:
  - port: 80
    targetPort: 8080
```

### HPA `k8s/api/hpa.yaml`

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: api-hpa
  namespace: data
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: api
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 60
```

### Ingress (ALB) `k8s/ingress/alb-ingress.yaml`

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: api-alb
  namespace: data
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTP":80},{"HTTPS":443}]'
    alb.ingress.kubernetes.io/ssl-redirect: '443'
    alb.ingress.kubernetes.io/certificate-arn: arn:aws:acm:<region>:<acct>:certificate/<id>
spec:
  rules:
  - host: api.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: api
            port:
              number: 80
```

---

## 8) GitOps with ArgoCD

### Install

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

### Project & ApplicationSet (`k8s/argocd/`)

```yaml
# project.yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: data-platform
  namespace: argocd
spec:
  destinations:
  - namespace: data
    server: https://kubernetes.default.svc
  sourceRepos:
  - https://github.com/<you>/devops-data-pipeline
```

```yaml
# application-set.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: api
  namespace: argocd
spec:
  project: data-platform
  source:
    repoURL: https://github.com/<you>/devops-data-pipeline
    path: k8s/api
    targetRevision: main
  destination:
    server: https://kubernetes.default.svc
    namespace: data
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
    - CreateNamespace=true
```

---

## 9) CI/CD (GitHub Actions)

`ci-cd/github/pipeline.yml`

```yaml
name: ci-cd
on:
  push:
    branches: ["main"]
  pull_request:
    branches: ["main"]

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest
    permissions:
      id-token: write
      contents: read
    steps:
    - uses: actions/checkout@v4
    - uses: aws-actions/configure-aws-credentials@v4
      with:
        role-to-assume: ${{ secrets.AWS_ROLE_ARN }}
        aws-region: ${{ secrets.AWS_REGION }}
    - name: Login to ECR
      id: ecr
      uses: aws-actions/amazon-ecr-login@v2
    - name: Build
      run: |
        IMAGE=${{ steps.ecr.outputs.registry }}/data-pipeline/api:${{ github.sha }}
        docker build -f docker/api.Dockerfile -t $IMAGE .
        docker push $IMAGE
        echo "IMAGE=$IMAGE" >> $GITHUB_ENV
    - name: Update K8s manifests (set image tag)
      run: |
        sed -i "s#__TAG__#${{ github.sha }}#" k8s/api/deployment.yaml
        git diff | cat
    - name: Commit & push manifest change
      run: |
        git config user.email "bot@users.noreply.github.com"
        git config user.name "ci-bot"
        git add k8s/api/deployment.yaml
        git commit -m "ci: deploy ${{ github.sha }}" || echo "no changes"
        git push
```

> ArgoCD will sync automatically from the repo.

---

## 10) Observability

* **Prometheus**: scrape API `/metrics` (add `prometheus-fastapi-instrumentator` or custom metrics).
* **Grafana**: Dashboards for API latency, throughput, HPA metrics, Kafka consumer lag, Spark job success.
* **Alertmanager**: Alerts on high error rate, pod restarts, Kafka lag > threshold.

Example alert (YAML):

```yaml
groups:
- name: api.rules
  rules:
  - alert: High5xxRate
    expr: sum(rate(http_requests_total{status=~"5.."}[5m])) > 5
    for: 10m
    labels:
      severity: critical
    annotations:
      summary: High 5xx rate on API
```

---

## 11) Logging

* **Loki + Promtail** for cluster logs
* Optional: **OpenSearch** for full‑text log search

---

## 12) Secrets & Security

* Use **IRSA** for service accounts (no node credentials).
* Store DB passwords in **AWS Secrets Manager**; sync to K8s Secret via `external-secrets` operator.
* Enforce **RBAC** and namespace boundaries.
* Enable **TLS** on Ingress with ACM.

---

## 13) Scaling & Deployment strategies

* **HPA** for API & Spark workers
* Canary or Blue/Green via **Argo Rollouts** (optional):

```bash
helm repo add argo https://argoproj.github.io/argo-helm
helm install argo-rollouts argo/argo-rollouts -n argo-rollouts --create-namespace
```

---

## 14) Cost & Cleanup

* Prefer managed services (EKS + Fargate for small clusters) or scale node groups to zero in off hours.
* `terraform destroy` to clean resources.

---

## 15) Smoke tests & load tests

* Simple bash script hitting `/healthz` & `/metrics`.
* Add `k6` or `vegeta` tests; wire into CI and alert on regressions.

---

## 16) Milestone checklist (commit‑by‑commit)

1. Repo scaffold + minimal FastAPI app
2. Dockerize API; local compose works
3. Terraform VPC + EKS + ECR; `kubectl get nodes` OK
4. Helm: ALB controller + metrics server installed
5. Push first image to ECR
6. K8s manifests (namespace, deploy, svc, ingress) → API reachable via ALB
7. ArgoCD bootstrap + auto‑sync from repo
8. GitHub Actions builds/pushes and updates manifests
9. Monitoring stack + Grafana dashboard committed
10. Alerts firing to Slack/Teams
11. Add Spark job + Kafka producer/consumer
12. HPA validated with load test; canary rollout demo

---

## 17) What to put in README.md

* Architecture diagram
* Tech stack badges
* One‑click setup steps
* CI/CD + GitOps flow picture
* Screenshots: Grafana, ArgoCD, API endpoint, ALB DNS

---

## 18) Next steps

* Replace placeholders (`<acct>`, `<region>`, `api.example.com`).
* I can generate this repo skeleton with boilerplate files if you want.
