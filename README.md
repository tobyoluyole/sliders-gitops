# Sliders Kubernetes GitOps

A GitOps deployment platform for Sliders microservices on Amazon EKS. I designed and implemented the Kubernetes deployment configuration, Argo CD application structure, AWS Secrets Manager integration, persistent storage, and monitoring setup in this repository.

This repository contains the Kubernetes manifests and Argo CD application definitions. AWS infrastructure, IAM roles, and cluster add-ons are managed in a separate Terraform repository.

The deployment separates application workloads, databases, monitoring, and shared infrastructure into distinct namespaces and Argo CD applications. The configuration has been deployed and verified on a staging EKS cluster, with application workloads running in the `production` namespace.

## Platform capabilities

- GitOps deployment with Argo CD, including automatic synchronization and self-healing
- Deployment of an API gateway and seven application services
- Kubernetes secret synchronization from AWS Secrets Manager using External Secrets Operator
- Persistent storage for Redis and RabbitMQ using Amazon EBS gp3 volumes
- HTTPS ingress through an AWS Application Load Balancer and ACM certificate
- Prometheus, Grafana, and Alertmanager deployment through the kube-prometheus-stack Helm chart

## Architecture

```text
Internet
   |
   v
AWS Application Load Balancer (HTTPS)
   |
   v
API Gateway
   |
   +-- Auth Service
   +-- Billing Service
   +-- Case Service
   +-- Client Service
   +-- Document Service
   +-- Notification Service
   `-- User Service
          |
          +-- Redis
          `-- RabbitMQ

AWS Secrets Manager -> External Secrets Operator -> Kubernetes Secrets
Amazon EBS (gp3) --------------------------------> Persistent Volumes
```

## Repository structure

```text
.
|-- aws-secret-store.yaml     # AWS Secrets Manager provider
|-- gp3-storageclass.yaml     # Default Amazon EBS gp3 storage class
|-- monitoring-app.yaml      # Argo CD application for monitoring
|-- argocd
|   |-- infrastructure-app.yaml # Root infrastructure manifests
|   |-- databases-app.yaml      # Redis and RabbitMQ workloads
|   `-- production-app.yaml     # Production namespace workloads
`-- k8s
    |-- databases
    |   |-- rabbitmq          # Deployment, Service, PVC and ExternalSecret
    |   `-- redis             # Deployment, Service and PVC
    `-- production
        |-- api-gateway       # Deployment, Service, ExternalSecret and Ingress
        |-- auth-service
        |-- billing-service
        |-- case-service
        |-- client-service
        |-- document-service
        |-- notification-service
        `-- user-service
```

Application workloads use the `production` namespace. Redis and RabbitMQ use `databases`, monitoring uses `monitoring`, and Argo CD applications live in `argocd`.

## Prerequisites

- An existing Amazon EKS cluster
- `kubectl` configured for the target cluster
- Argo CD installed in the `argocd` namespace, with access to your Git repository
- AWS CLI access to the target AWS account
- AWS Load Balancer Controller
- External Secrets Operator
- Amazon EBS CSI Driver
- Application images pushed to Amazon ECR
- Required values created in AWS Secrets Manager
- DNS records and an ACM certificate covering the API and Grafana hostnames

The current manifests target AWS region `eu-west-1`. Application source code and container builds are outside this repository; you need compatible images and their expected configuration to run the full platform.

## Configuration

If you fork this project, update these values for your own environment before deploying:

- ECR account ID, region, repository and image tag in each `deployment.yaml`
- Git repository URLs and target revisions in `argocd/`
- AWS region in `aws-secret-store.yaml`
- Hostname and ACM certificate ARN in `k8s/production/api-gateway/ingress.yaml`
- Grafana hostname and ACM certificate ARN in `monitoring-app.yaml`
- Replica counts and resource requests
- Redis and RabbitMQ PVC sizes
- AWS Secrets Manager names

The AWS identifiers and hostnames in these manifests belong to my project environment. They are configuration examples, not shared infrastructure for forks.

## Secrets

Credentials are not stored directly in this repository. External Secrets Operator reads AWS Secrets Manager and creates the Kubernetes Secrets consumed by each workload.

Expected secret names are:

```text
api-gateway-credentials
auth-service-credentials
billing-service-credentials
case-service-credentials
client-service-credentials
document-service-credentials
notification-service-credentials
user-service-credentials
rabbitmq-credentials
```

Application secrets must be JSON objects whose keys match the environment variables expected by their service. `rabbitmq-credentials` must contain `username` and `password`.

The `ClusterSecretStore` does not contain static AWS credentials. Configure EKS Pod Identity, IRSA, or another supported authentication method so External Secrets Operator can read the secrets.

## Deployment

### GitOps deployment with Argo CD

After configuring the prerequisites and pushing your customized manifests to your repository, confirm the target cluster and bootstrap the applications:

```bash
kubectl config current-context
kubectl apply -f argocd/
```

`sliders-infrastructure` tracks `main` and automatically syncs exactly the three root manifests: `aws-secret-store.yaml`, `gp3-storageclass.yaml`, and `monitoring-app.yaml`. The monitoring application then deploys its Helm chart into the `monitoring` namespace.

`sliders-databases` recursively syncs `k8s/databases` from `main` into the `databases` namespace. `sliders-production` recursively syncs `k8s/production` from the repository's default branch (`HEAD`) into the `production` namespace. Both applications enable automatic pruning, self-healing, and namespace creation. All applications target the cluster where Argo CD is installed.

The files under `argocd/` are bootstrap definitions. Reapply them yourself or through a deployment pipeline when they change; pushing changes to these files alone does not update the live Application resources.

Commit and push future manifest changes so Argo CD can reconcile them. Automatic pruning is disabled for the infrastructure application; removing a shared resource from Git requires explicit cleanup. The monitoring application retains its own pruning policy.

```bash
kubectl get applications -n argocd
kubectl get clustersecretstore aws-secrets-manager
kubectl get storageclass gp3
kubectl get pods -n monitoring
```

The gp3 manifest marks itself as the cluster's default StorageClass. Review the existing default when adapting the project to another cluster.

## Verification

```bash
kubectl get clustersecretstore
kubectl get externalsecret -A
kubectl get pods -n databases
kubectl get pods -n production
kubectl get pvc -n databases
kubectl get ingress -n production
```

Check a rollout or inspect a problem with:

```bash
kubectl rollout status deployment/<service-name> -n production
kubectl describe externalsecret <secret-name> -n <namespace>
kubectl describe pod <pod-name> -n <namespace>
kubectl logs <pod-name> -n <namespace>
```

After the ingress receives an ALB address, point the configured DNS hostname to it:

```bash
kubectl get ingress sliders-public-ingress -n production
```

## Updating a service

1. Build and push a new image to the service's ECR repository.
2. Update the image tag or digest in its `deployment.yaml`.
3. Commit and push the manifest change.
4. Let Argo CD reconcile the workload changes.
5. Confirm the deployment rollout.

## Security notes

- Never commit plaintext credentials, Kubernetes Secrets, kubeconfig files or AWS access keys.
- Scope IAM permissions to only the cluster and resources that require them.
- Keep the RabbitMQ management interface private.
- Prefer immutable image tags and enable ECR image scanning.


