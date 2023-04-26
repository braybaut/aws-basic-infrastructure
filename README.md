# aws-basic-infrastructure
Terraform code to deploy a basic aws infrastructure

## Usage

* set your AWS credentials 
* set a value to *owner* variable
* run *terraform plan* to check the infrastructure
* run *terraform apply* to create the infrastructure

### Example

Using module from local

```
module "networking" {
  source                     = "./networking"
  aws_region                 = var.aws_region
  owner                      = var.owner
  vpc_cidr_block             = var.vpc_cidr_block
  private_subnets_cidr_block = var.private_subnets_cidr_block
  public_subnets_cidr_block  = var.public_subnets_cidr_block
}

```

Using module from repository

```
module "networking" {
  source                     = "git@github.com:braybaut/aws-basic-infrastructure.git//networking"
  aws_region                 = var.aws_region
  owner                      = var.owner
  vpc_cidr_block             = var.vpc_cidr_block
  private_subnets_cidr_block = var.private_subnets_cidr_block
  public_subnets_cidr_block  = var.public_subnets_cidr_block
}
```
