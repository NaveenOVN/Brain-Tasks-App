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
  #Cloudwatch logs

   2026-09-20T13:15:48.978Z
[Container] 2026/09/20 13:15:44.942156 Processing environment variables
2026-09-20T13:15:48.978Z
[Container] 2026/09/20 13:15:45.128166 No runtime version selected in buildspec.
2026-09-20T13:15:48.978Z
[Container] 2026/09/20 13:15:45.194871 Moving to directory /codebuild/output/src2289385213/src/github.com/NaveenOVN/Brain-Tasks-App
2026-09-20T13:15:48.978Z
[Container] 2026/09/20 13:15:45.194897 Cache is not defined in the buildspec
2026-09-20T13:15:48.978Z
[Container] 2026/09/20 13:15:45.229027 Skip cache due to: no paths specified to be cached
2026-09-20T13:15:48.978Z
[Container] 2026/09/20 13:15:45.229260 Registering with agent
2026-09-20T13:15:48.978Z
[Container] 2026/09/20 13:15:45.259330 Phases found in YAML: 4
2026-09-20T13:15:48.978Z
[Container] 2026/09/20 13:15:45.259348 POST_BUILD: 5 commands
2026-09-20T13:15:48.978Z
[Container] 2026/09/20 13:15:45.259352 INSTALL: 8 commands
2026-09-20T13:15:48.978Z
[Container] 2026/09/20 13:15:45.259354 PRE_BUILD: 3 commands
2026-09-20T13:15:48.978Z
[Container] 2026/09/20 13:15:45.259358 BUILD: 2 commands
2026-09-20T13:15:48.978Z
[Container] 2026/09/20 13:15:45.259697 Phase complete: DOWNLOAD_SOURCE State: SUCCEEDED
2026-09-20T13:15:48.978Z
[Container] 2026/09/20 13:15:45.259711 Phase context status code: Message:
2026-09-20T13:15:48.978Z
[Container] 2026/09/20 13:15:45.868627 Entering phase INSTALL
2026-09-20T13:15:48.978Z
[Container] 2026/09/20 13:15:45.906553 Running command curl -LO "https://dl.k8s.io/release/v1.34.0/bin/linux/amd64/kubectl"
2026-09-20T13:15:48.978Z
% Total % Received % Xferd Average Speed Time Time Time Current
2026-09-20T13:15:48.978Z
Dload Upload Total Spent Left Speed
2026-09-20T13:15:48.978Z
0 0 0 0 0 0 0 0 --:--:-- --:--:-- --:--:-- 0 100 59140k 100 59140k 0 0 479.9M 0 --:--:-- --:--:-- --:--:-- 481.2M
2026-09-20T13:15:48.978Z
2026-09-20T13:15:48.978Z
[Container] 2026/09/20 13:15:46.074680 Running command chmod +x ./kubectl
2026-09-20T13:15:48.978Z
2026-09-20T13:15:48.978Z
[Container] 2026/09/20 13:15:46.112503 Running command mv ./kubectl /usr/local/bin/kubectl
2026-09-20T13:15:48.978Z
2026-09-20T13:15:48.978Z
[Container] 2026/09/20 13:15:46.222200 Running command kubectl version --client
2026-09-20T13:15:48.978Z
Client Version: v1.34.0
2026-09-20T13:15:48.978Z
Kustomize Version: v5.7.1
2026-09-20T13:15:48.978Z
2026-09-20T13:15:48.978Z
[Container] 2026/09/20 13:15:46.295645 Running command curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
2026-09-20T13:15:48.978Z
% Total % Received % Xferd Average Speed Time Time Time Current
2026-09-20T13:15:48.978Z
Dload Upload Total Spent Left Speed
2026-09-20T13:15:48.978Z
0 0 0 0 0 0 0 0 --:--:-- --:--:-- --:--:-- 0 100 71860k 100 71860k 0 0 419.9M 0 --:--:-- --:--:-- --:--:-- 420.2M
2026-09-20T13:15:48.978Z
2026-09-20T13:15:48.978Z
[Container] 2026/09/20 13:15:46.473943 Running command unzip -q -o awscliv2.zip
2026-09-20T13:15:48.978Z
2026-09-20T13:15:48.978Z
[Container] 2026/09/20 13:15:48.526196 Running command ./aws/install --update
2026-09-20T13:15:53.015Z
You can now run: /usr/local/bin/aws --version
2026-09-20T13:15:53.015Z
2026-09-20T13:15:53.015Z
[Container] 2026/09/20 13:15:52.245007 Running command aws --version
2026-09-20T13:15:53.015Z
aws-cli/2.36.49 Python/3.14.6 Linux/4.14.355-284.742.amzn2.x86_64 exec-env/AWS_ECS_EC2 exe/x86_64.amzn.2023
2026-09-20T13:15:53.015Z
2026-09-20T13:15:53.015Z
[Container] 2026/09/20 13:15:52.843662 Phase complete: INSTALL State: SUCCEEDED
2026-09-20T13:15:53.015Z
[Container] 2026/09/20 13:15:52.843684 Phase context status code: Message:
2026-09-20T13:15:53.015Z
[Container] 2026/09/20 13:15:52.875514 Entering phase PRE_BUILD
2026-09-20T13:15:53.015Z
[Container] 2026/09/20 13:15:52.876485 Running command aws ecr get-login-password --region $AWS_REGION | docker login --username AWS --password-stdin $ECR_REPO
2026-09-20T13:15:55.028Z
2026-09-20T13:15:55.028Z
WARNING! Your credentials are stored unencrypted in '/root/.docker/config.json'.
2026-09-20T13:15:55.028Z
Configure a credential helper to remove this warning. See
2026-09-20T13:15:55.028Z
https://docs.docker.com/go/credential-store/
2026-09-20T13:15:55.028Z
2026-09-20T13:15:55.028Z
Login Succeeded
2026-09-20T13:15:55.028Z
2026-09-20T13:15:55.028Z
[Container] 2026/09/20 13:15:53.664199 Running command COMMIT_HASH=$(echo $CODEBUILD_RESOLVED_SOURCE_VERSION | cut -c 1-7)
2026-09-20T13:15:55.028Z
2026-09-20T13:15:55.028Z
[Container] 2026/09/20 13:15:53.674236 Running command IMAGE_TAG=${COMMIT_HASH:=latest}
2026-09-20T13:15:55.028Z
2026-09-20T13:15:55.028Z
[Container] 2026/09/20 13:15:53.682373 Phase complete: PRE_BUILD State: SUCCEEDED
2026-09-20T13:15:55.028Z
[Container] 2026/09/20 13:15:53.682392 Phase context status code: Message:
2026-09-20T13:15:55.028Z
[Container] 2026/09/20 13:15:53.717848 Entering phase BUILD
2026-09-20T13:15:55.028Z
[Container] 2026/09/20 13:15:53.718780 Running command docker build -t $ECR_REPO:$IMAGE_TAG .
2026-09-20T13:15:55.028Z
#0 building with "default" instance using docker driver
2026-09-20T13:15:55.028Z
2026-09-20T13:15:55.028Z
#1 [internal] load build definition from Dockerfile
2026-09-20T13:15:55.028Z
#1 transferring dockerfile: 168B done
2026-09-20T13:15:55.028Z
#1 DONE 0.0s
2026-09-20T13:15:55.028Z
2026-09-20T13:15:55.028Z
#2 [internal] load metadata for docker.io/library/node:18-alpine
2026-09-20T13:15:57.042Z
#2 DONE 1.8s
2026-09-20T13:15:57.042Z
2026-09-20T13:15:57.042Z
#3 [internal] load .dockerignore
2026-09-20T13:15:57.042Z
#3 transferring context: 58B done
2026-09-20T13:15:57.042Z
#3 DONE 0.0s
2026-09-20T13:15:57.042Z
2026-09-20T13:15:57.042Z
#4 [internal] load build context
2026-09-20T13:15:57.042Z
#4 transferring context: 317.98kB done
2026-09-20T13:15:57.042Z
#4 DONE 0.0s
2026-09-20T13:15:57.042Z
2026-09-20T13:15:57.042Z
#5 [1/4] FROM docker.io/library/node:18-alpine@sha256:8d6421d663b4c28fd3ebc498332f249011d118945588d0a35cb9bc4b8ca09d9e
2026-09-20T13:15:57.042Z
#5 resolve docker.io/library/node:18-alpine@sha256:8d6421d663b4c28fd3ebc498332f249011d118945588d0a35cb9bc4b8ca09d9e 0.0s done
2026-09-20T13:15:57.042Z
#5 sha256:8d6421d663b4c28fd3ebc498332f249011d118945588d0a35cb9bc4b8ca09d9e 7.67kB / 7.67kB done
2026-09-20T13:15:57.042Z
#5 sha256:929b04d7c782f04f615cf785488fed452b6569f87c73ff666ad553a7554f0006 1.72kB / 1.72kB done
2026-09-20T13:15:57.042Z
#5 sha256:ee77c6cd7c1886ecc802ad6cedef3a8ec1ea27d1fb96162bf03dd3710839b8da 6.18kB / 6.18kB done
2026-09-20T13:15:57.042Z
#5 sha256:f18232174bc91741fdf3da96d85011092101a032a93a388b79e99e69c2d5c870 0B / 3.64MB 0.1s
2026-09-20T13:15:57.042Z
#5 sha256:dd71dde834b5c203d162902e6b8994cb2309ae049a0eabc4efea161b2b5a3d0e 0B / 40.01MB 0.1s
2026-09-20T13:15:57.042Z
#5 sha256:1e5a4c89cee5c0826c540ab06d4b6b491c96eda01837f430bd47f0d26702d6e3 0B / 1.26MB 0.1s
2026-09-20T13:15:57.042Z
#5 sha256:f18232174bc91741fdf3da96d85011092101a032a93a388b79e99e69c2d5c870 3.64MB / 3.64MB 0.2s done
2026-09-20T13:15:57.042Z
#5 extracting sha256:f18232174bc91741fdf3da96d85011092101a032a93a388b79e99e69c2d5c870 0.1s
2026-09-20T13:15:57.042Z
#5 sha256:25ff2da83641908f65c3a74d80409d6b1b62ccfaab220b9ea70b80df5a2e0549 0B / 446B 0.3s
2026-09-20T13:15:57.042Z
#5 sha256:dd71dde834b5c203d162902e6b8994cb2309ae049a0eabc4efea161b2b5a3d0e 19.92MB / 40.01MB 0.5s
2026-09-20T13:15:57.042Z
#5 extracting sha256:f18232174bc91741fdf3da96d85011092101a032a93a388b79e99e69c2d5c870 0.1s done
2026-09-20T13:15:57.042Z
#5 sha256:dd71dde834b5c203d162902e6b8994cb2309ae049a0eabc4efea161b2b5a3d0e 40.01MB / 40.01MB 0.6s
2026-09-20T13:15:57.042Z
#5 sha256:1e5a4c89cee5c0826c540ab06d4b6b491c96eda01837f430bd47f0d26702d6e3 1.26MB / 1.26MB 0.6s
2026-09-20T13:15:57.042Z
#5 sha256:25ff2da83641908f65c3a74d80409d6b1b62ccfaab220b9ea70b80df5a2e0549 446B / 446B 0.6s
2026-09-20T13:15:57.042Z
#5 sha256:dd71dde834b5c203d162902e6b8994cb2309ae049a0eabc4efea161b2b5a3d0e 40.01MB / 40.01MB 0.6s done
2026-09-20T13:15:57.042Z
#5 sha256:1e5a4c89cee5c0826c540ab06d4b6b491c96eda01837f430bd47f0d26702d6e3 1.26MB / 1.26MB 0.6s done
2026-09-20T13:15:57.042Z
#5 sha256:25ff2da83641908f65c3a74d80409d6b1b62ccfaab220b9ea70b80df5a2e0549 446B / 446B 0.6s done
2026-09-20T13:15:57.042Z
#5 extracting sha256:dd71dde834b5c203d162902e6b8994cb2309ae049a0eabc4efea161b2b5a3d0e 0.1s
2026-09-20T13:15:59.055Z
#5 extracting sha256:dd71dde834b5c203d162902e6b8994cb2309ae049a0eabc4efea161b2b5a3d0e 1.2s done
2026-09-20T13:15:59.055Z
#5 extracting sha256:1e5a4c89cee5c0826c540ab06d4b6b491c96eda01837f430bd47f0d26702d6e3
2026-09-20T13:15:59.055Z
#5 extracting sha256:1e5a4c89cee5c0826c540ab06d4b6b491c96eda01837f430bd47f0d26702d6e3 0.1s done
2026-09-20T13:15:59.055Z
#5 extracting sha256:25ff2da83641908f65c3a74d80409d6b1b62ccfaab220b9ea70b80df5a2e0549 done
2026-09-20T13:15:59.055Z
#5 DONE 2.2s
2026-09-20T13:15:59.055Z
2026-09-20T13:15:59.055Z
#6 [2/4] WORKDIR /app
2026-09-20T13:15:59.055Z
#6 DONE 1.0s
2026-09-20T13:16:01.071Z
2026-09-20T13:16:01.071Z
#7 [3/4] RUN npm install -g serve
2026-09-20T13:16:03.086Z
#7 3.552
2026-09-20T13:16:03.086Z
#7 3.552 added 85 packages in 3s
2026-09-20T13:16:03.086Z
#7 3.552
2026-09-20T13:16:03.086Z
#7 3.552 26 packages are looking for funding
2026-09-20T13:16:03.086Z
#7 3.552 run `npm fund` for details
2026-09-20T13:16:03.086Z
#7 3.555 npm notice
2026-09-20T13:16:03.086Z
#7 3.555 npm notice New major version of npm available! 10.8.2 -> 12.0.2
2026-09-20T13:16:03.086Z
#7 3.555 npm notice Changelog: https://github.com/npm/cli/releases/tag/v12.0.2
2026-09-20T13:16:03.086Z
#7 3.555 npm notice To update run: npm install -g npm@12.0.2
2026-09-20T13:16:03.086Z
#7 3.555 npm notice
2026-09-20T13:16:03.086Z
#7 DONE 3.9s
2026-09-20T13:16:03.086Z
2026-09-20T13:16:03.086Z
#8 [4/4] COPY dist ./dist
2026-09-20T13:16:03.086Z
#8 DONE 0.1s
2026-09-20T13:16:03.086Z
2026-09-20T13:16:03.086Z
#9 exporting to image
2026-09-20T13:16:03.086Z
#9 exporting layers
2026-09-20T13:16:05.097Z
#9 exporting layers 0.2s done
2026-09-20T13:16:05.097Z
#9 writing image sha256:cd8ad4eb37cae3bb90bd52f3378efa98877d517a2b414f2721f8396978fcf48c done
2026-09-20T13:16:05.097Z
#9 naming to 792682046782.dkr.ecr.ap-south-1.amazonaws.com/brain-tasks-app:ddb4b55 done
2026-09-20T13:16:05.097Z
#9 DONE 0.2s
2026-09-20T13:16:05.097Z
2026-09-20T13:16:05.097Z
[Container] 2026/09/20 13:16:03.176801 Running command docker tag $ECR_REPO:$IMAGE_TAG $ECR_REPO:latest
2026-09-20T13:16:05.097Z
2026-09-20T13:16:05.097Z
[Container] 2026/09/20 13:16:03.200685 Phase complete: BUILD State: SUCCEEDED
2026-09-20T13:16:05.097Z
[Container] 2026/09/20 13:16:03.200701 Phase context status code: Message:
2026-09-20T13:16:05.097Z
[Container] 2026/09/20 13:16:03.235444 Entering phase POST_BUILD
2026-09-20T13:16:05.097Z
[Container] 2026/09/20 13:16:03.236483 Running command docker push $ECR_REPO:$IMAGE_TAG
2026-09-20T13:16:05.097Z
The push refers to repository [792682046782.dkr.ecr.ap-south-1.amazonaws.com/brain-tasks-app]
2026-09-20T13:16:05.097Z
46547b5eb7c3: Preparing
2026-09-20T13:16:05.097Z
bcc8e92c928f: Preparing
2026-09-20T13:16:05.097Z
58c4e7fba993: Preparing
2026-09-20T13:16:05.097Z
82140d9a70a7: Preparing
2026-09-20T13:16:05.097Z
f3b40b0cdb1c: Preparing
2026-09-20T13:16:05.097Z
0b1f26057bd0: Preparing
2026-09-20T13:16:05.097Z
08000c18d16d: Preparing
2026-09-20T13:16:05.097Z
0b1f26057bd0: Waiting
2026-09-20T13:16:05.097Z
08000c18d16d: Waiting
2026-09-20T13:16:05.097Z
f3b40b0cdb1c: Layer already exists
2026-09-20T13:16:05.097Z
82140d9a70a7: Layer already exists
2026-09-20T13:16:05.097Z
0b1f26057bd0: Layer already exists
2026-09-20T13:16:05.097Z
08000c18d16d: Layer already exists
2026-09-20T13:16:05.097Z
58c4e7fba993: Pushed
2026-09-20T13:16:05.097Z
46547b5eb7c3: Pushed
2026-09-20T13:16:05.097Z
bcc8e92c928f: Pushed
2026-09-20T13:16:05.097Z
ddb4b55: digest: sha256:d78d91e817d4e0fc344e036ebe7d4bc635321b87ff6dae55bf9782213d16a95d size: 1786
2026-09-20T13:16:05.097Z
2026-09-20T13:16:05.097Z
[Container] 2026/09/20 13:16:04.656198 Running command docker push $ECR_REPO:latest
2026-09-20T13:16:05.097Z
The push refers to repository [792682046782.dkr.ecr.ap-south-1.amazonaws.com/brain-tasks-app]
2026-09-20T13:16:05.097Z
46547b5eb7c3: Preparing
2026-09-20T13:16:05.097Z
bcc8e92c928f: Preparing
2026-09-20T13:16:05.097Z
58c4e7fba993: Preparing
2026-09-20T13:16:05.097Z
82140d9a70a7: Preparing
2026-09-20T13:16:05.097Z
f3b40b0cdb1c: Preparing
2026-09-20T13:16:05.097Z
0b1f26057bd0: Preparing
2026-09-20T13:16:05.097Z
08000c18d16d: Preparing
2026-09-20T13:16:05.097Z
0b1f26057bd0: Waiting
2026-09-20T13:16:05.097Z
08000c18d16d: Waiting
2026-09-20T13:16:05.097Z
58c4e7fba993: Layer already exists
2026-09-20T13:16:05.097Z
bcc8e92c928f: Layer already exists
2026-09-20T13:16:05.097Z
46547b5eb7c3: Layer already exists
2026-09-20T13:16:05.097Z
f3b40b0cdb1c: Layer already exists
2026-09-20T13:16:05.097Z
82140d9a70a7: Layer already exists
2026-09-20T13:16:05.097Z
0b1f26057bd0: Layer already exists
2026-09-20T13:16:05.097Z
08000c18d16d: Layer already exists
2026-09-20T13:16:05.097Z
latest: digest: sha256:d78d91e817d4e0fc344e036ebe7d4bc635321b87ff6dae55bf9782213d16a95d size: 1786
2026-09-20T13:16:05.097Z
2026-09-20T13:16:05.097Z
[Container] 2026/09/20 13:16:04.794847 Running command aws eks update-kubeconfig --name $CLUSTER_NAME --region $AWS_REGION
2026-09-20T13:16:07.110Z
Added new context arn:aws:eks:ap-south-1:792682046782:cluster/brain-tasks-cluster to /root/.kube/config
2026-09-20T13:16:07.110Z
2026-09-20T13:16:07.110Z
[Container] 2026/09/20 13:16:05.677498 Running command kubectl set image deployment/brain-tasks-app brain-tasks-app=$ECR_REPO:$IMAGE_TAG
2026-09-20T13:16:07.110Z
deployment.apps/brain-tasks-app image updated
2026-09-20T13:16:07.110Z
2026-09-20T13:16:07.110Z
[Container] 2026/09/20 13:16:06.499598 Running command kubectl rollout status deployment/brain-tasks-app
2026-09-20T13:16:09.123Z
Waiting for deployment "brain-tasks-app" rollout to finish: 1 out of 2 new replicas have been updated...
2026-09-20T13:16:09.123Z
Waiting for deployment "brain-tasks-app" rollout to finish: 1 out of 2 new replicas have been updated...
2026-09-20T13:16:09.123Z
Waiting for deployment "brain-tasks-app" rollout to finish: 1 out of 2 new replicas have been updated...
2026-09-20T13:16:09.123Z
Waiting for deployment "brain-tasks-app" rollout to finish: 1 old replicas are pending termination...
2026-09-20T13:16:11.136Z
Waiting for deployment "brain-tasks-app" rollout to finish: 1 old replicas are pending termination...
2026-09-20T13:16:11.136Z
deployment "brain-tasks-app" successfully rolled out
2026-09-20T13:16:11.136Z
2026-09-20T13:16:11.136Z
[Container] 2026/09/20 13:16:11.019201 Phase complete: POST_BUILD State: SUCCEEDED
2026-09-20T13:16:11.136Z
[Container] 2026/09/20 13:16:11.019216 Phase context status code: Message:
2026-09-20T13:16:11.136Z
[Container] 2026/09/20 13:16:11.070269 Set report auto-discover timeout to 5 seconds
2026-09-20T13:16:11.136Z
[Container] 2026/09/20 13:16:11.070318 Expanding base directory path: .
2026-09-20T13:16:11.136Z
[Container] 2026/09/20 13:16:11.073894 Assembling file list
2026-09-20T13:16:11.136Z
[Container] 2026/09/20 13:16:11.073909 Expanding .
2026-09-20T13:16:11.136Z
[Container] 2026/09/20 13:16:11.077559 Expanding file paths for base directory .
2026-09-20T13:16:11.136Z
[Container] 2026/09/20 13:16:11.077573 Assembling file list
2026-09-20T13:16:11.136Z
[Container] 2026/09/20 13:16:11.077576 Expanding **/*
2026-09-20T13:16:11.136Z
[Container] 2026/09/20 13:16:11.113360 No matching auto-discover report paths found
2026-09-20T13:16:11.136Z
[Container] 2026/09/20 13:16:11.113425 Report auto-discover file discovery took 0.043155 seconds
2026-09-20T13:16:11.136Z
[Container] 2026/09/20 13:16:11.113454 Phase complete: UPLOAD_ARTIFACTS State: SUCCEEDED
2026-09-20T13:16:11.136Z
[Container] 2026/09/20 13:16:11.113468 Phase context status code: Message:

CodeBuild logs stream to CloudWatch Logs automatically under `/aws/codebui
ld/brain-tasks-build`. Container Insights metrics appear under the `ContainerInsights` CloudWatch namespace.

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
