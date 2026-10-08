# HumanGov — SaaS HR Application on Amazon EKS

Multi-tenant HR application for managing government employees, with an
isolated deployment per U.S. state. Containerized and running on Amazon EKS
behind an ALB Ingress with TLS. Delivered through a CodePipeline / CodeBuild
pipeline with test, staging, and manual approval gates.

→ [EKS Deployment](https://medium.com/@ivantrevino/humangov-deployment-of-humangov-saas-application-on-aws-elastic-kubernetes-service-eks-using-a-da0f64ab9ad9) · [CI/CD Pipeline](https://medium.com/@ivantrevino/humangov-automating-humangov-saas-application-build-and-deployment-process-on-kubernetes-with-a9f167546fad) · [Synthetics Monitoring](https://medium.com/@ivantrevino/humangov-implementing-aws-cloudwatch-synthetics-for-real-time-application-url-monitoring-98a3c5c3e153) · [Lambda Microservice](https://medium.com/@ivantrevino/humangov-developing-an-event-driven-and-serverless-python-micro-service-triggered-by-dynamodb-9ca4306293ce)

Infrastructure (per-state DynamoDB tables and S3 buckets) lives in the
companion repo: human-gov-infrastructure.

---

## What This Does

Each state gets its own pair of pods, NGINX reverse proxy and Flask app on
Gunicorn, in a shared EKS cluster. Each pair points at that state's own
DynamoDB table and S3 bucket. A single ALB, created by the AWS Load Balancer
Controller from one Ingress resource, routes each state subdomain to its NGINX
service over HTTPS. Pods reach DynamoDB and S3 through an IAM role bound to a
Kubernetes service account (IRSA), so no static keys are stored in the
cluster.

Pushes to master trigger the pipeline. Each run builds a new image tagged with
the commit ID, tests that exact image with pytest, and deploys it to staging.
After a manual approval, it rolls the same image to production.

---

## Architecture

- **Amazon EKS** — single-node cluster (t3.medium) provisioned with eksctl
- **NGINX (nginx:alpine)** — reverse proxy per state, config supplied by a ConfigMap
- **Flask + Gunicorn** — application container on port 8000
- **AWS Load Balancer Controller** — installed via Helm, provisions one internet-facing ALB from the Ingress
- **ACM** — wildcard certificate on the custom domain, HTTP → HTTPS redirect at the ALB
- **Route53** — alias record per state subdomain (cali., florida., staging.) pointing at the ALB
- **IRSA** — `humangov-pod-execution-role` service account grants pods DynamoDB and S3 access
- **ECR** — application image registry
- **DynamoDB** — employee records, one table per state
- **S3** — employee ID documents (PDF), one bucket per state
- **CodePipeline + CodeBuild** — Source → Build → Test → Deploy to Staging → Manual Approval → Deploy to Production
- **CloudWatch Synthetics** — one canary per state checking the URL every minute
- **CloudWatch Alarms + SNS + Amazon Q Developer in chat applications** — canary failures alert a Slack #sre-team channel
- **Lambda + DynamoDB Streams** — Python/Boto3 microservice that deletes a removed employee's PDF from S3

---

## Repository Structure

    src/
      humangov.py                 Flask app (Gunicorn entrypoint humangov:app)
      requirements.txt            App dependencies, including pytest
      templates/                  HTML templates
      Dockerfile                  python:3.8-slim-buster, Gunicorn bound to 0.0.0.0:8000
      humangov-california.yaml    Deployments, Services, and NGINX ConfigMap for California
      humangov-florida.yaml       Same, for Florida
      humangov-staging.yaml       Staging environment used by the pipeline
      humangov-ingress-all.yaml   Shared ALB Ingress, one host rule per state
      tests/
        app.py                    Home page string function under test
        test_app.py               pytest assertion run in the Test stage
    nginx/                        NGINX Dockerfile and config from the earlier
                                  ECS proof of concept (EKS uses nginx:alpine
                                  with a ConfigMap instead)

---

## Kubernetes Manifests

Each `humangov-<state>.yaml` defines:

- **App Deployment** — runs the app image under the `humangov-pod-execution-role` service account
- **App Service (ClusterIP)** — exposes the app on port 8000
- **NGINX Deployment** — `nginx:alpine` with the ConfigMap mounted at `/etc/nginx/`
- **NGINX Service** — exposes NGINX on port 80; this is the Ingress backend
- **ConfigMap** — `nginx.conf` proxying to the state's app service, plus `proxy_params`

Per-state app environment variables:

    AWS_BUCKET            State S3 bucket (from infrastructure Terraform outputs)
    AWS_DYNAMODB_TABLE    State DynamoDB table (from infrastructure Terraform outputs)
    AWS_REGION            us-east-1
    US_STATE              State name

The image field is the placeholder `CONTAINER_IMAGE`. The deploy stages
replace it with the exact image URI from the Build stage before applying the
manifest.

The Ingress uses `target-type: ip`, listens on 80 and 443, redirects to 443,
and joins the `frontend` ALB group. Every state shares that one ALB, with a
separate listener rule per host.

---

## Cluster Prerequisites

These are one-time setups done from the CLI, not stored in this repo:

1. EKS cluster created with eksctl, and kubeconfig updated with `aws eks update-kubeconfig`
2. IAM OIDC provider associated with the cluster
3. AWS Load Balancer Controller: IAM policy, IRSA service account, CRDs, and Helm install
4. `humangov-pod-execution-role` IRSA service account with DynamoDB and S3 access
5. ACM certificate issued and DNS-validated in the Route53 hosted zone

The Load Balancer Controller's CRDs must be applied separately before the Helm
install. Without them, the controller can't manage the ALB and the Ingress
never routes traffic.

---

## Pipeline Stages (CodeBuild)

The buildspecs are defined inline in each CodeBuild project, not as files in
this repo.

- **Build** — logs in to ECR and builds from `src/`. Tags the image with both `latest` and `$CODEBUILD_RESOLVED_SOURCE_VERSION` and pushes both. Writes `imagedefinitions.json`, then passes it and the manifests on as build artifacts.
- **Test** — pulls the exact image built above and runs `pytest tests/` inside it. A failing assertion stops the pipeline before anything is deployed.
- **Deploy to Staging** — reads the image URI with `jq`, substitutes it into `humangov-staging.yaml` with `sed`, and runs `kubectl apply`.
- **Manual Approval** — a human reviews staging before production.
- **Deploy to Production** — applies the same image to the California and Florida manifests.

Pipeline environment variables:

    ECR_REPO                 ECR repository URI (Build stage)
    AWS_ACCESS_KEY_ID        Cluster access for kubectl (Deploy stages)
    AWS_SECRET_ACCESS_KEY    Cluster access for kubectl (Deploy stages)

The Test stage was validated with a deliberate error: removing one letter from
the expected home page string stopped the pipeline at Test. Restoring it let
the pipeline pass through to production.

---

## Adding a State

1. Add the state to the `states` list in human-gov-infrastructure and run `terraform apply`
2. Copy an existing state manifest and replace the state name throughout
3. Set the new bucket and table values from the Terraform outputs
4. Add a host rule to `humangov-ingress-all.yaml` and run `kubectl apply`
5. Create a Route53 alias record for the new subdomain pointing at the ALB
6. Add the new manifest to the Build artifacts and the production deploy buildspec

---

## Bug Fix: Orphaned PDFs

Deleting an employee removed the DynamoDB record but left the PDF in S3. The
fix is an event-driven Lambda function, so no application code changed:

- DynamoDB Stream enabled on the state table, with new and old images
- Lambda (Python, Boto3) triggered by the stream; on `REMOVE` events it reads the PDF key from the old image and deletes that object from S3
- Each deletion is logged to the function's CloudWatch log group

---

## Known Limitations

- ECR repository is public. Production should use a private repository.
- Deploy stages authenticate to EKS with IAM user access keys. Production should map the CodeBuild service role into the cluster instead.
- Pod and Lambda roles use AmazonS3FullAccess and AmazonDynamoDBFullAccess. These should be scoped to each state's own table and bucket.
- The kubectl version pinned in the deploy buildspecs is outdated and should match the cluster version.
- Bucket names are hardcoded in the manifests and the Lambda function. They should be injected per environment.

---

The base application source was provided by The Cloud Bootcamp. The
containerization, Kubernetes deployment, pipeline, monitoring, and Lambda fix
are my own work.

Originally hosted on AWS CodeCommit, migrated to GitHub.
