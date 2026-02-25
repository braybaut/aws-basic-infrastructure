# Crossplane S3 Bucket Guide

This guide explains how to create an S3 bucket using Crossplane with the AWS S3 provider.

## Prerequisites

- Kubernetes cluster with Crossplane installed
- IRSA (IAM Roles for Service Accounts) configured
- AWS IAM role for Crossplane

## Step-by-Step Guide

### Step 1: Create Deployment Runtime Config

The DeploymentRuntimeConfig configures the service account that Crossplane providers will use.

```yaml
apiVersion: pkg.crossplane.io/v1beta1
kind: DeploymentRuntimeConfig
metadata:
  name: aws-config
spec:
  serviceAccountTemplate:
    metadata:
      name: crossplane
```

Apply it:

```bash
kubectl apply -f ../deploymentRuntineConfig.yaml
```

### Step 2: Install the AWS S3 Provider

Install the Crossplane AWS S3 provider which provides the S3 bucket resource definitions.

```yaml
apiVersion: pkg.crossplane.io/v1
kind: Provider
metadata:
  name: crossplane-contrib-provider-aws-s3
spec:
  package: xpkg.crossplane.io/crossplane-contrib/provider-aws-s3:v2.0.0
  runtimeConfigRef:
    name: aws-config
```

Apply it:

```bash
kubectl apply -f s3-provider.yaml
```

Wait for the provider to be healthy:

```bash
kubectl get providers.pkg.crossplane.io crossplane-contrib-provider-aws-s3
```

### Step 3: Create Cluster Provider Config

The ClusterProviderConfig defines how Crossplane authenticates with AWS using IRSA (WebIdentity).

```yaml
apiVersion: aws.m.upbound.io/v1beta1
kind: ClusterProviderConfig
metadata:
  name: default
spec:
  credentials:
    source: WebIdentity
    webIdentity:
      roleARN: arn:aws:iam::620525694764:role/crossplane
```

Apply it:

```bash
kubectl apply -f ../clusterProviderConfig.yaml
```

### Step 4: Create the S3 Bucket

Create the S3 bucket resource. Uncomment the `providerConfigRef` section if needed.

```yaml
apiVersion: s3.aws.m.upbound.io/v1beta1
kind: Bucket
metadata:
  namespace: default
  name: cloudcamp-test-crossplane-v3
  labels:
    bucket-name: cloudcamp-test-crossplane-v3
spec:
  forProvider:
    region: us-east-1
#  providerConfigRef:
#    name: default
#    kind: ClusterProviderConfig
```

Apply it:

```bash
kubectl apply -f cloudcamp-bucket.yaml
```

### Step 5: Verify the Bucket Creation

Check the bucket resource status:

```bash
kubectl get buckets.s3.aws.m.upbound.io -n default
```

Check detailed status:

```bash
kubectl describe bucket cloudcamp-test-crossplane-v3 -n default
```

Verify in AWS:

```bash
aws s3 ls | grep cloudcamp-test-crossplane-v3
```

## Troubleshooting

### Check Provider Status

```bash
kubectl get providers.pkg.crossplane.io
kubectl describe provider crossplane-contrib-provider-aws-s3
```

### Check Provider Config

```bash
kubectl get clusterproviderconfigs.aws.m.upbound.io
```

### Check Resource Events

```bash
kubectl get events -n default --field-selector involvedObject.name=cloudcamp-test-crossplane-v3
```

## Cleanup

To delete all resources in reverse order:

```bash
kubectl delete -f cloudcamp-bucket.yaml
kubectl delete -f s3-provider.yaml
kubectl delete -f ../clusterProviderConfig.yaml
kubectl delete -f ../deploymentRuntineConfig.yaml
```

## File Reference

| File | Description |
|------|-------------|
| `../deploymentRuntineConfig.yaml` | Service account configuration for providers |
| `s3-provider.yaml` | AWS S3 Crossplane provider installation |
| `../clusterProviderConfig.yaml` | AWS authentication configuration |
| `cloudcamp-bucket.yaml` | S3 bucket resource definition |
