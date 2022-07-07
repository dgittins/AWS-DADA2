# AWS DADA2  

Workflow for running [DADA2](https://benjjneb.github.io/dada2/index.html) on an AWS EC2 instance (includes downloading and pre-processing sequences from the [Sequence read Archive](https://www.ncbi.nlm.nih.gov/sra))  

Prerequisites:

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;a. Create an [AWS account](https://aws.amazon.com/premiumsupport/knowledge-center/create-and-activate-aws-account/)  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;b. Set [IAM permissions](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_users_change-permissions.html) to allow Amazon EC2 and Amazon S3 access  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;c. Install and configure [AWS Command Line Interface](https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-getting-started.html) (CLI)  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;d. Create and configure an [Amazon Virtual Private Cloud](https://docs.aws.amazon.com/directoryservice/latest/admin-guide/gsg_create_vpc.html) (Amazon VPC)  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;e. Create an [Amazon EC2 key pair](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-key-pairs.html)<br/><br/>

## 1. Configure a security group that has ports for SSH, HTTP, RStudio<br/>

A security group acts as a virtual firewall for an EC2 instance to control incoming and outgoing traffic. Security groups can be created using the [Amazon VPC console](https://console.aws.amazon.com/vpc/) or using the [AWS CLI](https://docs.aws.amazon.com/cli/latest/reference/ec2/create-security-group.html).  

Example security group:  

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;**Security group name:** RStudio-security-group  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;**Description:** Allow SSH, HTTP, RStudio  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;**VPC:** &lt;vpc ID&gt;

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Inbound rule 1 - SSH, Anywhere-IPv4, port 22  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Inbound rule 2 - HTTP, Anywhere-IPv4, port 80  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Inbound rule 3 - Custom TCP, Anywhere-IPv4, port 8787 (RStudio)

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Outbound rule 1 - All traffic  

*Record security group ID<br/><br/>

## 2. Create an EC2 instance running RStudio  

A couple of options here:<br/><br/>

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;**2.a  Create EC2 instance with an Amazon Linux AMI and configure RStudio**  

Free tier eligible AMI:  
Ubuntu Amazon Machine Image (AMI) Ubuntu Server 18.04 LTS (HVM), SSD Volume Type - ami-0c159d337b331627c (64-bit (x86)) 

```
$ aws ec2 run-instances --image-id ami-0c159d337b331627c --count 1 --instance-type t2.micro --key-name <key pair name> --security-group-ids <security group ID> --subnet-id <subnet ID> --tag-specifications ResourceType=instance,Tags='[{Key=Name,Value=DADA2}]'
``` 

Then follow AWS RStudio configuration workflow: https://github.com/dgittins/AWS-RStudio<br/><br/>

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;**2.b Create an EC2 instance with an RStudio AMI**  

Louis Aslett’s (awesome!) website provides RStudio AMIs for different regions: http://www.louisaslett.com/RStudio_AMI/  

```
$ aws ec2 run-instances --image-id ami-0315888c660b24d7c --count 1 --instance-type t2.micro --key-name <key pair name> --security-group-ids <security group ID> --subnet-id <subnet ID> --tag-specifications ResourceType=instance,Tags='[{Key=Name,Value=DADA2}]'
```
<br/>

Parameters:

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;**--image-id:** 'AMI catalog' in the EC2 portal       
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;**--instance-type:** Choose an instance with suitable compute, memory and networking resources: https://aws.amazon.com/ec2/instance-types/  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;**--key-name:** 'Key Pairs' in the EC2 portal  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;**--security-group-ids:** 'Security Groups' in the EC2 portal   
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;**--subnet-id:** 'Subnets' in VPC portal  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;**--tag-specifications:** provide an instance name, e.g., 'DADA2'  
<br/><br/>


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

## 3. Install DADA2  

More information: https://benjjneb.github.io/dada2/dada-installation.html  

```
$ R
$ library("devtools")
$ devtools::install_github("benjjneb/dada2", ref="v1.20")
```

Verify installation:

```
$ packageVersion("dada2")
```

## 4. Install seqinr

'''
$ R
$ install.packages("seqinr")
'''

## 5. Download DADA2-formatted reference database

Sequence classification requires a reference training dataset: https://benjjneb.github.io/dada2/training.html  

$ wget https://zenodo.org/record/4587955/files/silva_nr99_v138.1_train_set.fa.gz?download=1

## 6. Install SRA Toolkit  

Instructions provided here: https://github.com/ncbi/sra-tools/wiki/02.-Installing-SRA-Toolkit  

Download the SRA Toolkit TarBall for Ubuntu:

```
$ wget --output-document sratoolkit.tar.gz https://ftp-trace.ncbi.nlm.nih.gov/sra/sdk/current/sratoolkit.current-ubuntu64.tar.gz
```

Extract the tar file:

```
$ tar -vxzf sratoolkit.tar.gz
$ rm sratoolkit.tar.gz
```

Append the path to the binaries to PATH environment variable

```
$ export PATH=$PATH:/home/ubuntu/sratoolkit.3.0.0-ubuntu64/bin
```

Validate

```
$ echo $PATH
$ which fastq-dump
```

## 7. Configure SRA ToolKit (https://github.com/ncbi/sra-tools/wiki/03.-Quick-Toolkit-Configuration)
$ mkdir sradownloads (/home/ubuntu/sradownloads - this will be used as the download path *could also use S3 bucket)
$ vdb-config --interactive

CACHE - location of user repository: /home/ubuntu/sradownloads
AWS - select 'accept charges for AWS' and 'report cloud instance identity'
Save changes and Exit

-Verify that the toolkit is functional
$ fastq-dump --stdout SRR390728 | head -n 8 

#Install cutadapt
$ sudo apt install cutadapt
$ cutadapt --version
