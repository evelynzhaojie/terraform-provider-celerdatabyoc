---
# generated by https://github.com/hashicorp/terraform-plugin-docs
page_title: "celerdatabyoc_classic_cluster Resource - terraform-provider-celerdatabyoc"
subcategory: ""
description: |-
  
---

~> The resource's API may change in subsequent versions to simplify the user experience.

This document can help you deploy a classic cluster in AWS EC2. Please follow the runnable example on AWS.
When you create your first cluster consisting of a single FE node, a single BE node with an instance type of m6i.xlarge for this FE node and m5.xlarge for the BE node, this cluster will be automatically matched as a free tier cluster for you to experience the product features of the CelerData Cloud BYOC.

### Supported Instance type

<html>
 <head></head>
 <body>
  <table>
   <tbody>
    <tr>
     <td rowspan="2"></td>
     <td rowspan="2">Instance type</td>
     <td rowspan="2">Instance size</td>
    </tr>
    <tr>
    </tr>
    <tr>
     <td rowspan="7">FE</td>
    </tr>
    <tr>
    </tr>
    <tr>
     <td>m6i.xlarge</td>
     <td>4 Cores 16GB Memory 50 GB Storage</td>
    </tr>
    <tr>
     <td>m6i.2xlarge</td>
     <td>8 Cores 32GB Memory 200 GB Storage</td>
    </tr>
    <tr>
     <td>m6i.4xlarge</td>
     <td>16 Cores 64GB Memory 50 GB Storage</td>
    </tr>
    <tr>
     <td>m6i.8xlarge</td>
     <td>32 Cores 128GB Memory 50 GB Storage</td>
    </tr>
    <tr>
    </tr>
    <tr>
     <td rowspan="6">BE (General Purpose)</td>
    </tr>
    <tr>
     <td>m5.xlarge</td>
     <td>4 Cores 16GB Memory</td>
    </tr>
    <tr>
     <td>m5.2xlarge</td>
     <td>8 Cores 32GB Memory</td>
    </tr>
    <tr>
     <td>m5.4xlarge</td>
     <td>16 Cores 64GB Memory</td>
    </tr>
    <tr>
     <td>m5.8xlarge</td>
     <td>32 Cores 128GB Memory</td>
    </tr>
    <tr>
    </tr>
    <tr>
     <td rowspan="7">BE (Memory Optimized)</td>
    <tr>
     <td>r6i.xlarge</td>
     <td>4 Cores 32GB Memory</td>
    </tr>
    <tr>
     <td>r6i.2xlarge</td>
     <td>8 Cores 64GB Memory</td>
    </tr>
    <tr>
     <td>r6i.4xlarge</td>
     <td>16 Cores 128GB Memory</td>
    </tr>
    <tr>
     <td>r6i.8xlarge</td>
     <td>32 Cores 256GB Memory</td>
    </tr>
   </tbody>
  </table>
 </body>
</html>

### Example Usage

```terraform
locals {
  your_s3_bucket = "[your S3 bucket]" 
}

resource "celerdatabyoc_aws_data_credential_policy" "new" {
  bucket = local.your_s3_bucket
}

data "celerdatabyoc_aws_data_credential_assume_policy" "assume_role" {}

resource "aws_iam_role" "celerdata_data_cred_role" {
  name               = "celerdata_data_cred_role"
  assume_role_policy = data.celerdatabyoc_aws_data_credential_assume_policy.assume_role.json
  description        = "Celerdata Data Credential"
  inline_policy {
    name   = "celerdata_data_cred_role_policy"
    policy = celerdatabyoc_aws_data_credential_policy.new.json
  }
}

resource "aws_iam_instance_profile" "celerdata_data_cred_profile" {
  name = "celerdata_data_cred_profile"
  role = aws_iam_role.celerdata_data_cred_role.name
}

resource "celerdatabyoc_aws_deployment_credential_policy" "new" {
  bucket = local.your_s3_bucket
  data_role_arn = aws_iam_role.celerdata_data_cred_role.arn
}

resource "celerdatabyoc_aws_deployment_credential_assume_policy" "new" {}

resource "aws_iam_role" "deploy_cred_role" {
  name               = "deploy_cred_role"
  assume_role_policy = celerdatabyoc_aws_deployment_credential_assume_policy.new.json
  description        = "Celerdata Deploy Credential"
  inline_policy {
    name   = "deploy_cred_role-policy"
    policy = celerdatabyoc_aws_deployment_credential_policy.new.json
  }
}

resource "celerdatabyoc_aws_data_credential" "new" {
  name = "data-credential"
  role_arn = aws_iam_role.celerdata_data_cred_role.arn
  instance_profile_arn = aws_iam_instance_profile.celerdata_data_cred_profile.arn
  bucket_name = local.your_s3_bucket
  policy_version = celerdatabyoc_aws_data_credential_policy.new.version
}

resource "celerdatabyoc_aws_deployment_role_credential" "new" {
  name = "deployment-role-credential"
  role_arn = aws_iam_role.deploy_cred_role.arn
  external_id = celerdatabyoc_aws_deployment_credential_assume_policy.new.external_id
  policy_version = celerdatabyoc_aws_deployment_credential_policy.new.version
}

resource "celerdatabyoc_aws_network" "new" {
  name = "[name your net work]"
  subnet_id = "[your subnet id]"
  security_group_id = "[your security group id]"
  region = "[your AWS VPC region]"
  deployment_credential_id = celerdatabyoc_aws_deployment_role_credential.new.id
  vpc_endpoint_id = "[your vpc endpoint id]"
}

resource "celerdatabyoc_classic_cluster" "new" {
  cluster_name = "[your cluster name]"
  fe_instance_type = "[fe type]"
  fe_node_count = 1
  deployment_credential_id = celerdatabyoc_aws_deployment_role_credential.new.id
  data_credential_id = celerdatabyoc_aws_data_credential.new.id
  network_id = celerdatabyoc_aws_network.new.id
  be_instance_type = "[be type]"
  be_node_count = 1
  be_storage_size_gb = 100
  default_admin_password = "[set initial SQL user password]"
  expected_cluster_state = "Suspended"
  resource_tags = {
    celerdata = "[tag name]"
  }
  csp = "aws"
  region = "[your AWS VPC region]"

  init_scripts {
      logs_dir    = "log-s3-path/"
      script_path = "script-s3-path/test1.sh" 
  }
  run_scripts_parallel = false
}

```

### Argument Reference

  * `expected_cluster_state` - (Required) When creating a cluster, you need to declare whether the cluster status is `Suspended` or `Running`
  * `cluster_name` - (ForceNew) Should name your cluster.
  * `fe_instance_type` - (Required) Should select a fe instance type from the table above
  * `fe_node_count` - (Optional) Default number of fe nodes is `1`, optional numbers are: `1,3,5`
  * `deployment_credential_id` - (ForceNew) Type as "celerdatabyoc_aws_deployment_role_credential.new.id"
  * `data_credential_id` - (ForceNew) Type as `celerdatabyoc_aws_data_credential.new.id`
  * `network_id` - (ForceNew) Type as `celerdatabyoc_aws_network.new.id`
  * `be_instance_type` - (Required) Should select a be instance type from the table above.
  * `be_node_count` - (Optional)  Default number is `3`.
  * `be_storage_size_gb` - (Optional) The value set must be a multiple of `100`, and subsequent changes can only increase. The expansion interval must be greater than `6` hours.
  * `default_admin_password` - (Required) Set initial SQL user password
* `resource_tags` - (Optional)
* `csp` - (Required) Now, we only support `aws`
* `region` - (Required) Your AWS VPC region. The optional regions are as follows：
  - Asia Pacific (Singapore) ap-southeast-1
  - US East (N. Virginia) us-east-1
  - US West (Oregon) us-west-2
  - Europe (Ireland) eu-west-1
- init_scripts - （Optional）Configuration block to customize the script upload location. The maximum number of executable scripts is `20`. You can learn more about executable scripts with [Run scripts](https://docs-sandbox.celerdata.com/en-us/main/run_scripts).
  - logs_dir - (ForceNew) Storage path for script execution results.
  - script_path - (ForceNew) The S3 bucket address where the script is stored.
- run_scripts_parallel - (Optional) Execute/not execute script in parallel, the default value is false.

### Supplementary material

[The AWS IAM](https://us-east-1.console.aws.amazon.com/iamv2/home?region=us-east-1#/policies)<br />
[How to create AWS data credential](https://docs-sandbox.celerdata.com/en-us/main/cloud_settings/manage_storage_configurations)<br />
[How to create the AWS Deployment credential](https://docs-sandbox.celerdata.com/en-us/main/cloud_settings/manage_credentials)<br />
[Create a network configuration](https://docs-sandbox.celerdata.com/en-us/main/cloud_settings/manage_network_configurations)