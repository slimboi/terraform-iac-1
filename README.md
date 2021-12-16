# AUTOMATE INFRASTRUCTURE WITH IAC USING TERRAFORM

This project is to create a basic vpc with 2 subnets on aws

## PREREQUISITE
___
- Create a user in your AWS account named **terraform** with only programmatic access to AWS and grant the user **AdministratorAccess** permission.

- Configure programmatic access from your workstation to connect to AWS using the access keys for **terraform user** and a Python SDK (boto3). You must have Python 3.6 or higher on your workstation.

- For easier authentication configuration – use A**WS CLI** with *aws configure* command.

* Create an S3 bucket to store Terraform state file. You can name it something like *yourname-dev-terraform-bucket*.
  ![terraform bucket](https://user-images.githubusercontent.com/7505497/146407133-4cbba08b-119f-46f3-a114-df3c13429df6.png)

* When you have configured authentication and installed boto3, make sure you can programmatically access your AWS account by running following commands in >python:

```
import boto3
s3 = boto3.resource('s3')
for bucket in s3.buckets.all():
    print(bucket.name)
```

## Create VPC
---
- In VScode create a new directory called terraform, and create a new file named **main.tf** inside the directory.
- Setup Terraform CLI following this [instruction](https://learn.hashicorp.com/tutorials/terraform/install-cli "terraform")
- Add AWS as a provider, and a resource to create a VPC in the **main.tf** file.
- The **provider** block informs Terraform that we intend to build infrastructure within AWS.
- The **resource** block will create a VPC.

```
provider "aws" {
  region = "eu-central-1"
}

# Create VPC
resource "aws_vpc" "main" {
  cidr_block                     = "172.16.0.0/16"
  enable_dns_support             = "true"
  enable_dns_hostnames           = "true"
  enable_classiclink             = "false"
  enable_classiclink_dns_support = "false"
}
```
- Now we download the required plugins for Terraform to work using the **terraform init** command. These plugins are used by providers and provisioners.

  ![init](https://user-images.githubusercontent.com/7505497/146406447-ae58b119-47c2-4969-9caf-6293dd25ed08.png)

- Next, run **terraform plan** to preview the changes that Terraform will make.
- Now apply the changes using **terraform apply**.

  ![vpc](https://user-images.githubusercontent.com/7505497/146406972-d2a2ef48-c65e-4168-b101-d12bf38ae30f.png)

## Create Subnet Resources
---
We require 2 subnets, so we need to add the following block of code to **main.tf** file
```
# Create public subnets1
    resource "aws_subnet" "public1" {
    vpc_id                     = aws_vpc.main.id
    cidr_block                 = "172.16.0.0/24"
    map_public_ip_on_launch    = true
    availability_zone          = "eu-central-1a"

}

# Create public subnet2
    resource "aws_subnet" "public2" {
    vpc_id                     = aws_vpc.main.id
    cidr_block                 = "172.16.1.0/24"
    map_public_ip_on_launch    = true
    availability_zone          = "eu-central-1b"
}
```
- Now, run **terraform plan** and **terraform apply** to apply the changes to the infrastructure.

### Observation
- Hardcoded values were used for cidr_block and availability_zone arguments. This is not recommended best practise.
- Also, multiple resource blocks were used to create the public subnets. We need to create a single resource block that can dynamically create resources without specifying multiple blocks.

## Refactoring the Code
---
```
variable "region" {
    default = "eu-central-1"
}

provider "aws" {
       region = var.region
   }
   
variable "vpc_cidr" {
       default = "172.16.0.0/16"
   }
   variable "enable_dns_support" {
       default = "true"
   }

   variable "enable_dns_hostnames" {
       default ="true" 
   }

   variable "enable_classiclink" {
       default = "false"
   }

   variable "enable_classiclink_dns_support" {
       default = "false"
   }
   
   
    # Create VPC
   resource "aws_vpc" "main" {
   cidr_block                     = var.vpc_cidr
   enable_dns_support             = var.enable_dns_support 
   enable_dns_hostnames           = var.enable_dns_support
   enable_classiclink             = var.enable_classiclink
   enable_classiclink_dns_support = var.enable_classiclink

   }
   ```

### **Fixing Multiple Resource Blocks**
- We fix the multiple resource blocks creating public subnets using Terraform's **Data Sources**. Data sources enable us to fetch information outside of Terraform, in this case, from AWS.

```
# Get list of available zones from a region
data "aws_available_zones" "available" {
   state = "available"      
}

# make use of the data source to create subnets
resource "aws_subnet" "public" {
   count = 2
   vpc_id = aws_vpc.main.id
   cidr_block = "172.16.1.0/24"
   map_public_ip_on_launch = true
   availability_zone = data.aws_Availability_zone.available.names[count.index]
}
```
### **Make the cidr_block Dynamic**.
- Use Terraform's cidrsubnet() function to dynamically create cidr_block for the subnets.
- The cidrsubnet(prefix, newbits, netnum) function takes three parameters.
  - Prefix must be given in CIDR notation
  - Newbits is the number of additional bits with which to extend the prefix. For example, if given a prefix ending in /16 and a newbits value of 4, the resulting subnet address will have length /20.
  - Netnum is a whole number that can be represented as a binary integer with no more than newbits binary digits, which will be used to populate the additional bits added to the prefix.
```
resource "aws_subnet" "public" {
   count = 2
   vpc_id = aws_vpc.main.id
   cidr_block = cidrsubnet(var.vpc_cidr, 4, count.index)
   map_public_ip_on_launch = true
   availability_zone       = data.aws_availability_zones.available.names[count.index]
}
```

### **Remove the hardcoded count value**
- We can remove hardcoded count value by calling the length() function on the aws_availability_zone data source.
- Also to ensure we only create 2 public subnets, we create a variable for the preferred_number_of_subnets to be created.
```
variable "preferred_number_of_public_subnets" {
  default = 2
}

resource "aws_subnet" "public" {
     count = var.preferred_number_of_public_subnets == null ? length(data.availability_zones_available.names): var.preferred_number_of_public_subnets
     vpc_id = aws_vpc.main.id
     cidr_block = cidrsubnet(var.vpc_cidr, 4, count.index)
     map_public_ip_on_launch = true
     availability_zone       = data.aws_availability_zones.available.names[count.index]
}
 ```

### **Introducing variable.tf and terraform.tfvars file**
- Instead of havng a long list of variables in main.tf file, we can actually make our code a lot more readable and better structured by moving out some parts of the configuration content to other files.
- Create a new file and name it variables.tf . This file is used to define the variable type and optionally set default value
- Create a terraform.tfvars file to set the actual values of the variables.

#### **variables.tf file**
```
 # create region variable
    variable "region" {
        default =  "eu-central-1"
    }
    variable "vpc_cidr" {
        default = "172.16.0.0/16"
    }
    variable "enable_dns_support" {
        default = "true"
    }

    variable "enable_dns_hostnames" {
        default ="true" 
    }

    variable "enable_classiclink" {
        default = "false"
    }

    variable "enable_classiclink_dns_support" {
        default = "false"
    }

    variable "preferred_number_of_public_subnets" {
        default = "2"
    }
```
#### **terraform.tfvars file**
```
region = "eu-central-1"

vpc_cidr = "172.16.0.0/16"

enable_classiclink = "false"

enable_classiclink_dns_support = "false"

enable_dns_hostnames = "true"

enable_dns_support = "true"

preferred_number_of_public_subnets = 2
```
- Now the terraform folder should have the files listed below 
![structure](https://user-images.githubusercontent.com/7505497/146411974-8ca3da60-069f-4f5a-a4e8-0112ccda6c50.png)

- When we run **terraform plan** and ** terraform apply** now, a vpc and 2 subnets will be created in our AWS account.