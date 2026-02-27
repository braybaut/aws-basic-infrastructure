# Crossplane Go Templating Composition

This folder contains a Crossplane Composition using the Go Templating function for more flexible resource composition with conditionals and loops.

## Prerequisites

- Kubernetes cluster with Crossplane installed (v1.14+)
- `function-go-templating` function installed

## Architecture

Uses Go templates instead of patch-and-transform for:
- Conditional resource creation
- Complex transformations
- More readable templates (similar to Helm)

```
┌─────────────┐     ┌──────────────┐     ┌─────────────┐
│   Claim     │────▶│     XRD      │────▶│ Composition │
│   App       │     │ XApp         │     │ (Go Template)│
└─────────────┘     └──────────────┘     └─────────────┘
                                                   │
                      ┌──────────────────────────────┤
                      │                              │
                ┌─────▼─────┐                 ┌─────▼─────┐
                │Deployment │                 │ Service   │
                └───────────┘                 └───────────┘
                      │                              │
                      │                   ┌──────────┴──────────┐
                      │                   │                     │
                ┌─────▼─────┐      ┌─────▼─────┐       ┌─────▼─────┐
                │    S3     │      │   RDS      │       │SubnetGroup │
                │  Bucket   │      │ Instance   │       │ (optional) │
                └───────────┘      └────────────┘       └────────────┘
```

## Quick Start

### Step 1: Install the Go Templating Function

```bash
kubectl apply -f function.yaml
```

Or manually:

```yaml
apiVersion: pkg.crossplane.io/v1
kind: Function
metadata:
  name: crossplane-contrib-function-go-templating
spec:
  package: xpkg.crossplane.io/crossplane-contrib/function-go-templating:v0.5.0
```

Wait for the function to be healthy:

```bash
kubectl get functions.pkg.crossplane.io
```

### Step 2: Apply the XRD and Composition

```bash
kubectl apply -f compositeResourceDefinition.yaml
kubectl apply -f composition-go-templating.yaml
```

Wait for the XRD to be established:

```bash
kubectl get xrd apps.api.nequi.io
```

### Step 3: Create an Application

```bash
kubectl apply -f composition-app.yaml
```

## API Reference

### Spec Fields

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `image` | string | - | **Required.** Container image |
| `replicas` | integer | 2 | Number of replicas |
| `port` | integer | 80 | Container port |
| `resources` | object | - | Compute resources |
| `resources.requests.memory` | string | "50Mi" | Memory request |
| `resources.requests.cpu` | string | "100m" | CPU request |
| `resources.limits.memory` | string | - | Memory limit |
| `resources.limits.cpu` | string | - | CPU limit |
| `createService` | boolean | true | Create Kubernetes Service |
| `createBucket` | boolean | false | Create S3 Bucket |
| `bucketRegion` | string | "us-east-1" | S3 region |
| `bucketTags` | object | - | S3 tags |
| `createDatabase` | boolean | false | Create RDS Instance |
| `dbConfig` | object | - | Database configuration |

### dbConfig Fields

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `user` | string | - | Database username |
| `identifier` | string | - | Database identifier |
| `region` | string | "us-east-1" | Database region |
| `engine` | string | "postgres" | Database engine |
| `engineVersion` | string | - | Engine version |
| `instanceClass` | string | "db.t3.micro" | Instance class |
| `allocatedStorage` | integer | 20 | Storage in GB |
| `name` | string | "cloudcampdb" | Database name |
| `backupRetentionPeriod` | integer | 1 | Backup retention days |
| `publiclyAccessible` | boolean | false | Public access |
| `subnetIds` | array | - | Subnet IDs for RDS subnet group |

## Examples

### Minimal App (Kubernetes only)

```yaml
apiVersion: api.nequi.io/v1
kind: App
metadata:
  name: my-app
spec:
  image: nginx
```

### Full App (All resources)

```yaml
apiVersion: api.nequi.io/v1
kind: App
metadata:
  name: my-app
spec:
  image: nginx
  replicas: 3
  port: 8080
  resources:
    requests:
      memory: "128Mi"
      cpu: "250m"
    limits:
      memory: "256Mi"
      cpu: "500m"
  createService: true
  createBucket: true
  bucketRegion: us-west-2
  bucketTags:
    Environment: production
    Team: platform
  createDatabase: true
  dbConfig:
    user: admin
    identifier: my-app-db
    region: us-west-2
    engine: postgres
    engineVersion: "17.4"
    instanceClass: db.t3.micro
    allocatedStorage: 20
    name: myappdb
    backupRetentionPeriod: 7
    publiclyAccessible: false
    subnetIds:
      - subnet-086c285de4d1ea0e5
      - subnet-0805fff946bdfcbc4
```

### App without Service

```yaml
apiVersion: api.nequi.io/v1
kind: App
metadata:
  name: my-app
spec:
  image: nginx
  createService: false
```

### App with Database only

```yaml
apiVersion: api.nequi.io/v1
kind: App
metadata:
  name: my-app
spec:
  image: nginx
  createService: false
  createBucket: false
  createDatabase: true
  dbConfig:
    user: myuser
    identifier: mydb
```

### App with Database and SubnetGroup

```yaml
apiVersion: api.nequi.io/v1
kind: App
metadata:
  name: my-app
spec:
  image: nginx
  createService: false
  createDatabase: true
  dbConfig:
    user: admin
    identifier: my-db
    region: us-east-1
    engine: postgres
    instanceClass: db.t3.micro
    subnetIds:
      - subnet-086c285de4d1ea0e5
      - subnet-0805fff946bdfcbc4
```

**Note:** When `subnetIds` are provided, Crossplane will automatically create an RDS SubnetGroup with those subnet IDs.

## Troubleshooting

### Check XRD Status

```bash
kubectl get xrd apps.api.nequi.io
kubectl describe xrd apps.api.nequi.io
```

### Check Composition

```bash
kubectl get composition app-yaml
kubectl describe composition app-yaml
```

### Check Events

```bash
kubectl get events -n default --field-selector involvedObject.name=my-app
```

### Render Locally

Use Crossplane CLI to render locally:

```bash
crossplane beta render composition-go-templating.yaml compositeResourceDefinition.yaml function.yaml
```

## File Reference

| File | Description |
|------|-------------|
| `function.yaml` | Go Templating Function installation |
| `compositeResourceDefinition.yaml` | XRD definition for App composite resource |
| `composition-go-templating.yaml` | Go template composition |
| `composition-app.yaml` | Example App resources |

## Differences from Patch & Transform

| Feature | Patch & Transform | Go Templating |
|---------|-------------------|---------------|
| Conditionals | Requires extra function | Native if/else |
| Loops | Not supported | Native range |
| Defaults | Via patches | Via `default` function |
| Readability | Verbose patches | Helm-like templates |
| Complexity | Low | Medium |
