# Crossplane Resources

This directory contains Crossplane resource definitions for AWS services and composite resources.

## Directory Structure

| Folder | Description |
|--------|-------------|
| [bucket/](bucket/) | S3 Bucket provisioning |
| [ec2/](ec2/) | EC2 instances and networking |
| [rds/](rds/) | RDS database instances |
| [composite/](composite/) | Composite resources (XRD + Composition) |

## Shared Resources

These files are used across all AWS providers:

| File | Description |
|------|-------------|
| `deploymentRuntineConfig.yaml` | Service account configuration for providers |
| `clusterProviderConfig.yaml` | AWS authentication (IRSA) configuration |

## Quick Start

Choose a resource type to provision:

### S3 Bucket
```bash
cd bucket/
kubectl apply -f s3-provider.yaml
kubectl apply -f cloudcamp-bucket.yaml
```

### EC2
```bash
cd ec2/
kubectl apply -f ec2-provider.yaml
kubectl apply -f new-subnet.yaml
```

### RDS
```bash
cd rds/
kubectl apply -f rds-provider.yaml
kubectl apply -f rds-subnet-group.yaml
kubectl apply -f rds-instance.yaml
```

### Composite Resources
```bash
cd composite/
kubectl apply -f function-provider.yaml
kubectl apply -f compositeResourceDefinition.yaml
kubectl apply -f composition.yaml
kubectl apply -f composition-app.yaml
```
