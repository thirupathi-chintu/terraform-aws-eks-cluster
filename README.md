## Introduction

The module provisions the following resources:

- EKS cluster of master nodes that can be used together with the [terraform-aws-eks-workers](https://github.com/cloudposse/terraform-aws-eks-workers),
  [terraform-aws-eks-node-group](https://github.com/cloudposse/terraform-aws-eks-node-group) and
  [terraform-aws-eks-fargate-profile](https://github.com/cloudposse/terraform-aws-eks-fargate-profile)
  modules to create a full-blown cluster
- IAM Role to allow the cluster to access other AWS services
- Security Group which is used by EKS workers to connect to the cluster and kubelets and pods to receive communication from the cluster control plane
- The module creates and automatically applies an authentication ConfigMap to allow the wrokers nodes to join the cluster and to add additional users/roles/accounts

__NOTE:__ The module works with [Terraform Cloud](https://www.terraform.io/docs/cloud/index.html).

__NOTE:__ In `auth.tf`, we added `ignore_changes = [data["mapRoles"]]` to the `kubernetes_config_map` for the following reason:
- We provision the EKS cluster and then the Kubernetes Auth ConfigMap to map additional roles/users/accounts to Kubernetes groups
- Then we wait for the cluster to become available and for the ConfigMap to get provisioned (see `data "null_data_source" "wait_for_cluster_and_kubernetes_configmap"` in `examples/complete/main.tf`)
- Then we provision a managed Node Group
- Then EKS updates the Auth ConfigMap and adds worker roles to it (for the worker nodes to join the cluster)
- Since the ConfigMap is modified outside of Terraform state, Terraform wants to update it (remove the roles that EKS added) on each `plan/apply`

If you want to modify the Node Group (e.g. add more Node Groups to the cluster) or need to map other IAM roles to Kubernetes groups,
set the variable `kubernetes_config_map_ignore_role_changes` to `false` and re-provision the module. Then set `kubernetes_config_map_ignore_role_changes` back to `true`.

## Usage


**IMPORTANT:** The `master` branch is used in `source` just as an example. In your code, do not pin to `master` because there may be breaking changes between releases.
Instead pin to the release tag (e.g. `?ref=tags/x.y.z`) of one of our [latest releases](https://github.com/cloudposse/terraform-aws-eks-cluster/releases).



For a complete example, see [examples/complete](examples/complete).

For automated tests of the complete example using [bats](https://github.com/bats-core/bats-core) and [Terratest](https://github.com/gruntwork-io/terratest) (which tests and deploys the example on AWS), see [test](test).

Other examples:

- [terraform-root-modules/eks](https://github.com/cloudposse/terraform-root-modules/tree/master/aws/eks) - Cloud Posse's service catalog of "root module" invocations for provisioning reference architectures
- [terraform-root-modules/eks-backing-services-peering](https://github.com/cloudposse/terraform-root-modules/tree/master/aws/eks-backing-services-peering) - example of VPC peering between the EKS VPC and backing services VPC

```hcl
  provider "aws" {
    region = var.region
  }

  module "label" {
    source     = "git::https://github.com/cloudposse/terraform-null-label.git?ref=master"
    namespace  = var.namespace
    name       = var.name
    stage      = var.stage
    delimiter  = var.delimiter
    attributes = compact(concat(var.attributes, list("cluster")))
    tags       = var.tags
  }

  locals {
    # The usage of the specific kubernetes.io/cluster/* resource tags below are required
    # for EKS and Kubernetes to discover and manage networking resources
    # https://www.terraform.io/docs/providers/aws/guides/eks-getting-started.html#base-vpc-networking
    tags = merge(var.tags, map("kubernetes.io/cluster/${module.label.id}", "shared"))

    # Unfortunately, most_recent (https://github.com/cloudposse/terraform-aws-eks-workers/blob/34a43c25624a6efb3ba5d2770a601d7cb3c0d391/main.tf#L141)
    # variable does not work as expected, if you are not going to use custom AMI you should
    # enforce usage of eks_worker_ami_name_filter variable to set the right kubernetes version for EKS workers,
    # otherwise the first version of Kubernetes supported by AWS (v1.11) for EKS workers will be used, but
    # EKS control plane will use the version specified by kubernetes_version variable.
    eks_worker_ami_name_filter = "amazon-eks-node-${var.kubernetes_version}*"
  }

  module "vpc" {
    source     = "git::https://github.com/cloudposse/terraform-aws-vpc.git?ref=master"
    namespace  = var.namespace
    stage      = var.stage
    name       = var.name
    attributes = var.attributes
    cidr_block = "172.16.0.0/16"
    tags       = local.tags
  }

  module "subnets" {
    source               = "git::https://github.com/cloudposse/terraform-aws-dynamic-subnets.git?ref=master"
    availability_zones   = var.availability_zones
    namespace            = var.namespace
    stage                = var.stage
    name                 = var.name
    attributes           = var.attributes
    vpc_id               = module.vpc.vpc_id
    igw_id               = module.vpc.igw_id
    cidr_block           = module.vpc.vpc_cidr_block
    nat_gateway_enabled  = false
    nat_instance_enabled = false
    tags                 = local.tags
  }

  module "eks_workers" {
    source                             = "git::https://github.com/cloudposse/terraform-aws-eks-workers.git?ref=master"
    namespace                          = var.namespace
    stage                              = var.stage
    name                               = var.name
    attributes                         = var.attributes
    tags                               = var.tags
    instance_type                      = var.instance_type
    eks_worker_ami_name_filter          = local.eks_worker_ami_name_filter
    vpc_id                             = module.vpc.vpc_id
    subnet_ids                         = module.subnets.public_subnet_ids
    health_check_type                  = var.health_check_type
    min_size                           = var.min_size
    max_size                           = var.max_size
    wait_for_capacity_timeout          = var.wait_for_capacity_timeout
    cluster_name                       = module.label.id
    cluster_endpoint                   = module.eks_cluster.eks_cluster_endpoint
    cluster_certificate_authority_data = module.eks_cluster.eks_cluster_certificate_authority_data
    cluster_security_group_id          = module.eks_cluster.security_group_id

    # Auto-scaling policies and CloudWatch metric alarms
    autoscaling_policies_enabled           = var.autoscaling_policies_enabled
    cpu_utilization_high_threshold_percent = var.cpu_utilization_high_threshold_percent
    cpu_utilization_low_threshold_percent  = var.cpu_utilization_low_threshold_percent
  }

  module "eks_cluster" {
    source     = "git::https://github.com/cloudposse/terraform-aws-eks-cluster.git?ref=master"
    namespace  = var.namespace
    stage      = var.stage
    name       = var.name
    attributes = var.attributes
    tags       = var.tags
    vpc_id     = module.vpc.vpc_id
    subnet_ids = module.subnets.public_subnet_ids

    kubernetes_version    = var.kubernetes_version
    oidc_provider_enabled = false

    workers_security_group_ids   = [module.eks_workers.security_group_id]
    workers_role_arns            = [module.eks_workers.workers_role_arn]
  }
```

Module usage with two worker groups:

```hcl
  module "eks_workers" {
    source                             = "git::https://github.com/cloudposse/terraform-aws-eks-workers.git?ref=master"
    namespace                          = var.namespace
    stage                              = var.stage
    name                               = "small"
    attributes                         = var.attributes
    tags                               = var.tags
    instance_type                      = "t3.small"
    vpc_id                             = module.vpc.vpc_id
    subnet_ids                         = module.subnets.public_subnet_ids
    health_check_type                  = var.health_check_type
    min_size                           = var.min_size
    max_size                           = var.max_size
    wait_for_capacity_timeout          = var.wait_for_capacity_timeout
    cluster_name                       = module.label.id
    cluster_endpoint                   = module.eks_cluster.eks_cluster_endpoint
    cluster_certificate_authority_data = module.eks_cluster.eks_cluster_certificate_authority_data
    cluster_security_group_id          = module.eks_cluster.security_group_id

    # Auto-scaling policies and CloudWatch metric alarms
    autoscaling_policies_enabled           = var.autoscaling_policies_enabled
    cpu_utilization_high_threshold_percent = var.cpu_utilization_high_threshold_percent
    cpu_utilization_low_threshold_percent  = var.cpu_utilization_low_threshold_percent
  }

  module "eks_workers_2" {
    source                             = "git::https://github.com/cloudposse/terraform-aws-eks-workers.git?ref=master"
    namespace                          = var.namespace
    stage                              = var.stage
    name                               = "medium"
    attributes                         = var.attributes
    tags                               = var.tags
    instance_type                      = "t3.medium"
    vpc_id                             = module.vpc.vpc_id
    subnet_ids                         = module.subnets.public_subnet_ids
    health_check_type                  = var.health_check_type
    min_size                           = var.min_size
    max_size                           = var.max_size
    wait_for_capacity_timeout          = var.wait_for_capacity_timeout
    cluster_name                       = module.label.id
    cluster_endpoint                   = module.eks_cluster.eks_cluster_endpoint
    cluster_certificate_authority_data = module.eks_cluster.eks_cluster_certificate_authority_data
    cluster_security_group_id          = module.eks_cluster.security_group_id

    # Auto-scaling policies and CloudWatch metric alarms
    autoscaling_policies_enabled           = var.autoscaling_policies_enabled
    cpu_utilization_high_threshold_percent = var.cpu_utilization_high_threshold_percent
    cpu_utilization_low_threshold_percent  = var.cpu_utilization_low_threshold_percent
  }

  module "eks_cluster" {
    source     = "git::https://github.com/cloudposse/terraform-aws-eks-cluster.git?ref=master"
    namespace  = var.namespace
    stage      = var.stage
    name       = var.name
    attributes = var.attributes
    tags       = var.tags
    vpc_id     = module.vpc.vpc_id
    subnet_ids = module.subnets.public_subnet_ids

    kubernetes_version    = var.kubernetes_version
    oidc_provider_enabled = false

    workers_role_arns          = [module.eks_workers.workers_role_arn, module.eks_workers_2.workers_role_arn]
    workers_security_group_ids = [module.eks_workers.security_group_id, module.eks_workers_2.security_group_id]
  }
```






## Makefile Targets
```
Available targets:

  help                                Help screen
  help/all                            Display help for all targets
  help/short                          This help short screen
  lint                                Lint terraform code

```
## Inputs

| Name | Description | Type | Default | Required |
|------|-------------|:----:|:-----:|:-----:|
| allowed_cidr_blocks | List of CIDR blocks to be allowed to connect to the EKS cluster | list(string) | `<list>` | no |
| allowed_security_groups | List of Security Group IDs to be allowed to connect to the EKS cluster | list(string) | `<list>` | no |
| apply_config_map_aws_auth | Whether to apply the ConfigMap to allow worker nodes to join the EKS cluster and allow additional users, accounts and roles to acces the cluster | bool | `true` | no |
| attributes | Additional attributes (e.g. `1`) | list(string) | `<list>` | no |
| cluster_log_retention_period | Number of days to retain cluster logs. Requires `enabled_cluster_log_types` to be set. See https://docs.aws.amazon.com/en_us/eks/latest/userguide/control-plane-logs.html. | number | `0` | no |
| delimiter | Delimiter to be used between `namespace`, `environment`, `stage`, `name` and `attributes` | string | `-` | no |
| enabled | Set to false to prevent the module from creating any resources | bool | `true` | no |
| enabled_cluster_log_types | A list of the desired control plane logging to enable. For more information, see https://docs.aws.amazon.com/en_us/eks/latest/userguide/control-plane-logs.html. Possible values [`api`, `audit`, `authenticator`, `controllerManager`, `scheduler`] | list(string) | `<list>` | no |
| endpoint_private_access | Indicates whether or not the Amazon EKS private API server endpoint is enabled. Default to AWS EKS resource and it is false | bool | `false` | no |
| endpoint_public_access | Indicates whether or not the Amazon EKS public API server endpoint is enabled. Default to AWS EKS resource and it is true | bool | `true` | no |
| environment | Environment, e.g. 'prod', 'staging', 'dev', 'pre-prod', 'UAT' | string | `` | no |
| kubernetes_config_map_ignore_role_changes | Set to `true` to ignore IAM role changes in the Kubernetes Auth ConfigMap | bool | `true` | no |
| kubernetes_version | Desired Kubernetes master version. If you do not specify a value, the latest available version is used | string | `1.15` | no |
| local_exec_interpreter | shell to use for local_exec | list(string) | `<list>` | no |
| map_additional_aws_accounts | Additional AWS account numbers to add to `config-map-aws-auth` ConfigMap | list(string) | `<list>` | no |
| map_additional_iam_roles | Additional IAM roles to add to `config-map-aws-auth` ConfigMap | object | `<list>` | no |
| map_additional_iam_users | Additional IAM users to add to `config-map-aws-auth` ConfigMap | object | `<list>` | no |
| name | Solution name, e.g. 'app' or 'jenkins' | string | `` | no |
| namespace | Namespace, which could be your organization name or abbreviation, e.g. 'eg' or 'cp' | string | `` | no |
| oidc_provider_enabled | Create an IAM OIDC identity provider for the cluster, then you can create IAM roles to associate with a service account in the cluster, instead of using kiam or kube2iam. For more information, see https://docs.aws.amazon.com/eks/latest/userguide/enable-iam-roles-for-service-accounts.html | bool | `false` | no |
| public_access_cidrs | Indicates which CIDR blocks can access the Amazon EKS public API server endpoint when enabled. EKS defaults this to a list with 0.0.0.0/0. | list(string) | `<list>` | no |
| region | AWS Region | string | - | yes |
| stage | Stage, e.g. 'prod', 'staging', 'dev', OR 'source', 'build', 'test', 'deploy', 'release' | string | `` | no |
| subnet_ids | A list of subnet IDs to launch the cluster in | list(string) | - | yes |
| tags | Additional tags (e.g. `map('BusinessUnit','XYZ')` | map(string) | `<map>` | no |
| vpc_id | VPC ID for the EKS cluster | string | - | yes |
| wait_for_cluster_command | `local-exec` command to execute to determine if the EKS cluster is healthy. Cluster endpoint are available as environment variable `ENDPOINT` | string | `curl --silent --fail --retry 60 --retry-delay 5 --retry-connrefused --insecure --output /dev/null $ENDPOINT/healthz` | no |
| workers_role_arns | List of Role ARNs of the worker nodes | list(string) | `<list>` | no |
| workers_security_group_ids | Security Group IDs of the worker nodes | list(string) | `<list>` | no |

## Outputs

| Name | Description |
|------|-------------|
| eks_cluster_arn | The Amazon Resource Name (ARN) of the cluster |
| eks_cluster_certificate_authority_data | The Kubernetes cluster certificate authority data |
| eks_cluster_endpoint | The endpoint for the Kubernetes API server |
| eks_cluster_id | The name of the cluster |
| eks_cluster_identity_oidc_issuer | The OIDC Identity issuer for the cluster |
| eks_cluster_identity_oidc_issuer_arn | The OIDC Identity issuer ARN for the cluster that can be used to associate IAM roles with a service account |
| eks_cluster_managed_security_group_id | Security Group ID that was created by EKS for the cluster. EKS creates a Security Group and applies it to ENI that is attached to EKS Control Plane master nodes and to any managed workloads |
| eks_cluster_role_arn | ARN of the EKS cluster IAM role |
| eks_cluster_version | The Kubernetes server version of the cluster |
| kubernetes_config_map_id | ID of `aws-auth` Kubernetes ConfigMap |
| security_group_arn | ARN of the EKS cluster Security Group |
| security_group_id | ID of the EKS cluster Security Group |
| security_group_name | Name of the EKS cluster Security Group |




## Share the Love 

Like this project? Please give it a ★ on [our GitHub](https://github.com/cloudposse/terraform-aws-eks-cluster)! (it helps us **a lot**) 

Are you using this project or any of our other projects? Consider [leaving a testimonial][testimonial]. =)


## Related Projects

Check out these related projects.

- [terraform-aws-eks-workers](https://github.com/cloudposse/terraform-aws-eks-workers) - Terraform module to provision an AWS AutoScaling Group, IAM Role, and Security Group for EKS Workers
- [terraform-aws-ec2-autoscale-group](https://github.com/cloudposse/terraform-aws-ec2-autoscale-group) - Terraform module to provision Auto Scaling Group and Launch Template on AWS
- [terraform-aws-ecs-container-definition](https://github.com/cloudposse/terraform-aws-ecs-container-definition) - Terraform module to generate well-formed JSON documents (container definitions) that are passed to the  aws_ecs_task_definition Terraform resource
- [terraform-aws-ecs-alb-service-task](https://github.com/cloudposse/terraform-aws-ecs-alb-service-task) - Terraform module which implements an ECS service which exposes a web service via ALB
- [terraform-aws-ecs-web-app](https://github.com/cloudposse/terraform-aws-ecs-web-app) - Terraform module that implements a web app on ECS and supports autoscaling, CI/CD, monitoring, ALB integration, and much more
- [terraform-aws-ecs-codepipeline](https://github.com/cloudposse/terraform-aws-ecs-codepipeline) - Terraform module for CI/CD with AWS Code Pipeline and Code Build for ECS
- [terraform-aws-ecs-cloudwatch-autoscaling](https://github.com/cloudposse/terraform-aws-ecs-cloudwatch-autoscaling) - Terraform module to autoscale ECS Service based on CloudWatch metrics
- [terraform-aws-ecs-cloudwatch-sns-alarms](https://github.com/cloudposse/terraform-aws-ecs-cloudwatch-sns-alarms) - Terraform module to create CloudWatch Alarms on ECS Service level metrics
- [terraform-aws-ec2-instance](https://github.com/cloudposse/terraform-aws-ec2-instance) - Terraform module for providing a general purpose EC2 instance
- [terraform-aws-ec2-instance-group](https://github.com/cloudposse/terraform-aws-ec2-instance-group) - Terraform module for provisioning multiple general purpose EC2 hosts for stateful applications



## Help

**Got a question?** We got answers. 

File a GitHub [issue](https://github.com/cloudposse/terraform-aws-eks-cluster/issues), send us an [email][email] or join our [Slack Community][slack].

[![README Commercial Support][readme_commercial_support_img]][readme_commercial_support_link]

## DevOps Accelerator for Startups


We are a [**DevOps Accelerator**][commercial_support]. We'll help you build your cloud infrastructure from the ground up so you can own it. Then we'll show you how to operate it and stick around for as long as you need us. 

[![Learn More](https://img.shields.io/badge/learn%20more-success.svg?style=for-the-badge)][commercial_support]

Work directly with our team of DevOps experts via email, slack, and video conferencing.

We deliver 10x the value for a fraction of the cost of a full-time engineer. Our track record is not even funny. If you want things done right and you need it done FAST, then we're your best bet.

- **Reference Architecture.** You'll get everything you need from the ground up built using 100% infrastructure as code.
- **Release Engineering.** You'll have end-to-end CI/CD with unlimited staging environments.
- **Site Reliability Engineering.** You'll have total visibility into your apps and microservices.
- **Security Baseline.** You'll have built-in governance with accountability and audit logs for all changes.
- **GitOps.** You'll be able to operate your infrastructure via Pull Requests.
- **Training.** You'll receive hands-on training so your team can operate what we build.
- **Questions.** You'll have a direct line of communication between our teams via a Shared Slack channel.
- **Troubleshooting.** You'll get help to triage when things aren't working.
- **Code Reviews.** You'll receive constructive feedback on Pull Requests.
- **Bug Fixes.** We'll rapidly work with you to fix any bugs in our projects.

## Slack Community

Join our [Open Source Community][slack] on Slack. It's **FREE** for everyone! Our "SweetOps" community is where you get to talk with others who share a similar vision for how to rollout and manage infrastructure. This is the best place to talk shop, ask questions, solicit feedback, and work together as a community to build totally *sweet* infrastructure.

