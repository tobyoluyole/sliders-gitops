# Sliders Kubernetes GitOps

Kubernetes manifests for deploying the Sliders microservices platform and its supporting services to Amazon EKS.

This repository defines production application workloads, Redis and RabbitMQ data services, AWS Secrets Manager integration, persistent EBS storage, and public HTTPS access through an AWS Application Load Balancer.

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

Application workloads use the `production` namespace. Redis and RabbitMQ use the `databases` namespace.

## Prerequisites

- An existing Amazon EKS cluster
- `kubectl` configured for the target cluster
- AWS CLI access to the target AWS account
- AWS Load Balancer Controller
- External Secrets Operator
- Amazon EBS CSI Driver
- Application images pushed to Amazon ECR
- Required values created in AWS Secrets Manager
- A DNS record and ACM certificate for the public API hostname

The current manifests target AWS region `eu-west-1`.

## Configuration

Review these environment-specific values before deployment:

- ECR account ID, region, repository and image tag in each `deployment.yaml`
- AWS region in `aws-secret-store.yaml`
- Hostname and ACM certificate ARN in `k8s/production/api-gateway/ingress.yaml`
- Replica counts and resource requests
- Redis and RabbitMQ PVC sizes
- AWS Secrets Manager names

The deployments currently use the mutable `latest` image tag. Immutable version tags or image digests are recommended for reproducible releases.

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

### 1. Create the namespaces

```bash
kubectl create namespace production
kubectl create namespace databases
```

Skip a command if its namespace already exists.

### 2. Apply cluster-level resources

```bash
kubectl apply -f gp3-storageclass.yaml
kubectl apply -f aws-secret-store.yaml
```

> The gp3 manifest marks itself as the cluster's default StorageClass. Review any existing default before applying it.

### 3. Deploy Redis and RabbitMQ

```bash
kubectl apply --recursive -f k8s/databases/
```

### 4. Deploy the application

```bash
kubectl apply --recursive -f k8s/production/
```

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
4. Apply it directly or let the GitOps controller reconcile it.
5. Confirm the deployment rollout.

## Security notes

- Never commit plaintext credentials, Kubernetes Secrets, kubeconfig files or AWS access keys.
- Scope IAM permissions to only the cluster and resources that require them.
- Keep the RabbitMQ management interface private.
- Prefer immutable image tags and enable ECR image scanning.

## Contributing

Keep each service in its own directory and follow the existing manifest naming convention. Validate changes before opening a pull request:

```bash
kubectl apply --dry-run=client --recursive -f k8s/
```
