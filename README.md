# AWS DADA2  

Workflow for running [DADA2](https://benjjneb.github.io/dada2/index.html) on an AWS EC2 instance (includes downloading and pre-processing sequences from the [NCBIs Sequence Read Archive](https://www.ncbi.nlm.nih.gov/sra) and transferring processed files to [Amazon S3](https://aws.amazon.com/s3/) cloud storage)  

Prerequisites:

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;a. Create an [AWS account](https://aws.amazon.com/premiumsupport/knowledge-center/create-and-activate-aws-account/)  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;b. Set [IAM permissions](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_users_change-permissions.html) to allow Amazon EC2 access and Amazon S3 access  
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

Louis Aslett’s (awesome!) website (http://www.louisaslett.com/RStudio_AMI/) provides RStudio AMIs for different regions.  

```
$ aws ec2 run-instances --image-id ami-0315888c660b24d7c --count 1 --instance-type t2.micro --key-name <key pair name> --security-group-ids <security group ID> --subnet-id <subnet ID> --tag-specifications ResourceType=instance,Tags='[{Key=Name,Value=DADA2}]' --block-device-mappings 'DeviceName=/dev/sda1, Ebs={VolumeSize=50}'
```
<br/>

Parameters:

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;**--image-id:** 'AMI catalog' in the EC2 portal       
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;**--instance-type:** Choose an instance with suitable resources: https://aws.amazon.com/ec2/instance-types/  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;**--key-name:** 'Key Pairs' in the EC2 portal  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;**--security-group-ids:** 'Security Groups' in the EC2 portal   
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;**--subnet-id:** 'Subnets' in VPC portal  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;**--tag-specifications:** option to provide an instance name, e.g., 'DADA2'  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;**--block-device-mappings:** option to increase the size of the EC2 instance storage volume (useful when processing large datasets)
<br/>


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

Default usernames for different AMIs are listed here: https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/connection-prereqs.html<br/><br/>  

## 3. Install DADA2  

More information: https://benjjneb.github.io/dada2/dada-installation.html  

```
$ R
$ library("devtools")
$ devtools::install_github("benjjneb/dada2", ref="v1.20")
```

#### Verify installation:

```
$ packageVersion("dada2")
```

## 4. Install seqinr  

[seqinr package](https://cran.r-project.org/web/packages/seqinr/index.html used to write sequences into a FASTA format file.

```
$ R
$ install.packages("seqinr")
```

## 5. Download DADA2-formatted reference database

Sequence classification requires a reference training dataset: https://benjjneb.github.io/dada2/training.html. Update the command with the required reference dataset.  

```
$ wget https://zenodo.org/record/4587955/files/silva_nr99_v138.1_train_set.fa.gz?download=1
```

## 6. Install SRA Toolkit  

More information: https://github.com/ncbi/sra-tools/wiki/02.-Installing-SRA-Toolkit  

#### Download the SRA Toolkit for Ubuntu:

```
$ wget --output-document sratoolkit.tar.gz https://ftp-trace.ncbi.nlm.nih.gov/sra/sdk/current/sratoolkit.current-ubuntu64.tar.gz
$ tar -vxzf sratoolkit.tar.gz
$ rm sratoolkit.tar.gz
```

#### Append the path to the binaries to PATH environment variable

```
$ export PATH=$PATH:/home/ubuntu/sratoolkit.3.0.0-ubuntu64/bin
```

#### Validate installation

```
$ echo $PATH
$ which fastq-dump
```

## 7. Configure SRA ToolKit  

More information: https://github.com/ncbi/sra-tools/wiki/03.-Quick-Toolkit-Configuration  

```
$ mkdir sradownloads (/home/ubuntu/sradownloads - this will be used as the download path *could also use S3 bucket)
$ vdb-config --interactive
```

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;**CACHE** - location of user repository: /home/ubuntu/sradownloads  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;**AWS** - select 'accept charges for AWS' and 'report cloud instance identity'  

#### Verify that the toolkit is functional  

```
$ fastq-dump --stdout SRR390728 | head -n 8 
```  

## 8. Install cutadapt  

[Cutadapt](https://cutadapt.readthedocs.io/en/stable/) removes adapter sequences and primers from high-throughput sequencing reads.  

```
$ sudo apt install cutadapt
$ cutadapt --version
```
<br/>

## Option - Create an AMI from the EC2 Instance  

To save the installed programs and settings, an AMI can be created using the [EC2 portal](https://docs.aws.amazon.com/toolkit-for-visual-studio/latest/user-guide/tkv-create-ami-from-instance.html) or the [AWS CLI](https://awscli.amazonaws.com/v2/documentation/api/2.0.34/reference/ec2/create-image.html). This AMI could be used as a starting point for future sequence processing. 

Example AMI:  

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;**Name:** DADA2-Ubuntu-Server-18.04-LTS-(HVM)-SSD-Volume  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;**Description:** DADA2 Ubuntu Server 18.04 LTS (HVM), SSD Volume

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;**Delete on termination:** Disable (EBS volume will not be deleted on termination of the EC2 instance)  

*Record AMI ID for future use<br/><br/><br/>


## 9. Download SRA files and extract in FASTQ format 

```
$ prefetch SRR11027625 SRR11027623 SRR11027622	
$ fasterq-dump SRR11027625 SRR11027623 SRR11027622
$ gzip *.fastq	
$ chmod 755 *
```

## 10. Remove technical sequences using cutadapt  

Individual sequence files:  

```
cutadapt -g ^CCTACGGGAGGCAGCAG -G ^GACTACHVGGGTATCTAATCC --discard-untrimmed -o SRR11027625_1.noprim.fastq.gz -p SRR11027625_2.noprim.fastq.gz SRR11027625_1.fastq.gz SRR11027625_2.fastq.gz
```

Multiple sequence files:

```
for f in ./*_1.fastq.gz
do
newname=$(basename $f _1.fastq.gz)
cutadapt -g ^CCTACGGGAGGCAGCAG -G ^GACTACHVGGGTATCTAATCC --discard-untrimmed -o ${newname}_1.noprim.fastq.gz -p ${newname}_2.noprim.fastq.gz ${newname}_1.fastq.gz ${newname}_2.fastq.gz
done
```

#### Move files to rstudio directory for processing in RStudio Server  

```
$ sudo chmod 777 /home/rstudio/
$ mv *.fastq.gz /home/rstudio
```

## 11. Access RStudio Server and run the DADA2 workflow  

Open a web browser and enter Public DNS(IPv4) followed by the RStudio port (8787) as the URL:

&lt;Public DNS(IPv4)&gt;:8787<br/><br/>

Follow the DADA2 Pipeline Tutorial (1.8): https://benjjneb.github.io/dada2/tutorial_1_8.html  

NB: path <- /home/rstudio<br/><br/>  

## 12. Transfer sequence and DADA2 ouput files from the EC2 instance to an S3 bucket 

#### List S3 buckets
```
$ aws s3 ls
```

#### Copy new and updated sequence files from EC2 instance to S3 bucket

```
$ aws s3 sync /home/rstudio/ s3://<aws-S3-bucket>/<subfolder>/
```

#### Verify files were copied
```
$ aws s3 ls s3://<aws-S3-bucket>/<subfolder>/
```

