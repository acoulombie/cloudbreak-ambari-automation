## Cloudbreak / Ambari Automation Installation Guide

Author: Ritesh Patel | Last Updated: Oct 28, 2017

## Overview

This guide lists steps required for automating CloudBreak, Ambari and Hadoop Cluster. 

#### Automation Approach

For this automation I have used CloudFormation template and Ansible playbook. 

1) CloudFormation template is used to spin up CloudBreak instance. This template will generate dynamic script to install Ambari / Hadoop cluster
2) Playbook to execute CloudFormation template & Ambari / Hadoop cluster script. Ambari installation script is executed using CloudBreak Shell only.

*Each resource will be assigned a unique name by appending a UUID*

#### Dependencies 

##### IAM Role
IAM role has been created beforehand. This role will be attached to the CloudBreak InstanceProfile. Role has EC2 and S3 policies attached. CloudBreak instance will access AWS CLI through this role.

##### Vpc & Subnet
Vpc & Subnet must be created beforehand for the CloudBreak instance. Hadoop cluster will use the same Vpc but you must provide a Subnet Cidr to create a new Subnet under this Vpc. Install will launch Hadoop Cluster nodes in the new Subnet.

##### Security Groups
Security group for CloudBreak instance must be created beforehand. For Hadoop Cluster nodes this install will use a default Security Group provided by CloudBreak.

##### S3 Bucket
BluePrint and KeyPair used for this automation are stored in an S3 bucket. Access to this S3 bucket is permitted via Vpc Endpoints. Below is a sample policy on the Vpc Endpoint.

```JSON
{
    "Version": "2008-10-17",
    "Statement": [
        {
            "Sid": "Access-to-blueprint-bucket-only",
            "Effect": "Allow",
            "Principal": "*",
            "Action": [
                "s3:GetObject",
                "s3:ListBucket"
            ],
            "Resource": [
                "arn:aws:s3:::bucket_name",
                "arn:aws:s3:::bucket_name/*"
            ]
        }
    ]
}
```
##### Pre-Baked AMI for CloudBreak Instance

This template uses a Pre-Baked AMI for CloudBreak instance. AMI is launched on a m4.xlarge instance.

| Region | Image Id |
|---------------|------|
| us-east-1 |  ami-10a78606 |
| us-west-1 |  ami-01735e61 |
| us-west-2 |  ami-6ccbc015 |

##### CloudBreak Shell

Hadoop / Ambari install script must be executed using CloudBreak Shell. This shell must be initiated from /var/lib/cloudbreak-deployment. This is just for information purposes. Install is fully equipped to interact with CloudBreak shell. 

#### Resources Created By this template

| Resource | Description | DependsOn |
|---------------|------|-------------|
| rCloudBreakInstance |  CloudBreak Instance | rIAMInstanceProfile, rDeployerRole |
| rPublicIp |  Public IP Address | N/A |
| rAssociateEip |  Elastic IP association | rPublicIp |
| rIAMInstanceProfile |  Instance profile for the CloudBreak Instance | RAP_CLI_ROLE |
| rCloudBreakUser |  CloudBreak user for managing CBD process | N/A |
| rAccessKey |  CloudBreak user's access credentials for CBD Profile | rCloudBreakUser |
| rDeployreRole |  This role can be created using "cbd generate <role>". We are creating this role through resources. | N/A |

#### Mandatory & Optional Parameters

| Parameter Name | Purpose | Default Value | Type |
|---------------|------|-------------------|------|
| pApplicationName |  Application name | Dev-Cloudbreak-AZa | Optional |
| pVpcTenancy |  VPC tenancy | dedicated | Optional |
| pApplicationIdentifier |  Application identifier | cloudbreak | Optional |
| pUniqueHash | Hash to generate unique resource names | dedicated | *Automated* |
| pEnvironment | Stack environment | development | Optional |
| pTagTechnicalContact |  Technical contact for this template | Ritesh Patel | Optional |
| pVpcId | Vpc Identifier to launch resources | vpc-7fc41c07 | **Replace** |
| pSubnetId | Subnet Identifier to launch resources | subnet-4b3ed364 | **Replace** |
| pRegion | AWS Region | us-east-1 | Optional |
| pInstanceType | CloudBreak instance type | m4.xlarge | Optional |
| pImageId | CloudBreak pre-baked AMI | ami-10a78606 | Optional |
| pKeypair | Keypair to launch resources | mosaic-ritesh | **Replace** |
| pClusterInstanceType | Hadoop cluster instance type | m4.xlarge | Optional |
| pClusterSubnetCidr | Subnet Cidr for Hadoop Cluster | 10.0.7.0/24 | **Replace** |
| pSecurityGroup | Security group for CloudBreak Instance | sg-68b5ca1a | **Replace** |
| pClusterSecurityGroupName | Security group name for Hadoop Cluster nodes | default-aws-only-ssh-and-ssl | Optional |
| pInternetGatewayId | Internet gateway identifier to attach with Hadoop cluster subnet | igw-ff8c2386 | **Replace** |
| pVolumeSize | Hadoop cluster node volume size | 50 | Optional |
| pVolumeCount | Hadoop cluster node volume count | 2 | Optional |
| pBluePrintName | Hadoop Cluster blue print (stored in S3 bucket) | hdp-small-cluster | **Replace** |
| pS3EndPoint | S3 Endpoint | s3://rap-cloudbreak-endpoint | **Replace** |

#### Stack Outputs

Stack output provided by the CloudFormation template.

| Resource |
|----------|
| rAccessKey |
| rIAMInstanceProfile |
| rCloudBreakInstance |
| rCloudBreakInstance |
| rPublicIp |
| rPublicDns |
| rDeployerRole |

Sample output.

```JSON
{
    "msg": {
        "rAccessKey": "AKIA********************",
        "rCloudBreakInstance": "i-06786967978698",
        "rDeployerRole": "arn:aws:iam::*************:role/cloudbreak-deployer-18b97bda-fcec-5328-a2c1-acfdc3a9a92e",
        "rIAMInstanceProfile": "Cloudbreak-Instance-Profile-18b97bda-fcec-5328-a2c1-acfdc3a9a92e",
        "rPublicDns": "ec2-12-34-567-789.compute-1.amazonaws.com",
        "rPublicIp": "12.34.567.890"
    }
}
```

#### Playbooks & Roles

Playbook uses **ansible-vault** to generate encrypted credentials. These credentials are passed into plays. 

**Roles**

This playbook uses only one role: cft-cloudbreak

**Vars**

Encrypted Credentials

**Playbook Usage**

Shell command for executing playbook.

```shell
ritesh.patel@my-macbook-pro> ansible-playbook main.yml --valut-password-file password.txt
```

#### Hadoop / Ambari Install Progress
To track progress on Hadoop Cluster log into CloudBreak UI and click on the Cluster. CloudBreak UI will show you events history as it moves on with the install.

**Sample Events History**
```Shell
10/28/2017 1:26:12 PM mosaic-cms-hadoop-stack - create in progress: Setting up HDP image
10/28/2017 1:26:12 PM mosaic-cms-hadoop-stack - create in progress: Creating infrastructure
10/28/2017 1:27:54 PM mosaic-cms-hadoop-stack - update in progress: Infrastructure creation took 102 seconds
10/28/2017 1:27:56 PM mosaic-cms-hadoop-stack - update in progress: Infrastructure metadata collection finished
10/28/2017 1:28:10 PM mosaic-cms-hadoop-stack - available: Infrastructure successfully provisioned
10/28/2017 1:28:32 PM mosaic-cms-hadoop-stack - update in progress: Bootstrapping infrastructure cluster
10/28/2017 1:29:00 PM mosaic-cms-hadoop-stack - update in progress: Setting up infrastructure metadata
10/28/2017 1:29:01 PM mosaic-cms-hadoop-stack - update in progress: Starting Ambari cluster services
10/28/2017 1:32:15 PM mosaic-cms-hadoop-stack - update in progress: Building Ambari cluster; Ambari ip:34.342.212.113
10/28/2017 1:42:17 PM mosaic-cms-hadoop-stack - available: Ambari cluster built; Ambari ip:34.342.212.113
```
#### CloudBreak UI
Upon completing the install CloudBreak and Ambari UI will be accessible through browsers. Use rPublicIp to login to CloudBreak UI. CloudBreak UI allows you to create additional accounts as required. 

CloudBreak UI endpoint: **https://{{rPublicIp}}/**
(replace rPublicIp with an actual Ip Address)
Credentials: **admin@example.com / password**

#### Ambari UI
From CloudBreak UI, select the "Cluster" to retrieve Ambari endpoint. 

Ambari credentials: **admin/admin**

** *This stack completes execution in about 30 minutes*

#### Issues
Gimme a holler if you find one :) :)

#### Suggestions
I love new ideas & believe me I am **all ears** for any suggestions.