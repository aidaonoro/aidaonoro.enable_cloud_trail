## Enable CloudTrail Ansible Role

This role creates and enables a trail for CloudTrail in all regions with encrypted logging
to S3 activated. If you do not want to enable CloudWatch logs just leave
**cloud_trail_log_group** and **cloudtrail_logs_role** variables not defined.

### Prerequisites

In order to active encryption is needed an AWS KMS with permissions for
CloudTrail to encrypt and decrypt.
You should have a similar policy as the shown below and provide your KMS ARN:

```json
{
  "Version": "2012-10-17",
  "Id": "Key policy created by CloudTrail",
  "Statement": [
    {
      "Sid": "Enable IAM User Permissions",
      "Effect": "Allow",
      "Principal": {
        "AWS": [
          "arn:aws:iam::{{ account_id }}:user/{{ user }}",
          "arn:aws:iam::{{ account_id }}:root"
        ]
      },
      "Action": "kms:*",
      "Resource": "*"
    },
    {
      "Sid": "Allow CloudTrail to encrypt logs",
      "Effect": "Allow",
      "Principal": {
        "Service": "cloudtrail.amazonaws.com"
      },
      "Action": "kms:GenerateDataKey*",
      "Resource": "*",
      "Condition": {
        "StringLike": {
          "kms:EncryptionContext:aws:cloudtrail:arn": "arn:aws:cloudtrail:*:{{ account_id }}:trail/*"
        }
      }
    },
    {
      "Sid": "Allow CloudTrail to describe key",
      "Effect": "Allow",
      "Principal": {
        "Service": "cloudtrail.amazonaws.com"
      },
      "Action": "kms:DescribeKey",
      "Resource": "*"
    },
    {
      "Sid": "Allow principals in the account to decrypt log files",
      "Effect": "Allow",
      "Principal": {
        "AWS": "*"
      },
      "Action": [
        "kms:Decrypt",
        "kms:ReEncryptFrom"
      ],
      "Resource": "*",
      "Condition": {
        "StringEquals": {
          "kms:CallerAccount": "{{ account_id }}"
        },
        "StringLike": {
          "kms:EncryptionContext:aws:cloudtrail:arn": "arn:aws:cloudtrail:*:{{ account_id }}:trail/*"
        }
      }
    },
    {
      "Sid": "Allow alias creation during setup",
      "Effect": "Allow",
      "Principal": {
        "AWS": "*"
      },
      "Action": "kms:CreateAlias",
      "Resource": "*",
      "Condition": {
        "StringEquals": {
          "kms:CallerAccount": "{{ account_id }}",
          "kms:ViaService": "ec2.eu-west-1.amazonaws.com"
        }
      }
    },
    {
      "Sid": "Enable cross account log decryption",
      "Effect": "Allow",
      "Principal": {
        "AWS": "*"
      },
      "Action": [
        "kms:Decrypt",
        "kms:ReEncryptFrom"
      ],
      "Resource": "*",
      "Condition": {
        "StringEquals": {
          "kms:CallerAccount": "{{ account_id }}"
        },
        "StringLike": {
          "kms:EncryptionContext:aws:cloudtrail:arn": "arn:aws:cloudtrail:*:{{ account_id }}:trail/*"
        }
      }
    }
  ]
}
```

It is also necessary a bucket S3 with the proper permissions to CloudTrail because it will be used to deliver the logs. The bucket
needs a policy like this:

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "AWSCloudTrailAclCheck20150319",
            "Effect": "Allow",
            "Principal": {"Service": "cloudtrail.amazonaws.com"},
            "Action": "s3:GetBucketAcl",
            "Resource": "arn:aws:s3:::{{ bucket_name }}"
        },
        {
            "Sid": "AWSCloudTrailWrite20150319",
            "Effect": "Allow",
            "Principal": {"Service": "cloudtrail.amazonaws.com"},
            "Action": "s3:PutObject",
            "Resource": "arn:aws:s3:::{{ bucket_name }}/cloudtrail/AWSLogs/{{ account_id }}/*",
            "Condition": {"StringEquals": {"s3:x-amz-acl": "bucket-owner-full-control"}}
        }
    ]
}

```

### Usage

If you want to use a predefined IAM role rather than create the 
default one you can assingn your own IAM role arn to the variable **cloudtrail_logs_role**.

### Example

This example will create the default role.

    - include_role:
          name: aidaonoro.aidaonoro.enable_cloud_trail
        vars:
          account_id: "123456789012"
          region: "us-west-1"
          bucket_name: "trailLoggingBucket"
          trail_tags: {MyTag: "MyTagValue"}
          cloud_trail_log_group: "CloudTrailLogs"
          kms_id: "arn:aws:kms:{{ region }}:{{ account_id }}:key/{{ rest_of_arn }}"
          trail_name: "trailName"

