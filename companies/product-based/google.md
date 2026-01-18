# Google Interview Questions

Complete preparation guide for Google Software Engineer, SRE, and Cloud roles.

---

## Interview Process

| Round | Duration | Focus |
|-------|----------|-------|
| Phone Screen | 45 min | 1-2 coding problems |
| Onsite Round 1 | 45 min | Data Structures & Algorithms |
| Onsite Round 2 | 45 min | Data Structures & Algorithms |
| Onsite Round 3 | 45 min | System Design |
| Onsite Round 4 | 45 min | Behavioral (Googleyness) |
| Onsite Round 5 | 45 min | Coding or Domain-specific |

---

## What Google Looks For

| Attribute | What They Assess |
|-----------|------------------|
| **Coding** | Clean code, optimal solutions, edge cases |
| **Problem Solving** | Approach, clarifying questions, trade-offs |
| **System Design** | Scalability, reliability, real-world constraints |
| **Googleyness** | Collaboration, ambiguity handling, leadership |

---

## DevOps/SRE Interview Questions

### Question 1: Kubernetes - Debug CrashLoopBackOff

**Scenario:** A critical production pod is stuck in CrashLoopBackOff. Walk through your debugging process.

<details>
<summary><strong>View Answer</strong></summary>

**Step 1: Get pod status and events**
```bash
kubectl describe pod <pod-name> -n <namespace>
kubectl get events --sort-by='.lastTimestamp' -n <namespace>
```

**Step 2: Check container logs**
```bash
# Current logs
kubectl logs <pod-name> -n <namespace>

# Previous crashed container logs
kubectl logs <pod-name> -n <namespace> --previous
```

**Step 3: Common causes and fixes**

| Cause | Symptoms | Fix |
|-------|----------|-----|
| OOMKilled | Exit code 137 | Increase memory limits |
| Config error | Exit code 1 | Fix ConfigMap/Secrets |
| Missing dependency | Connection refused | Check service discovery |
| Liveness probe failing | Probe failed | Adjust probe thresholds |
| Image pull error | ImagePullBackOff | Fix registry credentials |

**Step 4: Debug interactively**
```bash
# Override entrypoint to keep container running
kubectl run debug --image=<image> --rm -it -- /bin/sh

# Or use ephemeral containers (K8s 1.23+)
kubectl debug <pod-name> -it --image=busybox
```

</details>

---

### Question 2: Design a Multi-Region Kubernetes Architecture

**Scenario:** Design a highly available, multi-region Kubernetes architecture for a financial services application with strict latency requirements (< 100ms) and 99.99% uptime SLA.

<details>
<summary><strong>View Answer</strong></summary>

**Architecture Overview:**
```
                        Global Load Balancer (Cloud LB / Cloudflare)
                                    │
                    ┌───────────────┼───────────────┐
                    │               │               │
                    ▼               ▼               ▼
              ┌─────────┐     ┌─────────┐     ┌─────────┐
              │ Region A │     │ Region B │     │ Region C │
              │ (US-East)│     │ (EU-West)│     │ (AP-South)│
              └────┬────┘     └────┬────┘     └────┬────┘
                   │               │               │
              ┌────┴────┐     ┌────┴────┐     ┌────┴────┐
              │   K8s   │     │   K8s   │     │   K8s   │
              │ Cluster │     │ Cluster │     │ Cluster │
              └────┬────┘     └────┬────┘     └────┬────┘
                   │               │               │
              ┌────┴────┐     ┌────┴────┐     ┌────┴────┐
              │  CockroachDB (Distributed)  │
              │  Multi-Region Replication   │
              └─────────────────────────────┘
```

**Key Components:**

1. **Global Load Balancing**
   - GeoDNS for latency-based routing
   - Health checks per region
   - Automatic failover

2. **Cluster Federation**
   - Consistent configuration across regions
   - GitOps with ArgoCD/Flux
   - Centralized monitoring

3. **Database Strategy**
   - CockroachDB or Spanner for global consistency
   - Regional read replicas for low latency
   - Conflict resolution for writes

4. **Disaster Recovery**
   - RPO: < 5 minutes
   - RTO: < 15 minutes
   - Cross-region backup with Velero

</details>

---

### Question 3: Docker - Optimize 2GB Image to Under 100MB

**Scenario:** Your Node.js microservice image is 2.1GB. Optimize it to under 100MB.

<details>
<summary><strong>View Answer</strong></summary>

**Problem: Current Dockerfile**
```dockerfile
FROM node:18
WORKDIR /app
COPY . .
RUN npm install
RUN npm run build
CMD ["node", "dist/server.js"]
```

**Solution: Multi-stage build**
```dockerfile
# Stage 1: Build
FROM node:18-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

# Stage 2: Production
FROM node:18-alpine AS production
RUN addgroup -g 1001 -S nodejs && adduser -S nodejs -u 1001
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production && npm cache clean --force
COPY --from=builder /app/dist ./dist
USER nodejs
EXPOSE 3000
CMD ["node", "dist/server.js"]
```

**Result:** 2.1GB → ~80MB (96% reduction)

**Why it works:**
- Build stage has dev dependencies (discarded)
- Production stage only has runtime code
- Alpine base is ~5MB vs ~900MB for full Node
- Non-root user adds security

</details>

---

### Question 4: GCP - Design Scalable API with Cloud Run

**Scenario:** Design a serverless API on GCP that handles 10K requests/second with automatic scaling and sub-100ms latency.

<details>
<summary><strong>View Answer</strong></summary>

**Architecture:**
```
Client → Cloud CDN → Cloud Load Balancer → Cloud Run → Cloud SQL/Firestore
                                             │
                                             ├── Cloud Memorystore (Redis)
                                             ├── Cloud Pub/Sub (async tasks)
                                             └── Cloud Storage (static assets)
```

**Cloud Run Configuration:**
```yaml
apiVersion: serving.knative.dev/v1
kind: Service
metadata:
  name: api-service
spec:
  template:
    metadata:
      annotations:
        autoscaling.knative.dev/minScale: "5"
        autoscaling.knative.dev/maxScale: "1000"
        run.googleapis.com/cpu-throttling: "false"
    spec:
      containerConcurrency: 80
      containers:
        - image: gcr.io/project/api:latest
          resources:
            limits:
              cpu: "2"
              memory: "2Gi"
```

**Key Optimizations:**
1. **Connection pooling** - Reuse DB connections
2. **Redis caching** - Cache frequent queries
3. **Minimum instances** - Avoid cold starts
4. **CPU always allocated** - No throttling during request processing

</details>

---

### Question 5: Terraform - Design Multi-Environment Infrastructure

**Scenario:** Design Terraform configuration for dev, staging, and prod environments with proper state management and minimal code duplication.

<details>
<summary><strong>View Answer</strong></summary>

**Directory Structure:**
```
terraform/
├── modules/
│   ├── vpc/
│   ├── gke/
│   ├── cloudsql/
│   └── monitoring/
├── environments/
│   ├── dev/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── terraform.tfvars
│   ├── staging/
│   └── prod/
└── backend.tf
```

**Module Usage:**
```hcl
# environments/prod/main.tf
module "vpc" {
  source = "../../modules/vpc"
  
  project_id   = var.project_id
  region       = var.region
  environment  = "prod"
  cidr_range   = "10.0.0.0/16"
}

module "gke" {
  source = "../../modules/gke"
  
  project_id     = var.project_id
  region         = var.region
  environment    = "prod"
  vpc_id         = module.vpc.vpc_id
  node_count     = 5  # More nodes for prod
  machine_type   = "n2-standard-4"
}
```

**Remote State:**
```hcl
# backend.tf
terraform {
  backend "gcs" {
    bucket = "company-terraform-state"
    prefix = "env/${terraform.workspace}"
  }
}
```

**Key Practices:**
- Modules for reusable components
- Workspaces or separate directories per environment
- Remote state with locking
- Variable files per environment

</details>

---

## System Design Questions

### Design YouTube

<details>
<summary><strong>View Answer</strong></summary>

**Requirements:**
- Upload videos
- Stream videos globally
- Handle 1B+ daily views
- Support 4K, adaptive bitrate

**High-Level Design:**
```
Upload Flow:
User → Upload Service → Object Storage → Transcoding Queue
                                              │
                                              ▼
                                    Transcoding Workers
                                              │
                                              ▼
                                    CDN Distribution

Watch Flow:
User → CDN (cache hit) → Video
User → CDN (cache miss) → Origin → Video → CDN → User
```

**Key Components:**
1. **Upload Service** - Chunked uploads, resumable
2. **Object Storage** - GCS/S3 for raw videos
3. **Transcoding** - Multiple resolutions (360p, 720p, 1080p, 4K)
4. **CDN** - Global edge caching
5. **Adaptive Bitrate** - HLS/DASH streaming

</details>

---

## Behavioral Questions (Googleyness)

1. **Tell me about a time you disagreed with a teammate. How did you handle it?**

2. **Describe a situation where you had to make a decision with incomplete information.**

3. **Tell me about your most challenging project and how you overcame obstacles.**

4. **How do you prioritize when you have multiple urgent tasks?**

5. **Describe a time you received critical feedback and how you responded.**

---

## Interview Tips for Google

1. **Think out loud** - Explain your thought process
2. **Ask clarifying questions** - Don't assume requirements
3. **Start with brute force** - Then optimize
4. **Test your code** - Walk through with examples
5. **Discuss trade-offs** - No solution is perfect
6. **Be collaborative** - The interviewer is your partner

---

## Preparation Timeline

```
Month 1-2: DSA Foundations
├── LeetCode: 100+ problems (Easy/Medium)
├── Focus: Arrays, Strings, Trees, Graphs
└── Goal: Solve Medium problems in 20-25 min

Month 3-4: Advanced DSA + System Design
├── LeetCode: 50+ Medium/Hard problems
├── System Design: Read DDIA, practice designs
└── Goal: Comfortable with common patterns

Month 5-6: Mock Interviews + Google-Specific
├── Pramp, Interviewing.io mock interviews
├── Google-specific: Read engineering blog
└── Goal: Simulate real interview conditions
```

---

## Resources

- [Kubernetes Questions](../../interview-questions/devops/kubernetes.md)
- [Docker Questions](../../interview-questions/devops/docker.md)
- [GCP Questions](../../interview-questions/cloud/gcp.md)
- [Terraform Questions](../../interview-questions/devops/terraform.md)
- [System Design Roadmap](../../roadmaps/cloud-engineer.md)

---

*Build real projects for your portfolio? [Try DeployU](https://deployu.ai?ref=github) - Deploy Kubernetes, Terraform, and cloud infrastructure with zero billing risk.*
