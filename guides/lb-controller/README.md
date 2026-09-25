# AWS Load Balancer Controller — Ingress & Gateway API

Step-by-step guide to install the **AWS Load Balancer Controller v2.14.1** on an Amazon EKS cluster, following the
official AWS documentation
([Install AWS Load Balancer Controller with manifests](https://docs.aws.amazon.com/eks/latest/userguide/lbc-manifest.html)).
It also covers the `IngressClass`, the `GatewayClass`, and a demo application (2048) exposed both through an
**Ingress** and through **Gateway API**.

Everything is installed with `kubectl apply` — **`eksctl` is not required**. AWS credentials are provided with
**IRSA** (IAM Roles for Service Accounts) instead of node instance profiles.

### Official references

| Topic | Link |
| --- | --- |
| **Install the LBC with manifests (this guide follows it)** | https://docs.aws.amazon.com/eks/latest/userguide/lbc-manifest.html |
| Create an IAM OIDC provider for your cluster | https://docs.aws.amazon.com/eks/latest/userguide/enable-iam-roles-for-service-accounts.html |
| IAM roles for service accounts (IRSA) | https://docs.aws.amazon.com/eks/latest/userguide/iam-roles-for-service-accounts.html |
| Route traffic with ALB Ingress | https://docs.aws.amazon.com/eks/latest/userguide/alb-ingress.html |
| Full controller documentation (project site) | https://kubernetes-sigs.github.io/aws-load-balancer-controller/latest/deploy/installation/ |
| Ingress annotations reference | https://kubernetes-sigs.github.io/aws-load-balancer-controller/latest/guide/ingress/annotations/ |
| Gateway API guide (L7 routing) | https://kubernetes-sigs.github.io/aws-load-balancer-controller/latest/guide/gateway/l7gateway/ |
| `LoadBalancerConfiguration` reference | https://kubernetes-sigs.github.io/aws-load-balancer-controller/latest/guide/gateway/loadbalancerconfig/ |
| Release assets for v2.14.1 | https://github.com/kubernetes-sigs/aws-load-balancer-controller/releases/tag/v2.14.1 |

> **Version pinning:** every download below targets **v2.14.1**. Keep the version consistent across the IAM policy,
> the full manifest and the IngressClass manifest.

---

## Table of contents

1. [Manifests in this repo](#1-manifests-in-this-repo)
2. [Prerequisites](#2-prerequisites)
3. [Step 1 — Configure IAM](#3-step-1--configure-iam)
4. [Step 2 — Install cert-manager](#4-step-2--install-cert-manager)
5. [Step 3 — Download and install the controller](#5-step-3--download-and-install-the-controller)
6. [Step 4 — Download and apply the IngressClass](#6-step-4--download-and-apply-the-ingressclass)
7. [Step 5 — Deploy the demo app + Ingress](#7-step-5--deploy-the-demo-app--ingress)
8. [Step 6 — Gateway API: GatewayClass](#8-step-6--gateway-api-gatewayclass)
9. [Step 7 — Gateway API: LoadBalancerConfiguration, Gateway and HTTPRoute](#9-step-7--gateway-api-loadbalancerconfiguration-gateway-and-httproute)
10. [Verify / Cleanup](#10-verify--cleanup)
11. [Extra manifests in this folder (IRSA & EBS demos)](#11-extra-manifests-in-this-folder-irsa--ebs-demos)
12. [Troubleshooting](#12-troubleshooting)

---

## 1. Manifests in this repo

Every file here is a **copy of the official release assets** plus the local app manifests, so the guide can be
followed offline. When in doubt, re-download the official file from the links above.

| File | Origin | Purpose |
| --- | --- | --- |
| `iam_policy.json` | `docs/install/iam_policy.json` @ `v2.14.1` | IAM policy that lets the controller call the AWS APIs. |
| `sa-lb-controller.yaml` | step 1 of the official guide | `ServiceAccount` in `kube-system` annotated with the IRSA role ARN. |
| `v2_14_1_full.yaml` | release asset `v2_14_1_full.yaml` | CRDs, RBAC, webhook, Service and Deployment of the controller. Already edited: the `ServiceAccount` block was **removed** (step 3.2) and `--cluster-name`, `--ingress-class` and `--aws-vpc-id` were set. |
| `v2.14.1_ingclass.yaml` | release asset `v2_14_1_ingclass.yaml` (renamed) | `IngressClassParams` + `IngressClass` named `alb`. |
| `alb-gatewayclass.yaml` | — | `GatewayClass` named `aws-alb-gateway-class` used by the Gateway API manifests. |
| `2048_full.yaml` | — | Demo app: namespace `game-2048`, Deployment, Service and `Ingress` (`alb`). |
| `gateway-api.yaml` | — | `LoadBalancerConfiguration`, `Gateway` and `HTTPRoute` for the same app. |
| `sa.yaml`, `pod-sa.yaml` | — | IRSA demo: ServiceAccounts in `development` and a Pod bound to one of them. |
| `storaclass.yaml`, `pvc.yaml`, `pod-pvc.yaml` | — | EBS CSI demo: `StorageClass`, `PersistentVolumeClaim` and a Pod writing to it. |

If you already cloned this repository you can skip the `curl` commands in steps 3 and 4 and apply the local files
instead:

```bash
kubectl apply -f sa-lb-controller.yaml
kubectl apply -f v2_14_1_full.yaml
kubectl apply -f v2.14.1_ingclass.yaml
```

---

## 2. Prerequisites

- An EKS cluster (this repo was tested with cluster `eks-bray`, region `us-east-1`).
- `kubectl` configured for the target cluster.
- AWS CLI configured with permissions to create IAM policies/roles and to read the cluster description.
- The node security group must allow **TCP 9443** from the Kubernetes control plane so the controller webhook works.
- Public subnets in the VPC that map public IPs to instances (required by `internet-facing` ALBs).
- Kubernetes Gateway API CRDs — v2.14.1 ships the `LoadBalancerConfiguration` CRD, but the core Gateway API CRDs
  must be installed separately (step 6).

Set your values once and reuse them in every command:

```bash
export CLUSTER=eks-bray
export REGION=us-east-1
export ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
aws eks update-kubeconfig --name $CLUSTER --region $REGION
```

Throughout the guide, `{{111122223333}}` means your **account ID** and `{{region-code}}` your **AWS Region**.

---

## 3. Step 1 — Configure IAM

> Source: [Install AWS Load Balancer Controller with manifests → Step 1: Configure IAM](https://docs.aws.amazon.com/eks/latest/userguide/lbc-manifest.html#lbc-iam)

The IAM policy and the role `AmazonEKSLoadBalancerControllerRole` can be reused across every EKS cluster in the
same account, but the **trust policy is cluster-specific** because it references that cluster's OIDC provider.

### 1.1 Download the IAM policy

```bash
curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.14.1/docs/install/iam_policy.json
```

> AWS GovCloud (US): use `.../v2.14.1/docs/install/iam_policy_us-gov.json` and rename it to `iam_policy.json`.
> China regions: `.../v2.14.1/docs/install/iam_policy_cn.json`.

The copy in this repository (`iam_policy.json`) is the same file.

### 1.2 Create the IAM policy

```bash
aws iam create-policy \
  --policy-name AWSLoadBalancerControllerIAMPolicy \
  --policy-document file://iam_policy.json
```

Note the returned ARN: `arn:aws:iam::{{111122223333}}:policy/AWSLoadBalancerControllerIAMPolicy`.

> In the AWS console this policy shows a warning for the **ELB** service but not for **ELB v2**. That is expected —
> some actions exist only for ELB v2. You can ignore it.

### 1.3 Check the OIDC provider of your cluster

```bash
oidc_id=$(aws eks describe-cluster --name my-cluster \
  --query "cluster.identity.oidc.issuer" --output text | cut -d '/' -f 5)
echo $oidc_id

aws iam list-open-id-connect-providers | grep $oidc_id | cut -d "/" -f4
```

If the second command prints an OIDC provider ID, the provider already exists. If it prints nothing, create it
(reference: [Create an IAM OIDC provider for your cluster](https://docs.aws.amazon.com/eks/latest/userguide/enable-iam-roles-for-service-accounts.html)):

```bash
aws iam create-open-id-connect-provider \
  --url https://oidc.eks.{{region-code}}.amazonaws.com/id/{{oidc_id}} \
  --client-id-list sts.amazonaws.com \
  --thumbprint-list 9e99a48a9960b14926bb7f3b02e22da2b0ab7280
```

### 1.4 Create the trust policy

```bash
cat > load-balancer-role-trust-policy.json <<EOF
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "Federated": "arn:aws:iam::{{111122223333}}:oidc-provider/{{oidc_id}}"
            },
            "Action": "sts:AssumeRoleWithWebIdentity",
            "Condition": {
                "StringEquals": {
                    "{{oidc_id}}:aud": "sts.amazonaws.com",
                    "{{oidc_id}}:sub": "system:serviceaccount:kube-system:aws-load-balancer-controller"
                }
            }
        }
    ]
}
EOF
```

The `sub` condition restricts the role to **only** the `aws-load-balancer-controller` ServiceAccount in
`kube-system`. Do not relax it.

### 1.5 Create the IAM role and attach the policy

```bash
aws iam create-role \
  --role-name AmazonEKSLoadBalancerControllerRole \
  --assume-role-policy-document file://"load-balancer-role-trust-policy.json"

aws iam attach-role-policy \
  --policy-arn arn:aws:iam::{{111122223333}}:policy/AWSLoadBalancerControllerIAMPolicy \
  --role-name AmazonEKSLoadBalancerControllerRole
```

### 1.6 Create the Kubernetes ServiceAccount

```bash
cat > aws-load-balancer-controller-service-account.yaml <<EOF
apiVersion: v1
kind: ServiceAccount
metadata:
  labels:
    app.kubernetes.io/component: controller
    app.kubernetes.io/name: aws-load-balancer-controller
  name: aws-load-balancer-controller
  namespace: kube-system
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::{{111122223333}}:role/AmazonEKSLoadBalancerControllerRole
EOF

kubectl apply -f aws-load-balancer-controller-service-account.yaml
```

This is the same manifest as `sa-lb-controller.yaml` in this repository. If you use the local copy, replace the
account ID and apply it, then confirm the annotation landed:

```bash
kubectl apply -f sa-lb-controller.yaml
kubectl -n kube-system get sa aws-load-balancer-controller \
  -o jsonpath='{.metadata.annotations.eks\.amazonaws\.com/role-arn}{"\n"}'
```

> Keep this step. In step 3.2 we **remove** the `ServiceAccount` from the controller manifest, so the annotated
> ServiceAccount created here is the one that will be used.

---

## 4. Step 2 — Install cert-manager

> Source: [Install AWS Load Balancer Controller with manifests → Step 2: Install cert-manager](https://docs.aws.amazon.com/eks/latest/userguide/lbc-manifest.html#lbc-cert)

The manifest install ships `Certificate` and `Issuer` resources that **require the cert-manager CRDs**, so
cert-manager must be installed before the controller manifest is applied. (Skip this step if you install the
controller with the Helm chart — the chart manages the webhook certificates itself.)

```bash
kubectl apply \
    --validate=false \
    -f https://github.com/cert-manager/cert-manager/releases/download/v1.13.5/cert-manager.yaml

kubectl -n cert-manager rollout status deploy --timeout=300s
kubectl get pods -n cert-manager
```

If your nodes cannot reach `quay.io`, download the manifest, replace the registry with your own (for example
Amazon ECR) and apply it locally:

```bash
curl -Lo cert-manager.yaml https://github.com/cert-manager/cert-manager/releases/download/v1.13.5/cert-manager.yaml
sed -i.bak -e 's|quay.io|{{111122223333}}.dkr.ecr.{{region-code}}.amazonaws.com|' ./cert-manager.yaml
kubectl apply --validate=false -f ./cert-manager.yaml
```

---

## 5. Step 3 — Download and install the controller

> Source: [Install AWS Load Balancer Controller with manifests → Step 3](https://docs.aws.amazon.com/eks/latest/userguide/lbc-manifest.html#lbc-install)

### 3.1 Download the official release manifest

```bash
curl -Lo v2_14_1_full.yaml \
  https://github.com/kubernetes-sigs/aws-load-balancer-controller/releases/download/v2.14.1/v2_14_1_full.yaml
```

### 3.2 Remove the `ServiceAccount` block

The release manifest contains its own `ServiceAccount` **without** the IRSA annotation, so applying it would
overwrite the ServiceAccount from step 1.6. Delete lines 764–772 of the freshly downloaded file:

```bash
sed -i.bak -e '764,772d' ./v2_14_1_full.yaml
```

If you downloaded a different version, open the file and remove this block by hand:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  labels:
    app.kubernetes.io/component: controller
    app.kubernetes.io/name: aws-load-balancer-controller
  name: aws-load-balancer-controller
  namespace: kube-system
---
```

Removing it also means your IRSA ServiceAccount survives if you later delete the controller.

> The copy in this repository (`v2_14_1_full.yaml`) already has this block removed.

### 3.3 Set your cluster name in the `Deployment`

```bash
sed -i.bak -e 's|your-cluster-name|my-cluster|' ./v2_14_1_full.yaml
```

```yaml
spec:
  containers:
    - args:
        - --cluster-name=your-cluster-name     # <-- replaced with the name of your cluster
        - --ingress-class=alb
```

Optionally pin the VPC and Region (useful with Fargate, Hybrid Nodes, or restricted IMDS access):

```bash
aws eks describe-cluster --name my-cluster \
  --query "cluster.resourcesVpcConfig.vpcId" --output text
```

```yaml
    - args:
        - --cluster-name=my-cluster
        - --ingress-class=alb
        - --aws-vpc-id=vpc-xxxxxxxx
        - --aws-region={{region-code}}
```

`--ingress-class=alb` must match the name of the `IngressClass` in step 4. When `--aws-vpc-id` is set, the
controller stops discovering the VPC from subnet tags.

If your nodes cannot pull from `public.ecr.aws`, mirror the image
`public.ecr.aws/eks/aws-load-balancer-controller:v2.14.1` into your own registry and rewrite the reference:

```bash
sed -i.bak -e 's|public.ecr.aws/eks/aws-load-balancer-controller|{{111122223333}}.dkr.ecr.{{region-code}}.amazonaws.com/eks/aws-load-balancer-controller|' ./v2_14_1_full.yaml
```

### 3.4 Apply and verify

```bash
kubectl apply -f v2_14_1_full.yaml
kubectl -n kube-system get deployment aws-load-balancer-controller
```

Expected output (the manifest install runs a single replica; `2/2` means you used Helm):

```
NAME                           READY   UP-TO-DATE   AVAILABLE   AGE
aws-load-balancer-controller   1/1     1            1           60s
```

Then check that the controller is reconciling:

```bash
kubectl -n kube-system rollout status deployment/aws-load-balancer-controller --timeout=300s
kubectl -n kube-system logs deployment/aws-load-balancer-controller --tail=20
kubectl -n kube-system get svc aws-load-balancer-webhook-service
```

The CRDs the controller needs:

```bash
kubectl get crd | grep -E "elbv2.k8s.aws|gateway.k8s.aws"
# ingressclassparams.elbv2.k8s.aws
# targetgroupbindings.elbv2.k8s.aws
# loadbalancerconfigurations.gateway.k8s.aws   <- required for Gateway API
```

Gateway API support is GA since controller v2.13.

> Before moving on, the cluster must meet the requirements in the official guides
> [ALB Ingress](https://docs.aws.amazon.com/eks/latest/userguide/alb-ingress.html) and
> [Network Load Balancing](https://docs.aws.amazon.com/eks/latest/userguide/network-load-balancing.html).

---

## 6. Step 4 — Download and apply the IngressClass

> Source: [Install AWS Load Balancer Controller with manifests → Step 3 (last two items)](https://docs.aws.amazon.com/eks/latest/userguide/lbc-manifest.html#lbc-install)

The release ships a default `IngressClass` named `alb` together with its `IngressClassParams`. Download it and
apply it exactly as published:

```bash
curl -Lo v2_14_1_ingclass.yaml \
  https://github.com/kubernetes-sigs/aws-load-balancer-controller/releases/download/v2.14.1/v2_14_1_ingclass.yaml

kubectl apply -f v2_14_1_ingclass.yaml
kubectl get ingressclass alb
```

```yaml
spec:
  controller: ingress.k8s.aws/alb        # the controller only watches Ingresses of this class
  parameters:
    apiGroup: elbv2.k8s.aws
    kind: IngressClassParams
    name: alb
```

Keep the name `alb` in sync with `--ingress-class=alb` from step 3, otherwise Ingresses are silently ignored.

> This repository ships the same file named `v2.14.1_ingclass.yaml` (with dots), so
> `kubectl apply -f v2.14.1_ingclass.yaml` is equivalent.

---

## 7. Step 5 — Deploy the demo app + Ingress

`2048_full.yaml` creates the namespace `game-2048`, a 5-replica Deployment, a `NodePort` Service and an `Ingress`
of class `alb`:

```bash
kubectl apply -f 2048_full.yaml
kubectl -n game-2048 rollout status deployment/deployment-2048 --timeout=180s
kubectl -n game-2048 get ingress
```

Key annotations used in the Ingress (full list in the
[annotations reference](https://kubernetes-sigs.github.io/aws-load-balancer-controller/latest/guide/ingress/annotations/)):

| Annotation | Value | Meaning |
| --- | --- | --- |
| `alb.ingress.kubernetes.io/scheme` | `internet-facing` | Public ALB (use `internal` for a private one). |
| `alb.ingress.kubernetes.io/target-type` | `ip` | Register pod IPs directly (recommended). |
| `alb.ingress.kubernetes.io/subnets` | 3 subnet IDs | Where the ALB nodes are placed. Replace with your own public subnets. |

```bash
# DNS name of the generated ALB
kubectl -n game-2048 get ingress ingress-2048 \
  -o jsonpath='{.status.loadBalancer.ingress[0].hostname}{"\n"}'

# Confirm the ALB exists in AWS (k8s-<hash>.<region>.elb.amazonaws.com)
aws elbv2 describe-load-balancers \
  --query 'LoadBalancers[].{Name:LoadBalancerName,DNS:DNSName,State:State.Code}'

# Confirm the target group has healthy targets
aws elbv2 describe-target-health --target-group-arn <TG-ARN>
```

Watch the reconciliation in the controller log:

```bash
kubectl -n kube-system logs deployment/aws-load-balancer-controller -f | grep -i 2048
```

---

## 8. Step 6 — Gateway API: GatewayClass

First install the **Gateway API CRDs** (published upstream, not bundled with the controller):

```bash
kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.4.0/standard-install.yaml
kubectl get crd | grep gateway.networking.k8s.io    # GatewayClass, Gateways, HTTPRoutes, ...
```

Then create the `GatewayClass` that points at the controller:

```bash
kubectl apply -f alb-gatewayclass.yaml
kubectl get gatewayclass aws-alb-gateway-class
```

```yaml
apiVersion: gateway.networking.k8s.io/v1beta1
kind: GatewayClass
metadata:
  name: aws-alb-gateway-class
spec:
  controllerName: gateway.k8s.aws/alb    # provided by the AWS Load Balancer Controller
```

`ACCEPTED = True` means the controller (v2.13+) picked it up:

```bash
kubectl get gatewayclass aws-alb-gateway-class \
  -o jsonpath='{.status.conditions[?(@.type=="Accepted")].status}{"\n"}'
```

Reference: [Gateway API guide](https://kubernetes-sigs.github.io/aws-load-balancer-controller/latest/guide/gateway/gateway/).

---

## 9. Step 7 — Gateway API: LoadBalancerConfiguration, Gateway and HTTPRoute

`gateway-api.yaml` contains three resources for the existing `game-2048` app:

1. **`LoadBalancerConfiguration`** (`gateway.k8s.aws/v1`) — the AWS-specific parameters (scheme, ALB name, subnets).
2. **`Gateway`** — references the `LoadBalancerConfiguration` through `infrastructure.parametersRef` and declares
   an HTTP listener on port 80.
3. **`HTTPRoute`** — routes traffic to the existing `service-2048` Service on port 80.

The namespace must already exist, so apply `2048_full.yaml` (step 5) first, or create it explicitly:

```bash
kubectl create namespace game-2048 --dry-run=client -o yaml | kubectl apply -f -
kubectl apply -f gateway-api.yaml
```

The subnets in the `LoadBalancerConfiguration` are selected by **tag name** (`public-us-east-1a/b/c`), so make sure
your subnets carry that `Name` tag (this is how the controller discovers subnets when `--aws-vpc-id` is set):

```bash
aws ec2 describe-subnets \
  --filters "Name=tag:Name,Values=public-us-east-1a,public-us-east-1b,public-us-east-1c" \
  --query 'Subnets[].{Id:SubnetId,AZ:AvailabilityZone,Name:Tags[?Key==`Name`].Value|[0]}' --output table
```

Watch it reconcile:

```bash
kubectl -n game-2048 get gateway,httproute,loadbalancerconfiguration
kubectl -n game-2048 describe gateway my-alb-gateway
kubectl -n game-2048 get gateway my-alb-gateway -o jsonpath='{.status.addresses[0].value}{"\n"}'
kubectl -n kube-system logs deployment/aws-load-balancer-controller -f | grep -i gateway
```

Expected status: the `Gateway` reports `Accepted: True` and `Programmed: True` with an address (the ALB DNS name),
and the `HTTPRoute` reports `Accepted: True` and `ResolvedRefs: True`.

Clean up this stack (also deletes the ALB and its security groups):

```bash
kubectl delete -f gateway-api.yaml
```

---

## 10. Verify / Cleanup

Quick end-to-end verification:

```bash
# 1. Controller is running
kubectl -n kube-system get deployment aws-load-balancer-controller

# 2. IngressClass and GatewayClass are accepted
kubectl get ingressclass alb
kubectl get gatewayclass aws-alb-gateway-class

# 3. Resources are synced
kubectl -n game-2048 get ingress ingress-2048
kubectl -n game-2048 get targetgroupbindings.elbv2.k8s.aws
```

Full cleanup, **in this order** so the controller deletes the AWS resources before it disappears:

```bash
kubectl delete -f gateway-api.yaml
kubectl delete -f 2048_full.yaml
kubectl delete -f v2.14.1_ingclass.yaml          # or v2_14_1_ingclass.yaml
kubectl delete -f alb-gatewayclass.yaml
kubectl delete -f v2_14_1_full.yaml
kubectl delete -f sa-lb-controller.yaml

aws iam detach-role-policy \
  --role-name AmazonEKSLoadBalancerControllerRole \
  --policy-arn arn:aws:iam::{{111122223333}}:policy/AWSLoadBalancerControllerIAMPolicy
aws iam delete-role --role-name AmazonEKSLoadBalancerControllerRole
aws iam delete-policy --policy-arn arn:aws:iam::{{111122223333}}:policy/AWSLoadBalancerControllerIAMPolicy
```

> Deleting the CRDs (`kubectl delete -f v2_14_1_full.yaml`) also removes them from the API server, so delete every
> Ingress/Gateway first or the ALBs will be orphaned in your account.

---

## 11. Extra manifests in this folder (IRSA & EBS demos)

These are not part of the load balancer setup, but they exercise the same IRSA pattern plus the EBS CSI driver.

### IRSA: ServiceAccounts and a Pod that uses one

```bash
kubectl create namespace development        # must exist first
kubectl apply -f sa.yaml
kubectl -n development get sa
```

- `build-robot` — no annotation, the default (no AWS access).
- `frontend-api` — annotated with `arn:aws:iam::314567760063:role/eks-access-s3`, so any Pod using it gets
  temporary S3 credentials from that role.

```bash
kubectl apply -f pod-sa.yaml
kubectl get pod my-pod -o jsonpath='{.spec.serviceAccountName}{"\n"}'
```

Verify the credentials are actually projected into the Pod:

```bash
kubectl exec my-pod -- env | grep AWS
kubectl get pod my-pod -o jsonpath='{.spec.volumes[*].name}{"\n"}'
```

### EBS CSI: StorageClass, PVC and a Pod mounting it

```bash
kubectl apply -f storaclass.yaml      # StorageClass ebs-sc (provisioner ebs.csi.aws.com)
kubectl apply -f pvc.yaml             # PVC ebs-claim-cloudcamp, 10Gi, RWO
kubectl get pvc ebs-claim-cloudcamp  # stays Pending until a Pod is scheduled (WaitForFirstConsumer)
```

```bash
kubectl apply -f pod-pvc.yaml
kubectl logs app-v2                   # appends a timestamp every 5s
kubectl exec app-v2 -- cat /data/out.txt
```

The PVC becomes `Bound` and the EBS volume is created in the AZ of the node where the Pod landed. Requires the
**Amazon EBS CSI driver** add-on in the cluster.

---

## 12. Troubleshooting

| Symptom | Likely cause / fix |
| --- | --- |
| `no matches for kind "Certificate"` when applying the controller | cert-manager CRDs are missing — install step 4 first. |
| Ingress/Gateway ignored, no ALB created | IngressClass name does not match `--ingress-class`, or the `GatewayClass` uses the wrong `controllerName` (`gateway.k8s.aws/alb`). |
| `CreateContainerConfigError: no valid IAM Roles attached` | The ServiceAccount annotation or the trust policy `sub` condition is wrong. Re-check steps 1.3–1.6, and make sure the `ServiceAccount` block was removed from the full manifest (3.2). |
| `AccessDenied` on EC2/ELBv2 calls in the controller log | `AWSLoadBalancerControllerIAMPolicy` is not attached to the role, or the role has a permissions boundary. |
| ALB stays `provisioning` / targets unhealthy | Security group must allow 80/443 from the VPC CIDR, and nodes need NTP/clock sync. |
| `Failed to provision` with a subnet error | The subnets in the annotation / `LoadBalancerConfiguration` are not in the cluster VPC, or lack public IP mapping. |
| `Gateway` never becomes `Programmed` | Check `kubectl describe gateway` events, confirm the Gateway API CRDs are installed, and that `loadbalancerconfigurations.gateway.k8s.aws` exists. |
| Webhook errors while applying the Ingress | The controller is not ready: `kubectl -n kube-system rollout status deploy/aws-load-balancer-controller`. |
| Upgrade notes | The controller does not auto-update. Follow the [upgrade guide](https://kubernetes-sigs.github.io/aws-load-balancer-controller/latest/deploy/installation/#create-update-strategy) and keep the version pinned across all three assets. |

Useful logs:

```bash
kubectl -n kube-system logs deployment/aws-load-balancer-controller --tail=100 -f
kubectl -n cert-manager logs deployment/cert-manager
```
