---
layout: post
title: "Detect Public S3 Bucket using Splunk"
comments: false
excerpt_separator: <!--more-->
---
In today's post, we will learn how to detect a public S3 bucket using Splunk. Later, we will see how we can respond to such incidents and even prevent it from happening in the first place. As you will see in the following examples, there are multiple ways to create a S3 bucket and make it public. Also, for this blog, I created some subdomains of [logsec.cloud](https://www.logsec.cloud){:target="_blank"} in [Route53](https://aws.amazon.com/route53/ "AWS Route 53 Documentation"){:target="_blank"}.

<!--more-->
* 
{:toc}
### **Scenario 1: Using the AWS Web Console**
{:toc}
Create a S3 bucket and Go to "Permissions" - "Access Control List" - "Public Access" - "Everyone" - 
Access to the objects -  
- List Objects [\*]: Allows Gurantee to list the objects in the bucket  
- Write Objects [\*]: Allows Gurantee to create and delete the objects in the bucket

<a href="/assets/img/blog/2020_06_19_1.jpg" data-lightbox="blog_five">
<img src="/assets/img/blog/2020_06_19_1.jpg"/>
</a>

Go to "Permissions" - "Access Control List" - "Public Access" - "Everyone" - 
Access to this bucket's ACL - 
- Read bucket permissions: Allow Gurantee to read the bucket ACL
- Write bucket permissions: Allow Gurantee to edit the bucket ACL

To detect the first two scenarios, below Splunk search will work:

```
index="*" sourcetype="aws:cloudtrail"eventName=CreateBucket OR eventName=PutBucketAcl OR eventName=PutObjectAcl 
    requestParameters.AccessControlPolicy.AccessControlList.Grant{}.Grantee.URI="http://acs.amazonaws.com/groups/global/AllUsers" 
    OR requestParameters.AccessControlPolicy.AccessControlList.Grant{}.Grantee.URI="http://acs.amazonaws.com/groups/global/AuthenticatedUsers" 
| rename requestParameters.AccessControlPolicy.Owner.DisplayName as bucketOwner  
| eval time=strftime(_time,"%c %p")
| stats values(time) values(eventSource) values(eventName) values(eventType) values(errorCode) values(requestParameters.bucketName) values(bucketOwner) values(sourceIPAddress) values(userIdentity.arn) values(userIdentity.type) values(awsRegion) values(userIdentity.accountId) values(index)
```

### **Scenario 2: Using the AWS CLI to create a bucket with a public-read policy:**
```bash
$ aws s3api create-bucket --acl public-read --bucket BUCKET_NAME --region us-east-1
```
Other Possible values are: [CANNED ACLs]
- private
- public-read
- public-read-write  
- authenticated-read

Reference: [Create-Bucket](https://docs.aws.amazon.com/cli/latest/reference/s3api/create-bucket.html "AWS CLI Command Reference"){:target="_blank"} 

Testing the value: authenticated-read: Add the "Any AWS User" under the Public Access. Even though this option to add Authenticated User is now removed from the AWS Console, but you can still use it using the AWS CLI.

<a href="/assets/img/blog/2020_06_19_2.jpg" data-lightbox="blog_five">
<img src="/assets/img/blog/2020_06_19_2.jpg"/>
</a>

SPL:
```
"requestParameters.x-amz-acl{}"="public-*"
```

To detect both values: 
- public-read
- public-read-write

```bash
aws s3api create-bucket --acl authenticated-read --bucket BUCKET_NAME --region us-east-1
```

SPL:
```
"requestParameters.x-amz-acl{}"="authenticated-read"
```
Combining the events under Scenario 2, SPL will look like this:
```
index="*" sourcetype="aws:cloudtrail" eventName=CreateBucket OR eventName="Put*Acl" "requestParameters.x-amz-acl{}"="public-*" OR "requestParameters.x-amz-acl{}"="authenticated-read" 
| eval time=strftime(_time,"%c %p")
| stats values(time) values(eventSource) values(eventName) values(eventType) values(errorCode) values(requestParameters.bucketName) values(bucketOwner) values(sourceIPAddress) values(userIdentity.arn) values(userIdentity.type) values(awsRegion) values(userIdentity.accountId) values(index)
```

### **Scenario 3: Another option to make an S3 bucket public: by specifying the Grant ACP permissions via the command line**

CLI Command used:
```bash
aws s3api create-bucket --bucket BUCKET_NAME --region us-east-1 --grant-write-acp uri="http://acs.amazonaws.com/groups/global/AllUsers"
[--grant-full-control <value>]
[--grant-read <value>]
[--grant-read-acp <value>]
[--grant-write <value>]- 
[--grant-write-acp <value>]
```

Here `value` could be either "http://acs.amazonaws.com/groups/global/AuthenticatedUsers" OR "http://acs.amazonaws.com/groups/global/AllUsers"

Reference:  
[Using API-Level (s3api) commands with the AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/cli-services-s3-apicommands.html "s3 api Documentation"){:target="_blank"} 

SPL:
```
eventName=CreateBucket requestParameters.x-amz-grant-read{}="uri=http://acs.amazonaws.com/groups/global/AllUsers"
```
Combining all the posible values, SPL will look like:
```
(requestParameters.x-amz-grant-full-control{}="*AllUsers" OR requestParameters.x-amz-grant-full-control{}="*AuthenticatedUsers")
(requestParameters.x-amz-grant-read{}="*AllUsers" OR requestParameters.x-amz-grant-read{}="*AuthenticatedUsers") OR
(requestParameters.x-amz-grant-read-acp{}="*AllUsers" OR requestParameters.x-amz-grant-read-acp{}="*AuthenticatedUsers") OR
(requestParameters.x-amz-grant-write{}="*AllUsers" OR requestParameters.x-amz-grant-write{}="*AuthenticatedUsers") OR
(requestParameters.x-amz-grant-write-acp{}="*AllUsers" OR requestParameters.x-amz-grant-write-acp{}="*AuthenticatedUsers")
```
***
Consolidatig all the above searches, to detect all possible scenarios, our complete search will be:
```
index="*" sourcetype="aws:cloudtrail" eventName=CreateBucket OR eventName="Put*Acl"  
("requestParameters.AccessControlPolicy.AccessControlList.Grant{}.Grantee.URI"="http://acs.amazonaws.com/groups/global/AllUsers") OR  
(requestParameters.x-amz-grant-full-control{}="*AllUsers" OR requestParameters.x-amz-grant-full-control{}="*AuthenticatedUsers") OR
    (requestParameters.x-amz-grant-read{}="*AllUsers" OR requestParameters.x-amz-grant-read{}="*AuthenticatedUsers") OR
    (requestParameters.x-amz-grant-read-acp{}="*AllUsers" OR requestParameters.x-amz-grant-read-acp{}="*AuthenticatedUsers") OR
    (requestParameters.x-amz-grant-write{}="*AllUsers" OR requestParameters.x-amz-grant-write{}="*AuthenticatedUsers") OR
    (requestParameters.x-amz-grant-write-acp{}="*AllUsers" OR requestParameters.x-amz-grant-write-acp{}="*AuthenticatedUsers") 
| eval time=strftime(_time,"%c %p")
| stats values(time) values(eventSource) values(eventName) values(eventType) values(errorCode) values(requestParameters.bucketName) values(bucketOwner) values(sourceIPAddress) values(userIdentity.arn) values(userIdentity.type) values(awsRegion) values(userIdentity.accountId) values(index)
```

If you are importing the S3 Server Access Logs in to Splunk, you can use `geostats` command to generate statistics to display geographic data and summarize the data on maps.
```
index="*" sourcetype="aws:s3:accesslogs" 
| iplocation remote_ip
| geostats count BY remote_ip
```
<a href="/assets/img/blog/2020_06_19_3.jpg" data-lightbox="blog_five">
<img src="/assets/img/blog/2020_06_19_3.jpg"/>
</a>

**Known false positives:** There could be undesired alerts that can occur from this search, when someone intentionally creates a public bucket. In this case, consider adding speicific buckets to whitelist.

**How to respond:** When this alert fires, ask yourself these questions. Is it still public? This is easy to detect, just search the logs for the bucket name and PutBucketACL, to see any subsequent ACL changes. Are the files public? and What is in it? These can be detected using S3 Access logs.

**How to prevent:** Configure S3 Block Public Access on the AWS account level (applies to all S3 buckets in all regions). S3 Block Public Access provides four settings:

- **Block Public ACLs**: Prevent any new operations to make buckets or objects public through Bucket or Object ACLs. (existing policies and ACLs for buckets and objects are not modified.)
- **Ignore Public ACLs**: Ignore all public ACLs on a bucket and any objects that it contains
- **Block Public Policy**: Reject calls to PUT Bucket policy if the specified bucket policy allows public access. (Enabling this setting doesn't affect existing bucket policies)
- **Restrict Public Buckets**: Restrict access to a bucket with a public policy to only AWS services and authorized users within the bucket owner's account. 

```terraform
provider "aws" {
}

resource "aws_s3_account_public_access_block" "BlockPublicAccess" {
  block_public_acls = "true"
  ignore_public_acls = "true"
  block_public_policy = "true"
  restrict_public_buckets = "true"
}
```