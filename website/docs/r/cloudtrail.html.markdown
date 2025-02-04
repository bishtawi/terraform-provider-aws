---
subcategory: "CloudTrail"
layout: "aws"
page_title: "AWS: aws_cloudtrail"
description: |-
  Provides a CloudTrail resource.
---

# Resource: aws_cloudtrail

Provides a CloudTrail resource.

~> *NOTE:* For a multi-region trail, this resource must be in the home region of the trail.

~> *NOTE:* For an organization trail, this resource must be in the master account of the organization.

## Example Usage

### Basic

Enable CloudTrail to capture all compatible management events in region.
For capturing events from services like IAM, `include_global_service_events` must be enabled.

```terraform
data "aws_caller_identity" "current" {}

resource "aws_cloudtrail" "foobar" {
  name                          = "tf-trail-foobar"
  s3_bucket_name                = aws_s3_bucket.foo.id
  s3_key_prefix                 = "prefix"
  include_global_service_events = false
}

resource "aws_s3_bucket" "foo" {
  bucket        = "tf-test-trail"
  force_destroy = true

  policy = <<POLICY
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "AWSCloudTrailAclCheck",
            "Effect": "Allow",
            "Principal": {
              "Service": "cloudtrail.amazonaws.com"
            },
            "Action": "s3:GetBucketAcl",
            "Resource": "arn:aws:s3:::tf-test-trail"
        },
        {
            "Sid": "AWSCloudTrailWrite",
            "Effect": "Allow",
            "Principal": {
              "Service": "cloudtrail.amazonaws.com"
            },
            "Action": "s3:PutObject",
            "Resource": "arn:aws:s3:::tf-test-trail/prefix/AWSLogs/${data.aws_caller_identity.current.account_id}/*",
            "Condition": {
                "StringEquals": {
                    "s3:x-amz-acl": "bucket-owner-full-control"
                }
            }
        }
    ]
}
POLICY
}
```

### Data Event Logging

CloudTrail can log [Data Events](https://docs.aws.amazon.com/awscloudtrail/latest/userguide/logging-data-events-with-cloudtrail.html) for certain services such as S3 bucket objects and Lambda function invocations. Additional information about data event configuration can be found in the [CloudTrail API DataResource documentation](https://docs.aws.amazon.com/awscloudtrail/latest/APIReference/API_DataResource.html).

#### Logging All Lambda Function Invocations

```terraform
resource "aws_cloudtrail" "example" {
  # ... other configuration ...

  event_selector {
    read_write_type           = "All"
    include_management_events = true

    data_resource {
      type   = "AWS::Lambda::Function"
      values = ["arn:aws:lambda"]
    }
  }
}
```

#### Logging All S3 Bucket Object Events

```terraform
resource "aws_cloudtrail" "example" {
  # ... other configuration ...

  event_selector {
    read_write_type           = "All"
    include_management_events = true

    data_resource {
      type   = "AWS::S3::Object"
      values = ["arn:aws:s3:::"]
    }
  }
}
```

#### Logging Individual S3 Bucket Events

```terraform
data "aws_s3_bucket" "important-bucket" {
  bucket = "important-bucket"
}

resource "aws_cloudtrail" "example" {
  # ... other configuration ...

  event_selector {
    read_write_type           = "All"
    include_management_events = true

    data_resource {
      type = "AWS::S3::Object"

      # Make sure to append a trailing '/' to your ARN if you want
      # to monitor all objects in a bucket.
      values = ["${data.aws_s3_bucket.important-bucket.arn}/"]
    }
  }
}
```

#### Sending Events to CloudWatch Logs

```terraform
resource "aws_cloudwatch_log_group" "example" {
  name = "Example"
}

resource "aws_cloudtrail" "example" {
  # ... other configuration ...

  cloud_watch_logs_group_arn = "${aws_cloudwatch_log_group.example.arn}:*" # CloudTrail requires the Log Stream wildcard
}
```

## Argument Reference

The following arguments are supported:

* `name` - (Required) Specifies the name of the trail.
* `s3_bucket_name` - (Required) Specifies the name of the S3 bucket designated for publishing log files.
* `s3_key_prefix` - (Optional) Specifies the S3 key prefix that follows
    the name of the bucket you have designated for log file delivery.
* `cloud_watch_logs_role_arn` - (Optional) Specifies the role for the CloudWatch Logs
    endpoint to assume to write to a user’s log group.
* `cloud_watch_logs_group_arn` - (Optional) Specifies a log group name using an Amazon Resource Name (ARN),
    that represents the log group to which CloudTrail logs will be delivered. Note that CloudTrail requires the Log Stream wildcard.
* `enable_logging` - (Optional) Enables logging for the trail. Defaults to `true`.
    Setting this to `false` will pause logging.
* `include_global_service_events` - (Optional) Specifies whether the trail is publishing events
    from global services such as IAM to the log files. Defaults to `true`.
* `is_multi_region_trail` - (Optional) Specifies whether the trail is created in the current
    region or in all regions. Defaults to `false`.
* `is_organization_trail` - (Optional) Specifies whether the trail is an AWS Organizations trail. Organization trails log events for the master account and all member accounts. Can only be created in the organization master account. Defaults to `false`.
* `sns_topic_name` - (Optional) Specifies the name of the Amazon SNS topic
    defined for notification of log file delivery.
* `enable_log_file_validation` - (Optional) Specifies whether log file integrity validation is enabled.
    Defaults to `false`.
* `kms_key_id` - (Optional) Specifies the KMS key ARN to use to encrypt the logs delivered by CloudTrail.
* `event_selector` - (Optional) Specifies an event selector for enabling data event logging. Fields documented below. Please note the [CloudTrail limits](https://docs.aws.amazon.com/awscloudtrail/latest/userguide/WhatIsCloudTrail-Limits.html) when configuring these.
* `insight_selector` - (Optional) Specifies an insight selector for identifying unusual operational activity. Fields documented below.
* `tags` - (Optional) A map of tags to assign to the trail. If configured with a provider [`default_tags` configuration block](/docs/providers/aws/index.html#default_tags-configuration-block) present, tags with matching keys will overwrite those defined at the provider-level.

### Event Selector Arguments
For **event_selector** the following attributes are supported.

* `read_write_type` (Optional) - Specify if you want your trail to log read-only events, write-only events, or all. By default, the value is All. You can specify only the following value: "ReadOnly", "WriteOnly", "All". Defaults to `All`.
* `include_management_events` (Optional) - Specify if you want your event selector to include management events for your trail.
* `data_resource` (Optional) - Specifies logging data events. Fields documented below.

#### Data Resource Arguments
For **data_resource** the following attributes are supported.

* `type` (Required) - The resource type in which you want to log data events. You can specify only the following value: "AWS::S3::Object", "AWS::Lambda::Function" and "AWS::DynamoDB::Table".
* `values` (Required) - A list of ARN strings or partial ARN strings to specify selectors for data audit events over data resources. ARN list is specific to single-valued `type`.
    * `arn:aws:s3:::<bucket_name>/` - all objects in bucket
    * `arn:aws:s3:::<bucket_name>/key` - specific key(s)
    * `arn:aws:lambda` - all lambda events within account
    * `arn:aws:lambda:us-west-2:111111111111:function:helloworld` - events for exact arn
    * `arn:aws:dynamodb` - all DDB events for all tables within account


### Insight Selector Arguments

For **insight_selector** the following attributes are supported.

* `insight_type` (Optional) - The type of insights to log on a trail. In this release, only `ApiCallRateInsight` is supported as an insight type.

## Attributes Reference

In addition to all arguments above, the following attributes are exported:

* `id` - The name of the trail.
* `home_region` - The region in which the trail was created.
* `arn` - The Amazon Resource Name of the trail.
* `tags_all` - A map of tags assigned to the resource, including those inherited from the provider [`default_tags` configuration block](/docs/providers/aws/index.html#default_tags-configuration-block).

## Import

Cloudtrails can be imported using the `name`, e.g.

```
$ terraform import aws_cloudtrail.sample my-sample-trail
```
