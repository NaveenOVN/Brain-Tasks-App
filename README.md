# Brain Tasks App — Production Deployment

This repository contains the production-ready build (`dist/`) for the Brain Tasks App, deployed end-to-end to AWS using Docker, ECR, EKS, CodeBuild, and CodePipeline, with CloudWatch monitoring.

**Live app:** running on an AWS Classic Load Balancer, port 3000 (see LoadBalancer ARN below).

---

## Architecture

GitHub (source) -> CodePipeline -> CodeBuild -> Amazon ECR -> Amazon EKS -> LoadBalancer (port 3000)
|
CloudWatch (build logs + Container Insights)


- **Docker**: static `dist/` build served via `serve` on port 3000
- **Registry**: Amazon ECR
- **Orchestration**: Amazon EKS (2-node managed nodegroup)
- **CI**: AWS CodeBuild — builds image, pushes to ECR, deploys to EKS via `kubectl`
- **CD**: AWS CodePipeline — triggers CodeBuild automatically on every push to `main`
- **Monitoring**: CloudWatch Container Insights (cluster/pod metrics) + CodeBuild logs (build history)

---

## Repository contents

| File | Purpose |
|---|---|
| `dist/` | Pre-built static production files (HTML/CSS/JS) |
| `Dockerfile` | Serves `dist/` via `serve` on port 3000 |
| `.dockerignore` | Excludes `.git`, `node_modules` from build context |
| `deployment.yaml` | Kubernetes Deployment (2 replicas, port 3000) |
| `service.yaml` | Kubernetes Service, type `LoadBalancer`, port 3000 |
| `buildspec.yml` | CodeBuild instructions: install tools, build, push, deploy |
| `screenshots/` | Evidence of each pipeline stage (see below) |

---

## Setup instructions (reproduce from scratch)

### 1. Prerequisites (Ubuntu)
Install Docker, AWS CLI v2, `kubectl`, and `eksctl`. Configure AWS credentials with an IAM user that has ECR/EKS/CodeBuild/CodePipeline permissions:
```bash
aws configure
```

### 2. Dockerize and test locally
```bash
docker build -t brain-tasks-app:latest .
docker run -d -p 3000:3000 --name brain-tasks-test brain-tasks-app:latest
curl -I http://localhost:3000   # expect HTTP/1.1 200 OK
```

### 3. Push to Amazon ECR
```bash
aws ecr create-repository --repository-name brain-tasks-app --region ap-south-1
aws ecr get-login-password --region ap-south-1 | docker login --username AWS --password-stdin 792682046782.dkr.ecr.ap-south-1.amazonaws.com
docker tag brain-tasks-app:latest 792682046782.dkr.ecr.ap-south-1.amazonaws.com/brain-tasks-app:latest
docker push 792682046782.dkr.ecr.ap-south-1.amazonaws.com/brain-tasks-app:latest
```

### 4. Create the EKS cluster
```bash
eksctl create cluster \
  --name brain-tasks-cluster \
  --region ap-south-1 \
  --nodegroup-name brain-tasks-nodes \
  --node-type t3.medium \
  --nodes 2 --nodes-min 1 --nodes-max 3 \
  --managed
aws eks update-kubeconfig --name brain-tasks-cluster --region ap-south-1
```

### 5. Deploy to Kubernetes
```bash
kubectl apply -f deployment.yaml
kubectl apply -f service.yaml
kubectl get svc brain-tasks-service   # note the EXTERNAL-IP / LoadBalancer DNS
```

### 6. CodeBuild
Create a CodeBuild project connected to this GitHub repo, environment set to **Privileged** (required for Docker builds), using the `buildspec.yml` in this repo. Grant the CodeBuild service role:
- `AmazonEC2ContainerRegistryPowerUser` (ECR push/pull)
- An inline policy for `eks:DescribeCluster` scoped to the cluster ARN
- An EKS access entry with `AmazonEKSClusterAdminPolicy` (cluster scope), and an entry in the `aws-auth` ConfigMap under `system:masters`

### 7. CodePipeline
Create a pipeline with:
- **Source**: this GitHub repo, branch `main` (triggers on push via GitHub App)
- **Build**: the CodeBuild project above (deploy to EKS happens inside the build via `buildspec.yml`'s `post_build` phase, so no separate Deploy stage is needed)

The CodePipeline service role needs `iam:PassRole` permission scoped to the CodeBuild service role's ARN.

### 8. Monitoring
```bash
curl -s https://raw.githubusercontent.com/aws-samples/amazon-cloudwatch-container-insights/latest/k8s-deployment-manifest-templates/deployment-mode/daemonset/container-insights-monitoring/quickstart/cwagent-fluent-bit-quickstart.yaml \
  -o cwagent-fluent-bit-quickstart.yaml
sed -i 's/{{cluster_name}}/brain-tasks-cluster/g; s/{{region_name}}/ap-south-1/g; s/{{http_server_toggle}}/"On"/g; s/{{http_server_port}}/"2020"/g; s/{{read_from_head}}/"Off"/g; s/{{read_from_tail}}/"On"/g' cwagent-fluent-bit-quickstart.yaml
kubectl apply -f cwagent-fluent-bit-quickstart.yaml

aws iam attach-role-policy \
  --role-name eksctl-brain-tasks-cluster-nodegro-NodeInstanceRole-6RvarFiShBP5 \
  --policy-arn arn:aws:iam::aws:policy/CloudWatchAgentServerPolicy
```
CodeBuild logs stream to CloudWatch Logs automatically under `/aws/codebuild/brain-tasks-build`. Container Insights metrics appear under the `ContainerInsights` CloudWatch namespace.

---

## Pipeline explanation

1. A developer pushes a commit to `main` on GitHub.
2. **CodePipeline** detects the push (via GitHub App webhook) and triggers the **Source** stage.
3. **CodeBuild** runs, following `buildspec.yml`:
   - `install`: pulls a pinned version of `kubectl` (v1.34.0) and `aws-cli` (avoids stale-tool auth issues against the EKS API)
   - `pre_build`: authenticates Docker to ECR
   - `build`: builds the Docker image, tags it with the Git commit hash and `latest`
   - `post_build`: pushes both tags to ECR, updates kubeconfig for the EKS cluster, runs `kubectl set image` to roll the new image out to the running Deployment, and waits for `kubectl rollout status` to confirm the rollout succeeded
4. If the build succeeds, the pipeline reports success; the new version is now live behind the LoadBalancer with zero manual steps.

---

## Deployed LoadBalancer

- **DNS name:** a90a6a6edb4de4783a5f3582f89e3989-1111815946.ap-south-1.elb.amazonaws.com
- **ARN:** arn:aws:elasticloadbalancing:ap-south-1:792682046782:loadbalancer/a90a6a6edb4de4783a5f3582f89e3989
