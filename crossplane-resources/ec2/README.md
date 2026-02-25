# Crossplane EC2 Guide

This guide explains how to create EC2 instances and networking resources using Crossplane with the AWS EC2 provider.

## Prerequisites

- Kubernetes cluster with Crossplane installed
- IRSA (IAM Roles for Service Accounts) configured
- AWS IAM role for Crossplane
- VPC already created in AWS

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

### Step 2: Install the AWS EC2 Provider

Install the Crossplane AWS EC2 provider which provides the EC2 and networking resource definitions.

```yaml
apiVersion: pkg.crossplane.io/v1
kind: Provider
metadata:
  name: upbound-provider-aws-ec2
spec:
  package: xpkg.upbound.io/upbound/provider-aws-ec2:v2.4.0
  runtimeConfigRef:
    name: aws-config
```

Apply it:

```bash
kubectl apply -f ec2-provider.yaml
```

Wait for the provider to be healthy:

```bash
kubectl get providers.pkg.crossplane.io upbound-provider-aws-ec2
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

### Step 4: Create a Subnet

Create a subnet resource for your EC2 instance.

```yaml
apiVersion: ec2.aws.m.upbound.io/v1beta1
kind: Subnet
metadata:
  name: crossplane-private-subnet
spec:
  forProvider:
    availabilityZone: us-east-1a
    cidrBlock: 10.0.7.0/24
    region: us-east-1
    vpcId: vpc-067dbfd55af6fb0f8
```

Apply it:

```bash
kubectl apply -f new-subnet.yaml
```

Wait for the subnet to be available:

```bash
kubectl get subnets.ec2.aws.m.upbound.io -n default
```

### Step 5: Verify the Subnet Creation

Check the subnet resource status:

```bash
kubectl get subnets.ec2.aws.m.upbound.io -n default
```

Check detailed status:

```bash
kubectl describe subnet crossplane-private-subnet -n default
```

Verify in AWS:

```bash
aws ec2 describe-subnets --subnet-ids <subnet-id>
```

## Configuration Options

### Common Subnet Parameters

| Parameter | Description | Example |
|-----------|-------------|---------|
| `availabilityZone` | AZ for the subnet | `us-east-1a`, `us-east-1b` |
| `cidrBlock` | CIDR block for the subnet | `10.0.7.0/24` |
| `region` | AWS region | `us-east-1` |
| `vpcId` | VPC ID | `vpc-067dbfd55af6fb0f8` |

## Troubleshooting

### Check Provider Status

```bash
kubectl get providers.pkg.crossplane.io
kubectl describe provider upbound-provider-aws-ec2
```

### Check Provider Config

```bash
kubectl get clusterproviderconfigs.aws.m.upbound.io
```

### Check Resource Events

```bash
kubectl get events -n default --field-selector involvedObject.name=crossplane-private-subnet
```

## Cleanup

To delete all resources in reverse order:

```bash
kubectl delete -f new-subnet.yaml
kubectl delete -f ec2-provider.yaml
kubectl delete -f ../clusterProviderConfig.yaml
kubectl delete -f ../deploymentRuntineConfig.yaml
```

## File Reference

| File | Description |
|------|-------------|
| `../deploymentRuntineConfig.yaml` | Service account configuration for providers |
| `ec2-provider.yaml` | AWS EC2 Crossplane provider installation |
| `../clusterProviderConfig.yaml` | AWS authentication configuration |
| `new-subnet.yaml` | EC2 Subnet resource definition |
