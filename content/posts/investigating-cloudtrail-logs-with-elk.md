---
title: "Investigating CloudTrail Logs using ELK Stack"
date: 2021-04-20
description: "Seven Kibana (KQL) queries for spotting suspicious AWS CloudTrail activity, from disabled logging to backdoor access keys, mapped to MITRE ATT&CK."
tags: [aws, cloudtrail, elastic, kibana, detection, mitre-attack]
ShowToc: true
---
AWS CloudTrail is an AWS service that helps you enable governance, compliance, and operational and risk auditing of your AWS account. Actions taken by a user, role, or an AWS service are recorded as events in CloudTrail. Events include actions taken in the AWS Management Console, AWS Command Line Interface, and AWS SDKs and APIs.

You can ship these CloudTrail logs into the ELK stack and learn how to visualize these events, near real time, using Kibana. Here are some of the [KQL](https://www.elastic.co/guide/en/kibana/current/kuery-query.html) (Kibana Query Language) queries that you can use to search for some key CloudTrail events to monitor for Security in AWS.  
 
Please note that I'm using the [index pattern](https://www.elastic.co/guide/en/kibana/current/index-patterns.html) `cloudwatch*` for the CloudTrail logs.

## 1. CloudTrail Logging Turned Off
Identifies the deletion of an AWS log trail. An adversary may delete trails in an attempt to evade defenses.

KQL Query:

```
_index:cloudwatch* and eventName:(DeleteTrail or UpdateTrail or StopLogging)
```

**MITRE ATT&CK Tactic:** Defense Evasion  
**MITRE ATT&CK Techniques:** [T1562](https://attack.mitre.org/techniques/T1562/001/) - Impair Defenses


## 2. Delete Attempts on CloudTrail S3 Buckets
Identifies the deletion of an AWS S3 bucket containing the CloudTrail logs. You can update the `requestParameters.bucketName` as per the naming convention for the S3 buckets storing CloudTrail logs. An adversary may delete trails in an attempt to evade defenses.

KQL Query:

```
_index:cloudwatch* AND eventName:(DeleteBucket OR DeleteObject OR DeleteObjects) AND requestParameters.bucketName: *logs
```

**MITRE ATT&CK Tactics:** Impact  
**MITRE ATT&CK Techniques:** [T1485](https://attack.mitre.org/techniques/T1485/) - (Data Destruction)


## 3. Creation of New IAM User
Detects the creation of a new user. In a production environment, you will most probably be using federated sign-in through Active Directory (AD) and Active Directory Federation Services (ADFS), instead of local IAM users. With the help of this search, you can easily search if someone creates a new local user.

KQL Query:

```
_index:cloudwatch* AND eventName:CreateUser AND eventSource:iam.amazonaws.com
```

**MITRE ATT&CK Tactics:** Persistence  
**MITRE ATT&CK Techniques:** [T1136](https://attack.mitre.org/techniques/T1136/) - Create Account


## 4. Cross Account VPC Peering Connections
A [VPC peering](https://docs.aws.amazon.com/vpc/latest/peering/what-is-vpc-peering.html) connection is a networking connection between two VPCs that enables you to route traffic between them using private IPv4 addresses or IPv6 addresses. Instances in either VPC can communicate with each other as if they are within the same network. You can create a VPC peering connection between your own VPCs, or with a VPC in another AWS account. The VPCs can be in different regions (also known as an inter-region VPC peering connection).

This can produce false positives for cases where VPC peering is created for legitimate purposes, but you can look for events where a connection is made to a VPC outside of your organization.

KQL Query:

```
_index:cloudwatch* and eventSource:"ec2.amazonaws.com" and eventName:"CreateVpcPeeringConnection"
```

**MITRE ATT&CK Tactics:** Initial Access  
**MITRE ATT&CK Techniques:** [T1199](https://attack.mitre.org/techniques/T1199/)

## 5. AWS IAM Backdoor Users Keys
Detects AWS API key creation for a user by another user. Backdoored users can be used to obtain persistence in the AWS environment. Also with this alert, you can detect a flow of AWS keys in your org.

KQL Query:

```
((eventSource:"iam.amazonaws.com" AND eventName:"CreateAccessKey") AND (NOT (userIdentity.arn.keyword:*responseElements.accessKey.userName*)))
```

**MITRE ATT&CK Tactics:** Persistence  
**MITRE ATT&CK Techniques:** [T1098](https://attack.mitre.org/techniques/T1098/)  
**Sub-Technique:** T1098.001 Additional Cloud Credentials

## 6. AWS EC2 Startup Shell Script Change
Detects changes to the EC2 instance startup script. The shell script will be executed as root/SYSTEM every time the specific instances are booted up.

KQL Query:

```
(eventSource:"ec2.amazonaws.com" AND requestParameters.userData.keyword:* AND eventName:"ModifyInstanceAttribute")
```

## 7. AWS - Retrieve Secret on Secrets Manager Detected
This rule identifies when users attempt to access the secrets in AWS Secrets Manager to steal certificates, credentials, or other sensitive material.

KQL Query:

```
(eventSource:"secretsmanager.amazonaws.com" AND eventName:"GetSecretValue")
```

**MITRE ATT&CK Tactics:** Credential Access  
**MITRE ATT&CK Techniques:** [T1528](https://attack.mitre.org/techniques/T1528/)

***
