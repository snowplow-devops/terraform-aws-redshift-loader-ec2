[![Release][release-image]][release] [![CI][ci-image]][ci] [![License][license-image]][license] [![Registry][registry-image]][registry] [![Source][source-image]][source]

# terraform-aws-redshift-loader-ec2

A Terraform module which deploys the Snowplow Redshift Loader on an EC2 node.

## Telemetry

This module by default collects and forwards telemetry information to Snowplow to understand how our applications are being used.  No identifying information about your sub-account or account fingerprints are ever forwarded to us - it is very simple information about what modules and applications are deployed and active.

If you wish to subscribe to our mailing list for updates to these modules or security advisories please set the `user_provided_id` variable to include a valid email address which we can reach you at.

### How do I disable it?

To disable telemetry simply set variable `telemetry_enabled = false`.

### What are you collecting?

For details on what information is collected please see this module: https://github.com/snowplow-devops/terraform-snowplow-telemetry

## Usage

Redshift Loader loads shredded events from S3 bucket to Redshift. 

For more information on how it works, see [this overview](https://docs.snowplow.io/docs/storing-querying/loading-process/?warehouse=redshift&cloud=aws-micro-batching).

To configure Redshift, please refer to the [quick start guide](https://docs.snowplow.io/docs/getting-started-on-snowplow-open-source/quick-start/?warehouse=redshift#prepare-the-destination).

_NOTE_: You will need to ensure that the loader can access the cluster over whatever port is configured for the cluster (generally `5439`). If running in the same VPC as the pipeline you can allowlist the security group of the loader directly (`module output = sg_id`).

Duration settings such as `folder_monitoring_period` or `retry_period` should be given in the [documented duration format][duration-doc].

## Example

Normally, this module would be used as part of our [quick start guide](https://docs.snowplow.io/docs/getting-started-on-snowplow-open-source/quick-start/). However, you can also use it standalone for a custom setup.

See example below:

```hcl
# Note: This should be the same bucket that is used by the transformer to produce data to load
module "s3_pipeline_bucket" {
  source = "snowplow-devops/s3-bucket/aws"

  bucket_name = "your-bucket-name"
}

# Note: This should be the same queue that is passed to the transformer to produce data to load
resource "aws_sqs_queue" "rs_message_queue" {
  content_based_deduplication = true
  kms_master_key_id           = "alias/aws/sqs"
  name                        = "rs-loader.fifo"
  fifo_queue                  = true
}

module "transformer_stsv" {
  source  = "snowplow-devops/transformer-kinesis-ec2/aws"

  name       = "transformer-server-stsv"
  vpc_id     = var.vpc_id
  subnet_ids = var.subnet_ids

  stream_name             = module.enriched_stream.name
  s3_bucket_name          = module.s3_pipeline_bucket.id
  s3_bucket_object_prefix = "transformed/good/shredded/tsv"
  window_period_min       = 1
  sqs_queue_name          = aws_sqs_queue.rs_message_queue.name

  transformation_type  = "shred"
  default_shred_format = "TSV"

  ssh_key_name     = "your-key-name"
  ssh_ip_allowlist = ["0.0.0.0/0"]

  # Linking in the custom Iglu Server here
  custom_iglu_resolvers = [
    {
      name            = "Iglu Server"
      priority        = 0
      uri             = "http://your-iglu-server-endpoint/api"
      api_key         = var.iglu_super_api_key
      vendor_prefixes = []
    }
  ]
}

module "rs_loader" {
  source = "snowplow-devops/redshift-loader-ec2/aws"

  name       = "rs-loader-server"
  vpc_id     = var.vpc_id
  subnet_ids = var.subnet_ids

  sqs_queue_name = aws_sqs_queue.rs_message_queue.name

  redshift_host               = "<HOST>"
  redshift_database           = "<DATABASE>"
  redshift_port               = <PORT>
  redshift_schema             = "<SCHEMA>"
  redshift_loader_user        = "<LOADER_USER>"
  redshift_password           = "<PASSWORD>"
  redshift_aws_s3_bucket_name = module.s3_pipeline_bucket.id

  ssh_key_name     = "your-key-name"
  ssh_ip_allowlist = ["0.0.0.0/0"]

  # Linking in the custom Iglu Server here
  custom_iglu_resolvers = [
    {
      name            = "Iglu Server"
      priority        = 0
      uri             = "http://your-iglu-server-endpoint/api"
      api_key         = var.iglu_super_api_key
      vendor_prefixes = []
    }
  ]
}
```

## Requirements

| Name | Version |
|------|---------|
| <a name="requirement_terraform"></a> [terraform](#requirement\_terraform) | >= 1.0.0 |
| <a name="requirement_aws"></a> [aws](#requirement\_aws) | >= 3.72.0 |

## Providers

| Name | Version |
|------|---------|
| <a name="provider_aws"></a> [aws](#provider\_aws) | >= 3.72.0 |

## Modules

| Name | Source | Version |
|------|--------|---------|
| <a name="module_instance_type_metrics"></a> [instance\_type\_metrics](#module\_instance\_type\_metrics) | snowplow-devops/ec2-instance-type-metrics/aws | 0.1.2 |
| <a name="module_service"></a> [service](#module\_service) | snowplow-devops/service-ec2/aws | 0.2.0 |
| <a name="module_telemetry"></a> [telemetry](#module\_telemetry) | snowplow-devops/telemetry/snowplow | 0.4.0 |

## Resources

| Name | Type |
|------|------|
| [aws_cloudwatch_log_group.log_group](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/cloudwatch_log_group) | resource |
| [aws_iam_instance_profile.instance_profile](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/iam_instance_profile) | resource |
| [aws_iam_policy.iam_policy](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/iam_policy) | resource |
| [aws_iam_policy.sts_credentials_policy](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/iam_policy) | resource |
| [aws_iam_role.iam_role](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/iam_role) | resource |
| [aws_iam_role.sts_credentials_role](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/iam_role) | resource |
| [aws_iam_role_policy_attachment.policy_attachment](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/iam_role_policy_attachment) | resource |
| [aws_iam_role_policy_attachment.sts_credentials_policy_attachement](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/iam_role_policy_attachment) | resource |
| [aws_security_group.sg](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/security_group) | resource |
| [aws_security_group_rule.egress_tcp_443](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/security_group_rule) | resource |
| [aws_security_group_rule.egress_tcp_80](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/security_group_rule) | resource |
| [aws_security_group_rule.egress_udp_123](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/security_group_rule) | resource |
| [aws_security_group_rule.egress_udp_statsd](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/security_group_rule) | resource |
| [aws_security_group_rule.ingress_tcp_22](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/security_group_rule) | resource |
| [aws_caller_identity.current](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/data-sources/caller_identity) | data source |
| [aws_iam_policy_document.sts_credentials_role](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/data-sources/iam_policy_document) | data source |
| [aws_region.current](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/data-sources/region) | data source |

## Inputs

| Name | Description | Type | Default | Required |
|------|-------------|------|---------|:--------:|
| <a name="input_name"></a> [name](#input\_name) | A name which will be prepended to the resources created | `string` | n/a | yes |
| <a name="input_redshift_aws_s3_bucket_name"></a> [redshift\_aws\_s3\_bucket\_name](#input\_redshift\_aws\_s3\_bucket\_name) | AWS bucket name where data to load is stored | `string` | n/a | yes |
| <a name="input_redshift_database"></a> [redshift\_database](#input\_redshift\_database) | Redshift database name | `string` | n/a | yes |
| <a name="input_redshift_host"></a> [redshift\_host](#input\_redshift\_host) | Redshift cluster hostname | `string` | n/a | yes |
| <a name="input_redshift_loader_user"></a> [redshift\_loader\_user](#input\_redshift\_loader\_user) | Name of the user that will be used for loading data | `string` | n/a | yes |
| <a name="input_redshift_password"></a> [redshift\_password](#input\_redshift\_password) | Password for redshift\_loader\_user used by loader to perform loading | `string` | n/a | yes |
| <a name="input_redshift_schema"></a> [redshift\_schema](#input\_redshift\_schema) | Redshift schema name | `string` | n/a | yes |
| <a name="input_sqs_queue_name"></a> [sqs\_queue\_name](#input\_sqs\_queue\_name) | SQS queue name | `string` | n/a | yes |
| <a name="input_ssh_key_name"></a> [ssh\_key\_name](#input\_ssh\_key\_name) | The name of the SSH key-pair to attach to all EC2 nodes deployed | `string` | n/a | yes |
| <a name="input_subnet_ids"></a> [subnet\_ids](#input\_subnet\_ids) | The list of subnets to deploy Loader across | `list(string)` | n/a | yes |
| <a name="input_vpc_id"></a> [vpc\_id](#input\_vpc\_id) | The VPC to deploy Loader within | `string` | n/a | yes |
| <a name="input_amazon_linux_2_ami_id"></a> [amazon\_linux\_2\_ami\_id](#input\_amazon\_linux\_2\_ami\_id) | The AMI ID to use which must be based of of Amazon Linux 2; by default the latest community version is used | `string` | `""` | no |
| <a name="input_associate_public_ip_address"></a> [associate\_public\_ip\_address](#input\_associate\_public\_ip\_address) | Whether to assign a public ip address to this instance | `bool` | `true` | no |
| <a name="input_cloudwatch_logs_enabled"></a> [cloudwatch\_logs\_enabled](#input\_cloudwatch\_logs\_enabled) | Whether application logs should be reported to CloudWatch | `bool` | `true` | no |
| <a name="input_cloudwatch_logs_retention_days"></a> [cloudwatch\_logs\_retention\_days](#input\_cloudwatch\_logs\_retention\_days) | The length of time in days to retain logs for | `number` | `7` | no |
| <a name="input_custom_iglu_resolvers"></a> [custom\_iglu\_resolvers](#input\_custom\_iglu\_resolvers) | The custom Iglu Resolvers that will be used by Stream Shredder | <pre>list(object({<br>    name            = string<br>    priority        = number<br>    uri             = string<br>    api_key         = string<br>    vendor_prefixes = list(string)<br>  }))</pre> | `[]` | no |
| <a name="input_default_iglu_resolvers"></a> [default\_iglu\_resolvers](#input\_default\_iglu\_resolvers) | The default Iglu Resolvers that will be used by Stream Shredder | <pre>list(object({<br>    name            = string<br>    priority        = number<br>    uri             = string<br>    api_key         = string<br>    vendor_prefixes = list(string)<br>  }))</pre> | <pre>[<br>  {<br>    "api_key": "",<br>    "name": "Iglu Central",<br>    "priority": 10,<br>    "uri": "http://iglucentral.com",<br>    "vendor_prefixes": []<br>  },<br>  {<br>    "api_key": "",<br>    "name": "Iglu Central - Mirror 01",<br>    "priority": 20,<br>    "uri": "http://mirror01.iglucentral.com",<br>    "vendor_prefixes": []<br>  }<br>]</pre> | no |
| <a name="input_folder_monitoring_enabled"></a> [folder\_monitoring\_enabled](#input\_folder\_monitoring\_enabled) | Whether folder monitoring should be activated or not | `bool` | `false` | no |
| <a name="input_folder_monitoring_period"></a> [folder\_monitoring\_period](#input\_folder\_monitoring\_period) | How often to folder should be checked by folder monitoring | `string` | `"8 hours"` | no |
| <a name="input_folder_monitoring_since"></a> [folder\_monitoring\_since](#input\_folder\_monitoring\_since) | Specifies since when folder monitoring will check | `string` | `"14 days"` | no |
| <a name="input_folder_monitoring_until"></a> [folder\_monitoring\_until](#input\_folder\_monitoring\_until) | Specifies until when folder monitoring will check | `string` | `"6 hours"` | no |
| <a name="input_health_check_enabled"></a> [health\_check\_enabled](#input\_health\_check\_enabled) | Whether health check should be enabled or not | `bool` | `false` | no |
| <a name="input_health_check_freq"></a> [health\_check\_freq](#input\_health\_check\_freq) | Frequency of health check | `string` | `"1 hour"` | no |
| <a name="input_health_check_timeout"></a> [health\_check\_timeout](#input\_health\_check\_timeout) | How long to wait for a response for health check query | `string` | `"1 min"` | no |
| <a name="input_iam_permissions_boundary"></a> [iam\_permissions\_boundary](#input\_iam\_permissions\_boundary) | The permissions boundary ARN to set on IAM roles created | `string` | `""` | no |
| <a name="input_instance_type"></a> [instance\_type](#input\_instance\_type) | The instance type to use | `string` | `"t3a.micro"` | no |
| <a name="input_java_opts"></a> [java\_opts](#input\_java\_opts) | Custom JAVA Options | `string` | `"-Dorg.slf4j.simpleLogger.defaultLogLevel=info -XX:MinRAMPercentage=50 -XX:MaxRAMPercentage=75"` | no |
| <a name="input_redshift_aws_s3_folder_monitoring_stage_url"></a> [redshift\_aws\_s3\_folder\_monitoring\_stage\_url](#input\_redshift\_aws\_s3\_folder\_monitoring\_stage\_url) | AWS bucket URL of folder monitoring stage - must be within 'redshift\_aws\_s3\_bucket\_name' (NOTE: must be set if 'folder\_monitoring\_enabled' is true) | `string` | `""` | no |
| <a name="input_redshift_aws_s3_folder_monitoring_transformer_output_stage_url"></a> [redshift\_aws\_s3\_folder\_monitoring\_transformer\_output\_stage\_url](#input\_redshift\_aws\_s3\_folder\_monitoring\_transformer\_output\_stage\_url) | AWS bucket URL of transformer output stage - must be within 'redshift\_aws\_s3\_bucket\_name' (NOTE: must be set if 'folder\_monitoring\_enabled' is true) | `string` | `""` | no |
| <a name="input_redshift_jsonpaths_bucket"></a> [redshift\_jsonpaths\_bucket](#input\_redshift\_jsonpaths\_bucket) | S3 path that holds JSONPaths | `string` | `""` | no |
| <a name="input_redshift_max_error"></a> [redshift\_max\_error](#input\_redshift\_max\_error) | Redshift max error setting which controls amount of acceptable loading errors | `number` | `1` | no |
| <a name="input_redshift_port"></a> [redshift\_port](#input\_redshift\_port) | Redshift port | `number` | `5439` | no |
| <a name="input_retry_period"></a> [retry\_period](#input\_retry\_period) | How often batch of failed folders should be pulled into a discovery queue | `string` | `"10 min"` | no |
| <a name="input_retry_queue_enabled"></a> [retry\_queue\_enabled](#input\_retry\_queue\_enabled) | Whether retry queue should be enabled or not | `bool` | `false` | no |
| <a name="input_retry_queue_interval"></a> [retry\_queue\_interval](#input\_retry\_queue\_interval) | Artificial pause after each failed folder being added to the queue | `string` | `"10 min"` | no |
| <a name="input_retry_queue_max_attempt"></a> [retry\_queue\_max\_attempt](#input\_retry\_queue\_max\_attempt) | How many attempt to make for each folder | `number` | `-1` | no |
| <a name="input_retry_queue_size"></a> [retry\_queue\_size](#input\_retry\_queue\_size) | How many failures should be kept in memory | `number` | `-1` | no |
| <a name="input_sentry_dsn"></a> [sentry\_dsn](#input\_sentry\_dsn) | DSN for Sentry instance | `string` | `""` | no |
| <a name="input_sentry_enabled"></a> [sentry\_enabled](#input\_sentry\_enabled) | Whether Sentry should be enabled or not | `bool` | `false` | no |
| <a name="input_sp_tracking_app_id"></a> [sp\_tracking\_app\_id](#input\_sp\_tracking\_app\_id) | App id for Snowplow tracking | `string` | `""` | no |
| <a name="input_sp_tracking_collector_url"></a> [sp\_tracking\_collector\_url](#input\_sp\_tracking\_collector\_url) | Collector URL for Snowplow tracking | `string` | `""` | no |
| <a name="input_sp_tracking_enabled"></a> [sp\_tracking\_enabled](#input\_sp\_tracking\_enabled) | Whether Snowplow tracking should be activated or not | `bool` | `false` | no |
| <a name="input_ssh_ip_allowlist"></a> [ssh\_ip\_allowlist](#input\_ssh\_ip\_allowlist) | The list of CIDR ranges to allow SSH traffic from | `list(any)` | <pre>[<br>  "0.0.0.0/0"<br>]</pre> | no |
| <a name="input_statsd_enabled"></a> [statsd\_enabled](#input\_statsd\_enabled) | Whether Statsd should be enabled or not | `bool` | `false` | no |
| <a name="input_statsd_host"></a> [statsd\_host](#input\_statsd\_host) | Hostname of StatsD server | `string` | `""` | no |
| <a name="input_statsd_port"></a> [statsd\_port](#input\_statsd\_port) | Port of StatsD server | `number` | `8125` | no |
| <a name="input_stdout_metrics_enabled"></a> [stdout\_metrics\_enabled](#input\_stdout\_metrics\_enabled) | Whether logging metrics to stdout should be activated or not | `bool` | `false` | no |
| <a name="input_tags"></a> [tags](#input\_tags) | The tags to append to this resource | `map(string)` | `{}` | no |
| <a name="input_telemetry_enabled"></a> [telemetry\_enabled](#input\_telemetry\_enabled) | Whether or not to send telemetry information back to Snowplow Analytics Ltd | `bool` | `true` | no |
| <a name="input_user_provided_id"></a> [user\_provided\_id](#input\_user\_provided\_id) | An optional unique identifier to identify the telemetry events emitted by this stack | `string` | `""` | no |
| <a name="input_webhook_collector"></a> [webhook\_collector](#input\_webhook\_collector) | URL of webhook collector | `string` | `""` | no |
| <a name="input_webhook_enabled"></a> [webhook\_enabled](#input\_webhook\_enabled) | Whether webhook should be enabled or not | `bool` | `false` | no |

## Outputs

| Name | Description |
|------|-------------|
| <a name="output_asg_id"></a> [asg\_id](#output\_asg\_id) | ID of the ASG |
| <a name="output_asg_name"></a> [asg\_name](#output\_asg\_name) | Name of the ASG |
| <a name="output_sg_id"></a> [sg\_id](#output\_sg\_id) | ID of the security group attached to the Redshift Loader servers |

# Copyright and license

The Terraform AWS Redshift Loader on EC2 project is Copyright 2023-2023 Snowplow Analytics Ltd.

Licensed under the [Apache License, Version 2.0][license] (the "License");
you may not use this software except in compliance with the License.

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.

[duration-doc]: https://github.com/lightbend/config/blob/main/HOCON.md#duration-format

[release]: https://github.com/snowplow-devops/terraform-aws-redshift-loader-ec2/releases/latest
[release-image]: https://img.shields.io/github/v/release/snowplow-devops/terraform-aws-redshift-loader-ec2

[ci]: https://github.com/snowplow-devops/terraform-aws-redshift-loader-ec2/actions?query=workflow%3Aci
[ci-image]: https://github.com/snowplow-devops/terraform-aws-redshift-loader-ec2/workflows/ci/badge.svg

[license]: https://www.apache.org/licenses/LICENSE-2.0
[license-image]: https://img.shields.io/badge/license-Apache--2-blue.svg?style=flat

[registry]: https://registry.terraform.io/modules/snowplow-devops/redshift-loader-ec2/aws/latest
[registry-image]: https://img.shields.io/static/v1?label=Terraform&message=Registry&color=7B42BC&logo=terraform

[source]: https://github.com/snowplow/snowplow-rdb-loader
[source-image]: https://img.shields.io/static/v1?label=Snowplow&message=Redshift%20Loader&color=0E9BA4&logo=GitHub
