# Crossplane RDS Instance Guide

This guide explains how to create an RDS instance using Crossplane with the AWS RDS provider.

## Prerequisites

- Kubernetes cluster with Crossplane installed
- IRSA (IAM Roles for Service Accounts) configured
- AWS IAM role for Crossplane
- VPC with subnets for the RDS instance

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

### Step 2: Install the AWS RDS Provider

Install the Crossplane AWS RDS provider which provides the RDS instance resource definitions.

```yaml
apiVersion: pkg.crossplane.io/v1
kind: Provider
metadata:
  name: upbound-provider-aws-rds
spec:
  package: xpkg.upbound.io/upbound/provider-aws-rds:v2.4.0
  runtimeConfigRef:
    name: aws-config
```

Apply it:

```bash
kubectl apply -f rds-provider.yaml
```

Wait for the provider to be healthy:

```bash
kubectl get providers.pkg.crossplane.io upbound-provider-aws-rds
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

### Step 4: Create RDS Subnet Group

An RDS subnet group is required for the RDS instance. It defines which subnets the RDS instance can use.

```yaml
apiVersion: rds.aws.m.upbound.io/v1beta1
kind: SubnetGroup
metadata:
  name: small-database-subnet-group
  namespace: default
spec:
  forProvider:
    region: us-east-1
    subnetIds:
      - subnet-075c871cbe1e5305f
      - subnet-0beec03fd70b7302a
```

Apply it:

```bash
kubectl apply -f rds-subnet-group.yaml
```

Wait for the subnet group to be ready:

```bash
kubectl get subnetgroups.rds.aws.m.upbound.io -n default
```

### Step 5: Create the RDS Instance

Create the RDS instance resource with the desired configuration.

```yaml
apiVersion: rds.aws.m.upbound.io/v1beta1
kind: Instance
metadata:
  name: small-database
  namespace: default
spec:
  forProvider:
    identifier: small-database
    region: us-east-1
    engine: postgres
    engineVersion: "17.4"
    instanceClass: db.t3.micro
    allocatedStorage: 20
    dbName: cloudcampdb
    username: cloudcamp
    autoGeneratePassword: true
    passwordSecretRef:
      key: password
      name: example-dbinstance
    backupRetentionPeriod: 1
    backupWindow: 03:00-05:00
    maintenanceWindow: Mon:00:00-Mon:03:00
    multiAz: false
    publiclyAccessible: false
    storageEncrypted: true
    dbSubnetGroupNameRef:
      name: small-database-subnet-group
    skipFinalSnapshot: true
  writeConnectionSecretToRef:
    name: database-credentials-out
```

Apply it:

```bash
kubectl apply -f rds-instance.yaml
```

### Step 6: Verify the RDS Instance Creation

Check the RDS instance resource status:

```bash
kubectl get instances.rds.aws.m.upbound.io -n default
```

Check detailed status:

```bash
kubectl describe instance small-database -n default
```

Verify in AWS:

```bash
aws rds describe-db-instances --db-instance-identifier small-database
```

### Step 7: Get Connection Credentials

After the RDS instance is created, Crossplane stores the connection credentials in a Kubernetes secret:

```bash
kubectl get secret database-credentials-out -n default -o yaml
```

The secret contains:
- `username`: Database username
- `password`: Database password
- `endpoint`: Database endpoint
- `port`: Database port

## Configuration Options

### Common Instance Parameters

| Parameter | Description | Example |
|-----------|-------------|---------|
| `engine` | Database engine | `postgres`, `mysql`, `oracle-ee`, `sqlserver-ee` |
| `engineVersion` | Database version | `17.4`, `15.3`, `8.0.35` |
| `instanceClass` | DB instance class | `db.t3.micro`, `db.r5.large` |
| `allocatedStorage` | Storage in GB | `20`, `100`, `500` |
| `dbName` | Initial database name | `cloudcampdb` |
| `username` | Master username | `admin` |
| `multiAz` | Multi-AZ deployment | `true`, `false` |
| `publiclyAccessible` | Public access | `true`, `false` |
| `storageEncrypted` | Enable encryption | `true`, `false` |

### Backup Configuration

| Parameter | Description | Example |
|-----------|-------------|---------|
| `backupRetentionPeriod` | Days to retain backups | `1`, `7`, `30` |
| `backupWindow` | Backup window | `03:00-05:00` |
| `maintenanceWindow` | Maintenance window | `Mon:00:00-Mon:03:00` |

## Troubleshooting

### Check Provider Status

```bash
kubectl get providers.pkg.crossplane.io
kubectl describe provider upbound-provider-aws-rds
```

### Check Provider Config

```bash
kubectl get clusterproviderconfigs.aws.m.upbound.io
```

### Check Resource Events

```bash
kubectl get events -n default --field-selector involvedObject.name=small-database
```

### Check RDS Subnet Group

```bash
kubectl get subnetgroups.rds.aws.m.upbound.io -n default
kubectl describe subnetgroup small-database-subnet-group -n default
```

### Common Issues

1. **Instance fails to create**: Check subnet group and security groups
2. **Authentication errors**: Verify IRSA configuration and role ARN
3. **Storage issues**: Ensure sufficient IAM permissions for EBS volumes

## Cleanup

To delete all resources in reverse order:

```bash
kubectl delete -f rds-instance.yaml
kubectl delete -f rds-subnet-group.yaml
kubectl delete -f rds-provider.yaml
kubectl delete -f ../clusterProviderConfig.yaml
kubectl delete -f ../deploymentRuntineConfig.yaml
```

## File Reference

| File | Description |
|------|-------------|
| `../deploymentRuntineConfig.yaml` | Service account configuration for providers |
| `rds-provider.yaml` | AWS RDS Crossplane provider installation |
| `../clusterProviderConfig.yaml` | AWS authentication configuration |
| `rds-subnet-group.yaml` | RDS subnet group resource definition |
| `rds-instance.yaml` | RDS instance resource definition |
