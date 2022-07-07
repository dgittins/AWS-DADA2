# AWS DADA2  

Instructions for running DADA2 on an AWS EC2 instance  

Prerequisites:

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;a. Create an [AWS account](https://aws.amazon.com/premiumsupport/knowledge-center/create-and-activate-aws-account/)  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;b. Set [IAM permissions](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_users_change-permissions.html) to allow Amazon EC2 and Amazon S3 access  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;c. Install and configure [AWS Command Line Interface](https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-getting-started.html) (CLI)  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;d. Create and configure an [Amazon Virtual Private Cloud](https://docs.aws.amazon.com/directoryservice/latest/admin-guide/gsg_create_vpc.html) (Amazon VPC)  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;e. Create an [Amazon EC2 key pair](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-key-pairs.html)<br/><br/>

## 1. Create an EC2 instance running RStudio  

A couple of options here:<br/><br/>

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;1.a  Create EC2 instance with an Amazon Linux AMI and configure RStudio  

```
$ aws ec2 run-instances --image-id ami-0c159d337b331627c --count 1 --instance-type t2.micro --key-name <key pair name> --security-group-ids <security group ID> --subnet-id <subnet ID> --tag-specifications ResourceType=instance,Tags='[{Key=Name,Value=DADA2}]'
``` 

Follow AWS RStudio configuration workflow: https://github.com/dgittins/AWS-RStudio<br/><br/>

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;1.b Create an EC2 instance with an RStudio AMI  

Louis Aslett’s (awesome!) website provides RStudio AMIs for different regions: http://www.louisaslett.com/RStudio_AMI/  

```
$ aws ec2 run-instances --image-id ami-0315888c660b24d7c --count 1 --instance-type t2.micro --key-name <key pair name> --security-group-ids <security group ID> --subnet-id <subnet ID> --tag-specifications ResourceType=instance,Tags='[{Key=Name,Value=DADA2}]'
```
<br/>

Parameters:

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;**--image-id:** 'AMI catalog' in the EC2 portal       
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;**--key-name:** 'Key Pairs' in the EC2 portal  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;**--security-group-ids:** 'Security Groups' in the EC2 portal   
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;**--subnet-id:** 'Subnets' in VPC portal  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;**--tag-specifications:** provide an instance name, e.g., 'DADA2'

#### Check the instance is running 

```
$ aws ec2 describe-instances --filters "Name=tag:Name,Values=DADA2"
```

*Record instance ID  
*Record PublicDnsName  

#### Connect to EC2 instance using Secure Shell Protocol (SSH)  

```
$ ssh -i </path/to/my-key-pair.pem> ubuntu@<my-instance-public-dns-name>
```  

Default usernames for different AMIs are listed here: https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/connection-prereqs.html  
