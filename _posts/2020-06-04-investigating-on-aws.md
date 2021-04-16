---
layout: post
title: "Investigating S3 Scanning Activites on AWS"
comments: true
excerpt_separator: <!--more-->
---
During the past week, we detected some suspicious activities across multiple AWS accounts in one of our client's environment. These activites seems related to scanning activities from a bad actor on S3 buckets. One of the sample logs from S3 Access logs:
<!--more-->
```
84fd7086179626a759fb59a0252a26d26dc1685e30f0ab922266b5abace8f998 <AWS-ACCOUNT-ID>-config-org-bucket [01/Jun/2020:13:27:34 +0000]  
10.247.13.187 arn:aws:iam::799199334739:user/starling-ladyblackhawk-iad-prod 9WDX6MBZFQ6Z6RDR REST.HEAD.BUCKET  
- "HEAD / HTTP/1.1" 403 AccessDenied 243 - 9 - "-" "AWSConfig" - wFvPfIaSbdpcnNMdcTj+BNWn00r3OfqHV8sTt8gUXMzLA7O/MiWg+NPckix2TThK15V/p1JigVc=  
SigV4 ECDHE-RSA-AES128-SHA AuthHeader <AWS-ACCOUNT-ID>-config-org-bucket.s3.amazonaws.com TLSv1.2
```

First thing that got our attention to this specific log is the username `starling-ladyblack-iad-prod`, question to ask at this stage is this a rouge IAM user? We do not follow such naming convention across the organization. Second, looking at the AWS account id `799199334739` - this account does not belongs to us. So, definitely this is not a rougue user in our environement. 

Next, extending the date time range in Splunk used the below search to look for `http_status` other than 403. Fortunantely, there were no 200, but these activities were continuous from last couple of months. 

```
index="aws" sourcetype="aws:s3:accesslogs"
| table error_code http_status operation bucket_name remote_ip requester request_uri user_agent index
```

Other interesting thing to notice in the S3 access logs is the `user agent` is `AWSConfig` and remote IP is private. When we access S3 bucket using AWS CLI, remote IP is the public IP of the user machine. But in this case, it was private. To gather more inforamtion about this behavior, search for [S3 access logs format](https://docs.aws.amazon.com/AmazonS3/latest/dev/LogFormat.html "AWS Documentation"){:target="_blank"}. As per AWS documentation:

> Remote IP: The apparent internet address of the requester. Intermediate proxies and firewalls might obscure the actual address of the machine making the request.

Next step was to determine the reason behind user agent AWS Config. Is this some kind of misconfiguration on someone's AWS account or someone trying to manipulate the user Agent or maybe event using AWS config for these scanning activities? To answer these questions, I tried to replicate the same behaviour using AWS Config in our learning environment.

In Account A, created two S3 buckets named **`AWS-ACCOUNT-ID`-config-org-bucket** and **`AWS-ACCOUNT-ID`-s3-access-logs**. Turned on the `Server access logging` for first bucket and selected second bucket as the target. Block public access flag is enabled on account level to ensure that public access to all S3 buckets and objects is blocked. Refer to [Using Amazon S3 block public access](https://docs.aws.amazon.com/AmazonS3/latest/dev/access-control-block-public-access.html "AWS Documentation"){:target="_blank"}.

In Account B, Turned ON the recording under Settings for AWS Config as shown in diagram below, it works when we select an S3 buckets that exists and have the proper permissions. You can look for the correct permissions at [S3 bucket policy](https://docs.aws.amazon.com/config/latest/developerguide/s3-bucket-policy.html "AWS Documentation"){:target="_blank"}.

<a href="/assets/img/blog/2020_06_03_00.jpg" data-lightbox="blog_four">
<img src="/assets/img/blog/2020_06_03_00.jpg"/>
</a>

But when we attempt to select a bucket **`AWS-ACCOUNT-ID`-config-org-bucket** from account A that does not exist or not have the correct bucket policy, it throws an error:

<a href="/assets/img/blog/2020_06_03_01.jpg" data-lightbox="blog_four">
<img src="/assets/img/blog/2020_06_03_01.jpg"/>
</a>

Next, log in to Splunk and looked in to investigate S3 access logs for **`AWS-ACCOUNT-ID`-s3-access-logs** in account A. Using the same search we used earlier, found out a similar pattern in the S3 access logs:

```
84fd7086179626a759fb59a0252a26d26dc1685e30f0ab922266b5abace8f998 <AWS-ACCOUNT-ID>-config-org-bucket [03/Jun/2020:03:12:08 +0000]  
10.247.239.161 arn:aws:sts::xxxxxxxxxxxx:assumed-role/account/AWSConfig-Describe 944A6758900290DD  
REST.GET.ACCELERATE - "GET /?accelerate HTTP/1.1" 200 - 113 - 10 - "-" "AWSConfig" -  
joUWdNULBFcczofcQ4HDKGOMGCSy4mCVeAh/oc66yuelKFYnazB6dftz2d6c91GXfZRqtNNmXCc= SigV4  
ECDHE-RSA-AES128-SHA AuthHeader <AWS-ACCOUNT-ID>-config-org-bucket.s3.us-east-1.amazonaws.com TLSv1.2
```
As this log is similar to what we have seen in the client's environment, we can be sure, either someone misconfigured their AWS Config `Amazon S3 Bucket` setting to write their logs to our bucket or using this to determine if a particular S3 buckets exisits or not. Also, now we are confident regarding the user agent as AWS Config and remote IP is a private IP, which is based on VPC CIDR range from Account B.

To take this a step further, we tried to write a script to test if a potential hacker can use this for malicious purposes - like reconnaissance activites for S3 buckets. Please note there are various tools available to validate S3 buckets like <https://github.com/dagrz/aws_pwn>{:target="_blank"}, but I'm not able to find anything using AWS Config.


Below script takes two inputs, `-i` for text file with bucket names like `bucket_names.txt` (one word per line) and `-o` to output the results in to a file in a json format, like `output.json`. Replace the `Role_ARN` with the IAM Role ARN you want to use for AWS Config.

```bash
maven@pluto:~$ ./config.py -h
usage: config.py [-h] -i INPUT_FILE [-o OUTPUT_FILE]

Validate existence of a S3 bucket.

optional arguments:
  -h, --help     show this help message and exit
  -i INPUT_FILE, --input-file INPUT_FILE
  -o OUTPUT_FILE, --output-file OUTPUT_FILE
```
Example: 
```bash 
maven@pluto:~$ ./config.py -i bucket_names.txt -o output.json
```

```python
#!/usr/bin/env python3
import boto3
import json
import argparse


def main(args):

    role_arn = "ROLE_ARN"

    all_results = []
    for line in args.input_file.readlines():
        line = line.strip()
        if(line and not line.startswith('#')):
            result = validate_s3_bucket(line, role_arn)
            all_results.append(result)
            print("Bucket: %s" % line)
            print(json.dumps(result, indent=2))
            print("#" * 10)
    args.input_file.close()

    if(args.output_file is not None):
        args.output_file.write(json.dumps(all_results, indent=2))
        args.output_file.close()


def validate_s3_bucket(bucket_name, role_arn):
    config_client = boto3.client('config', region_name='us-east-1')

    response = config_client.start_configuration_recorder(
        ConfigurationRecorderName='default'
    )

    response = config_client.put_configuration_recorder(
        ConfigurationRecorder={
            'name': 'default',
            'recordingGroup': {
                'allSupported': True,
                'includeGlobalResourceTypes': True
            },
            'roleARN': role_arn
        }
    )

    try:
        result = config_client.put_delivery_channel(
            DeliveryChannel={
                'name': 'default',
                's3BucketName': bucket_name
            }
        )
    except config_client.exceptions.InsufficientDeliveryPolicyException as e:
        result = (
            "Your Amazon S3 bucket policy does not permit AWS Config to write to it.: %s" % e)
    except config_client.exceptions.NoSuchBucketException as e:
        result = ("The specified Amazon S3 bucket does not exist.: %s" % e)
    except ClientError as e:
        result = ("Unexpected error: %s" % e)
    return result


if __name__ == '__main__':
    parser = argparse.ArgumentParser(
        description="Validate existence of a S3 bucket.")
    parser.add_argument('-i',
                        '--input-file',
                        type=argparse.FileType('r'),
                        required=True)
    parser.add_argument('-o',
                        '--output-file',
                        type=argparse.FileType('w'))
    args = parser.parse_args()
    main(args)
```

While testing in our learning environment, found another similar user from same AWS Account `arn:aws:iam::799199334739:user/starling-dove-iad-prod`. As of now, we're seeing these two different IAM users from the same AWS Account `799199334739`. As of now, not sure who is the actual owner of this account or this account belongs to AWS itself. To be continued...

```
index="aws" sourcetype="aws:s3:accesslogs" requester="arn:aws:iam::*:user/starling-*" 
| stats count BY requester
```
<a href="/assets/img/blog/2020_06_03_02.jpg" data-lightbox="blog_four">
<img src="/assets/img/blog/2020_06_03_02.jpg"/>
</a>