# Terraform Lab 11 - Cloud Computing

**Course:** Cloud Computing  
**Lab Title:** Infrastructure as Code with Terraform - Variables, Data Types, and AWS Deployment  
**Student:** Arshia Jadoon  
**Registration:** 2023-BSE-014  
**Semester:** VA

## Overview

This comprehensive lab introduces Infrastructure as Code (IaC) using Terraform to provision and manage AWS infrastructure. Students will learn Terraform fundamentals including variables, data types, state management, and real-world AWS resource deployment. The lab progresses from basic variable concepts to deploying a complete web infrastructure with VPC, subnets, security groups, and EC2 instances running Nginx.

## Table of Contents

### Task A: Terraform Fundamentals
1. [Provider & Variable Precedence](#task-1-provider--variable-precedence)
2. [Variable Validation & Sensitive Variables](#task-2-variable-validation--sensitive-variables)
3. [Project Variables, Locals & Outputs](#task-3-project-variables-locals--outputs)
4. [Maps and Objects](#task-4-maps-and-objects)
5. [Collections: Lists, Tuples & Sets](#task-5-collections-lists-tuples--sets)
6. [Null, Any Type & Dynamic Values](#task-6-null-any-type--dynamic-values)
7. [Git Ignore Configuration](#task-7-git-ignore-configuration)
8. [VPC Infrastructure Deployment](#task-8-vpc-infrastructure-deployment)
9. [EC2 Instance & Web Server](#task-9-ec2-instance--web-server)
10. [Resource Cleanup](#cleanup-destroy-resources)

---

## What is Terraform?

**Terraform** is an open-source Infrastructure as Code (IaC) tool created by HashiCorp that allows you to define, provision, and manage infrastructure using declarative configuration files.

### Key Features:
- **Declarative Syntax:** Define what you want, not how to build it
- **Provider Support:** AWS, Azure, GCP, and 1000+ providers
- **State Management:** Tracks real infrastructure state
- **Plan & Apply:** Preview changes before applying
- **Version Control:** Infrastructure configurations in Git
- **Reusability:** Modules for consistent deployments

### Why Terraform?

✅ **Consistency:** Same configuration = same infrastructure  
✅ **Automation:** No manual console clicking  
✅ **Version Control:** Track infrastructure changes over time  
✅ **Collaboration:** Teams work on same infrastructure code  
✅ **Multi-Cloud:** Single tool for multiple cloud providers  
✅ **Documentation:** Code serves as documentation  

---

## Task 1: Provider & Variable Precedence

**Objective:** Initialize Terraform, configure AWS provider, and understand variable precedence hierarchy

### Part A: Initialize Terraform Project

**Create main.tf:**
```bash
touch main.tf
```

**Configure AWS Provider:**
```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = "us-east-1"
}
```

**Initialize Terraform:**
```bash
terraform init
```

**What Happens:**
- Downloads AWS provider plugin
- Creates `.terraform/` directory
- Generates `.terraform.lock.hcl` lock file
- Prepares backend for state storage

### Part B: Variable Declaration & Usage

**Add Variable to main.tf:**
```hcl
variable "student_name" {
  description = "Name of the student"
  type        = string
  default     = "John Doe"
}

output "greeting" {
  value = "Hello, ${var.student_name}!"
}
```

**Apply Without Value (uses default):**
```bash
terraform apply
# Output: Hello, John Doe!
```

### Part C: Variable Precedence Testing

Terraform follows this precedence order (highest to lowest):

1. **Command-line flags** (`-var`)
2. **terraform.tfvars file**
3. **Environment variables** (`TF_VAR_`)
4. **Default values** in variable block

**Test 1: Prompt Input (no default)**
```bash
# Remove default value from variable
terraform apply
# Prompts: Enter a value for student_name:
```

**Test 2: Environment Variable**
```bash
export TF_VAR_student_name="Arshia"
terraform apply
# Output: Hello, Arshia!
```

**Test 3: terraform.tfvars File**
```bash
# Create terraform.tfvars
echo 'student_name = "Arshia Jadoon"' > terraform.tfvars
terraform apply
# Output: Hello, Arshia Jadoon!
```

**Test 4: Command-line Override (Highest Priority)**
```bash
terraform apply -var="student_name=CLI Override"
# Output: Hello, CLI Override!
```

**Clean Environment:**
```bash
unset TF_VAR_student_name
```

### Key Concepts:

**Terraform Workflow:**
```
terraform init     # Initialize & download providers
terraform plan     # Preview changes
terraform apply    # Create/update resources
terraform destroy  # Delete resources
```

**Variable Precedence Summary:**
```
-var flag  >  terraform.tfvars  >  TF_VAR_*  >  default value
(highest)                                      (lowest)
```

**File Types:**
- `.tf` - Terraform configuration files
- `.tfvars` - Variable value files
- `.tfstate` - State file (tracks infrastructure)
- `.terraform.lock.hcl` - Provider version lock

---

## Task 2: Variable Validation & Sensitive Variables

**Objective:** Implement input validation and handle sensitive data securely

### Part A: Variable Validation

**Add Subnet Variable with Validation:**
```hcl
variable "subnet_cidr" {
  description = "CIDR block for subnet"
  type        = string
  
  validation {
    condition     = can(cidrsubnet(var.subnet_cidr, 0, 0))
    error_message = "Must be a valid CIDR block (e.g., 10.0.1.0/24)"
  }
}

output "subnet_info" {
  value = "Subnet CIDR: ${var.subnet_cidr}"
}
```

**Test Invalid Input:**
```bash
terraform apply -var="subnet_cidr=invalid"
# Error: Must be a valid CIDR block
```

**Test Valid Input:**
```bash
terraform apply -var="subnet_cidr=10.0.1.0/24"
# Success!
```

### Part B: Sensitive Variables

**Add Sensitive API Token:**
```hcl
variable "api_token" {
  description = "API authentication token"
  type        = string
  sensitive   = true
  default     = "default-secret-token"
}

output "api_token_output" {
  value     = var.api_token
  sensitive = true
}
```

**Apply and Observe:**
```bash
terraform apply
# Output shows: (sensitive value) instead of actual token
```

**Check State File:**
```bash
cat terraform.tfstate | grep api_token
# WARNING: Token visible in state file!
```

### Part C: Ephemeral Variables (Terraform 1.10+)

**Ephemeral Variables:**
```hcl
variable "api_token" {
  description = "API authentication token"
  type        = string
  ephemeral   = true  # Not stored in state
}
```

**Key Difference:**
- **Sensitive:** Hidden in output but stored in state
- **Ephemeral:** Not stored in state at all (most secure)

### Security Best Practices:

✅ Mark secrets as `sensitive = true`  
✅ Use ephemeral variables when possible  
✅ Never commit `.tfstate` files to Git  
✅ Use remote state with encryption  
✅ Store secrets in AWS Secrets Manager/Parameter Store  
✅ Use environment variables for sensitive inputs  

---

## Task 3: Project Variables, Locals & Outputs

**Objective:** Organize project with structured variables, computed locals, and outputs

### Part A: Project-Level Variables

**Create variables.tf:**
```hcl
variable "project_name" {
  description = "Name of the project"
  type        = string
}

variable "environment" {
  description = "Environment (dev/staging/prod)"
  type        = string
  
  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "Environment must be dev, staging, or prod"
  }
}

variable "vpc_cidr" {
  description = "CIDR block for VPC"
  type        = string
}

variable "subnet_id" {
  description = "Subnet ID for resources"
  type        = string
}
```

**Populate terraform.tfvars:**
```hcl
project_name = "cloudlab"
environment  = "dev"
vpc_cidr     = "10.0.0.0/16"
subnet_id    = "subnet-06cf8046b6d80c766"
```

### Part B: Local Values

**Create locals.tf:**
```hcl
locals {
  # Computed naming convention
  resource_prefix = "${var.project_name}-${var.environment}"
  
  # Common tags
  common_tags = {
    Project     = var.project_name
    Environment = var.environment
    ManagedBy   = "Terraform"
    CreatedAt   = timestamp()
  }
  
  # Computed values
  vpc_name = "${local.resource_prefix}-vpc"
  
  # Conditional logic
  is_production = var.environment == "prod"
}
```

**Usage in main.tf:**
```hcl
output "resource_names" {
  value = {
    prefix      = local.resource_prefix
    vpc_name    = local.vpc_name
    is_prod     = local.is_production
  }
}
```

### Part C: Outputs

**Create outputs.tf:**
```hcl
output "project_info" {
  description = "Project configuration details"
  value = {
    name        = var.project_name
    environment = var.environment
    vpc_cidr    = var.vpc_cidr
    subnet      = var.subnet_id
  }
}

output "tags" {
  description = "Common tags applied to resources"
  value       = local.common_tags
}
```

### Key Concepts:

**Variables vs Locals:**

| Feature | Variables | Locals |
|---------|-----------|--------|
| Input from | User/tfvars | Computed in code |
| Can change | Yes | No (computed once) |
| Validation | Yes | No |
| Use case | User inputs | Derived values |
| Reference | `var.name` | `local.name` |

**Best Practices:**
- Use variables for inputs
- Use locals for computed/derived values
- Use outputs to expose important values
- Organize by file: variables.tf, locals.tf, outputs.tf

---

## Task 4: Maps and Objects

**Objective:** Work with complex data types for structured configuration

### Part A: Map Variables

**Add Tags Variable:**
```hcl
variable "resource_tags" {
  description = "Tags to apply to resources"
  type        = map(string)
  default = {
    Environment = "dev"
    Project     = "cloudlab"
    Owner       = "DevOps Team"
  }
}

output "tags_output" {
  value = {
    all_tags     = var.resource_tags
    environment  = var.resource_tags["Environment"]
    project      = var.resource_tags["Project"]
  }
}
```

**Apply and View:**
```bash
terraform apply
# Shows map structure and individual key access
```

### Part B: Object Variables

**Define Server Configuration:**
```hcl
variable "server_config" {
  description = "Server configuration object"
  type = object({
    instance_type = string
    ami_id        = string
    disk_size     = number
    monitoring    = bool
    tags          = map(string)
  })
  
  default = {
    instance_type = "t2.micro"
    ami_id        = "ami-0c55b159cbfafe1f0"
    disk_size     = 20
    monitoring    = true
    tags = {
      Name = "WebServer"
      Type = "Application"
    }
  }
}

output "server_details" {
  value = {
    type       = var.server_config.instance_type
    ami        = var.server_config.ami_id
    storage    = "${var.server_config.disk_size}GB"
    monitoring = var.server_config.monitoring ? "Enabled" : "Disabled"
    tags       = var.server_config.tags
  }
}
```

### Data Type Reference:

**Primitive Types:**
- `string` - Text values
- `number` - Numeric values (int or float)
- `bool` - true or false

**Complex Types:**
- `list(type)` - Ordered collection `["a", "b", "c"]`
- `map(type)` - Key-value pairs `{key = "value"}`
- `set(type)` - Unordered unique values
- `object({...})` - Structured data with typed attributes
- `tuple([...])` - Fixed-length collection with types

---

## Task 5: Collections: Lists, Tuples & Sets

**Objective:** Understand different collection types and their behaviors

### Part A: Define Collections

**Create variables.tf:**
```hcl
variable "availability_zones_list" {
  description = "List of AZs (ordered, duplicates allowed)"
  type        = list(string)
  default     = ["us-east-1a", "us-east-1b", "us-east-1a"]
}

variable "availability_zones_tuple" {
  description = "Tuple of AZs (fixed length, typed)"
  type        = tuple([string, string, string])
  default     = ["us-east-1a", "us-east-1b", "us-east-1c"]
}

variable "availability_zones_set" {
  description = "Set of AZs (unordered, unique only)"
  type        = set(string)
  default     = ["us-east-1a", "us-east-1b", "us-east-1a"]  
  # Automatically deduplicates to 2 items
}
```

### Part B: Compare Collections

**Create outputs:**
```hcl
output "collections_comparison" {
  value = {
    list_value   = var.availability_zones_list
    list_length  = length(var.availability_zones_list)  # 3
    
    tuple_value  = var.availability_zones_tuple
    tuple_length = length(var.availability_zones_tuple)  # 3
    
    set_value    = var.availability_zones_set
    set_length   = length(var.availability_zones_set)  # 2 (deduped)
  }
}
```

### Part C: Mutations with Locals

**Create locals.tf:**
```hcl
locals {
  # List operations
  list_sorted       = sort(var.availability_zones_list)
  list_distinct     = distinct(var.availability_zones_list)
  list_reversed     = reverse(var.availability_zones_list)
  
  # Set operations
  set_to_list       = tolist(var.availability_zones_set)
  
  # List transformations
  azs_uppercase     = [for az in var.availability_zones_list : upper(az)]
  azs_with_index    = [for i, az in var.availability_zones_list : "${i}: ${az}"]
  
  # Filtering
  azs_filtered      = [for az in var.availability_zones_list : az if az != "us-east-1a"]
}

output "mutations" {
  value = {
    original      = var.availability_zones_list
    sorted        = local.list_sorted
    distinct      = local.list_distinct
    reversed      = local.list_reversed
    uppercase     = local.azs_uppercase
    with_index    = local.azs_with_index
    filtered      = local.azs_filtered
  }
}
```

### Collection Type Comparison:

| Type | Ordered | Duplicates | Mutable Length | Use Case |
|------|---------|------------|----------------|----------|
| List | ✅ Yes | ✅ Allowed | ✅ Yes | General purpose collections |
| Tuple | ✅ Yes | ✅ Allowed | ❌ No (fixed) | Fixed structure data |
| Set | ❌ No | ❌ Unique only | ✅ Yes | Unique values only |

**Common Functions:**
- `length()` - Get collection size
- `sort()` - Sort list/set
- `distinct()` - Remove duplicates from list
- `reverse()` - Reverse list order
- `concat()` - Combine lists
- `contains()` - Check if value exists
- `element()` - Get item by index
- `tolist()`, `toset()` - Type conversion

---

## Task 6: Null, Any Type & Dynamic Values

**Objective:** Handle optional values, flexible types, and dynamic data

### Part A: Optional Variables with Null

**Define Optional Tag:**
```hcl
variable "optional_tag" {
  description = "Optional tag value (can be null)"
  type        = string
  default     = null
}
```

**Merge with Locals:**
```hcl
locals {
  base_tags = {
    Environment = "dev"
    ManagedBy   = "Terraform"
  }
  
  # Conditionally add optional tag
  all_tags = merge(
    local.base_tags,
    var.optional_tag != null ? { Optional = var.optional_tag } : {}
  )
}

output "tag_result" {
  value = local.all_tags
}
```

**Test Without Value:**
```bash
terraform apply
# Output: { Environment = "dev", ManagedBy = "Terraform" }
```

**Test With Value:**
```bash
terraform apply -var='optional_tag=Production'
# Output: { Environment = "dev", ManagedBy = "Terraform", Optional = "Production" }
```

### Part B: Any Type (Flexible Variables)

**Define Dynamic Variable:**
```hcl
variable "dynamic_value" {
  description = "Can accept any type"
  type        = any
}

output "dynamic_output" {
  value = {
    value = var.dynamic_value
    type  = type(var.dynamic_value)
  }
}
```

**Test Different Types:**

**String:**
```bash
terraform apply -var='dynamic_value=hello'
# type: string
```

**Number:**
```bash
terraform apply -var='dynamic_value=42'
# type: number
```

**List:**
```bash
terraform apply -var='dynamic_value=["a","b","c"]'
# type: list
```

**Map:**
```bash
terraform apply -var='dynamic_value={key="value"}'
# type: object
```

**Null:**
```bash
terraform apply -var='dynamic_value=null'
# type: null
```

### Use Cases:

**Null:**
- Optional configuration parameters
- Conditional resource creation
- Default behavior when value not provided

**Any Type:**
- Generic modules accepting flexible inputs
- Configuration that varies by environment
- Wrapper modules/abstractions

### Conditional Expressions:

```hcl
# Ternary operator
value = condition ? true_value : false_value

# Null coalescing
value = var.optional != null ? var.optional : "default"

# Shorter form
value = coalesce(var.optional, "default")
```

---

## Task 7: Git Ignore Configuration

**Objective:** Protect sensitive files from version control

### Create .gitignore

**Create .gitignore file:**
```bash
touch .gitignore
```

**Add Terraform-specific ignores:**
```gitignore
# Local .terraform directories
**/.terraform/*

# .tfstate files (contain sensitive data)
*.tfstate
*.tfstate.*

# Crash log files
crash.log
crash.*.log

# Exclude all .tfvars files (may contain sensitive variables)
*.tfvars
*.tfvars.json

# Ignore override files
override.tf
override.tf.json
*_override.tf
*_override.tf.json

# Ignore CLI configuration files
.terraformrc
terraform.rc

# Ignore key pairs
*.pem
*.key

# Ignore environment files
.env
.env.*

# IDE files
.vscode/
.idea/
*.swp
*.swo
*~

# OS files
.DS_Store
Thumbs.db
```

### Why Ignore These Files?

**Security Risks:**
- `.tfstate` - Contains all resource details including secrets
- `.tfvars` - May contain passwords, API keys
- `.pem` files - Private SSH keys
- `.env` files - Environment variables with secrets

**Best Practices:**
✅ Commit `.tf` configuration files  
✅ Commit `.tf` modules and reusable code  
✅ Commit `.gitignore` itself  
✅ Commit `README.md` documentation  
❌ Never commit state files  
❌ Never commit variable value files  
❌ Never commit private keys  
❌ Never commit credentials  

### Verify Gitignore:

```bash
# Check what git will track
git status

# Should NOT see:
# - .terraform/
# - terraform.tfstate
# - *.tfvars
# - *.pem
```

---

## Task 8: VPC Infrastructure Deployment

**Objective:** Deploy real AWS infrastructure - VPC, Subnets, Internet Gateway, and routing

### Part A: Clean Slate

**Remove Practice Files:**
```bash
rm main.tf variables.tf locals.tf outputs.tf terraform.tfvars
```

**Keep:**
- `.gitignore`
- `.terraform/` directory
- `.terraform.lock.hcl`

### Part B: Project Variables

**Create variables.tf:**
```hcl
variable "aws_region" {
  description = "AWS region for resources"
  type        = string
  default     = "us-east-1"
}

variable "project_name" {
  description = "Project name for resource naming"
  type        = string
}

variable "environment" {
  description = "Environment (dev/staging/prod)"
  type        = string
}

variable "vpc_cidr" {
  description = "CIDR block for VPC"
  type        = string
}

variable "public_subnet_cidr" {
  description = "CIDR block for public subnet"
  type        = string
}

variable "availability_zone" {
  description = "Availability zone for subnet"
  type        = string
}
```

**Create terraform.tfvars:**
```hcl
project_name        = "cloudlab"
environment         = "dev"
vpc_cidr            = "10.0.0.0/16"
public_subnet_cidr  = "10.0.1.0/24"
availability_zone   = "us-east-1a"
```

### Part C: VPC Resources

**Create vpc.tf:**
```hcl
# VPC
resource "aws_vpc" "main" {
  cidr_block           = var.vpc_cidr
  enable_dns_hostnames = true
  enable_dns_support   = true
  
  tags = {
    Name        = "${var.project_name}-${var.environment}-vpc"
    Environment = var.environment
  }
}

# Public Subnet
resource "aws_subnet" "public" {
  vpc_id                  = aws_vpc.main.id
  cidr_block              = var.public_subnet_cidr
  availability_zone       = var.availability_zone
  map_public_ip_on_launch = true
  
  tags = {
    Name        = "${var.project_name}-${var.environment}-public-subnet"
    Environment = var.environment
    Type        = "Public"
  }
}
```

**Apply VPC and Subnet:**
```bash
terraform plan
terraform apply
```

### Part D: Internet Gateway

**Add to vpc.tf:**
```hcl
# Internet Gateway
resource "aws_internet_gateway" "main" {
  vpc_id = aws_vpc.main.id
  
  tags = {
    Name        = "${var.project_name}-${var.environment}-igw"
    Environment = var.environment
  }
}
```

### Part E: Route Table

**Add to vpc.tf:**
```hcl
# Route Table for Public Subnet
resource "aws_route_table" "public" {
  vpc_id = aws_vpc.main.id
  
  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.main.id
  }
  
  tags = {
    Name        = "${var.project_name}-${var.environment}-public-rt"
    Environment = var.environment
  }
}

# Route Table Association
resource "aws_route_table_association" "public" {
  subnet_id      = aws_subnet.public.id
  route_table_id = aws_route_table.public.id
}
```

**Apply Networking:**
```bash
terraform apply
```

### Part F: Default Route Table

**Add to vpc.tf:**
```hcl
# Modify Default Route Table
resource "aws_default_route_table" "default" {
  default_route_table_id = aws_vpc.main.default_route_table_id
  
  # No routes (isolated)
  
  tags = {
    Name        = "${var.project_name}-${var.environment}-default-rt"
    Environment = var.environment
    Note        = "Default route table - no internet access"
  }
}
```

**Apply Final Network Configuration:**
```bash
terraform apply
```

### Infrastructure Architecture:

```
┌──────────────────────────────────────────────────────────┐
│                    VPC (10.0.0.0/16)                     │
│                                                          │
│  ┌─────────────────────────────────────────────────┐   │
│  │     Public Subnet (10.0.1.0/24)                 │   │
│  │     - Auto-assign public IP                      │   │
│  │     - Availability Zone: us-east-1a              │   │
│  └─────────────────┬───────────────────────────────┘   │
│                    │                                     │
│              ┌─────▼──────┐                            │
│              │ Route Table │                            │
│              │ 0.0.0.0/0   │                            │
│              │     ↓ IGW   │                            │
│              └─────┬──────┘                            │
│                    │                                     │
└────────────────────┼─────────────────────────────────────┘
                     │
              ┌──────▼───────┐
              │    Internet  │
              │    Gateway   │
              └──────┬───────┘
                     │
                 Internet
```

### Key Concepts:

**VPC (Virtual Private Cloud):**
- Isolated virtual network
- Define IP address range (CIDR)
- Logical isolation from other VPCs

**Subnets:**
- Subdivisions of VPC
- Mapped to specific AZ
- Public vs Private based on routing

**Internet Gateway:**
- Enables internet connectivity
- Attached to VPC
- Route target for public internet (0.0.0.0/0)

**Route Tables:**
- Control traffic routing
- Associated with subnets
- Define where traffic goes

**Resource References:**
```hcl
aws_vpc.main.id              # VPC ID
aws_subnet.public.id         # Subnet ID
aws_internet_gateway.main.id # IGW ID
```

---

## Task 9: EC2 Instance & Web Server

**Objective:** Launch EC2 instance with security group and deploy Nginx web server

### Part A: Get Your Public IP

**Add Variable:**
```hcl
variable "my_ip" {
  description = "Your public IP for SSH access"
  type        = string
}
```

**Find Your IP:**
```bash
curl https://checkip.amazonaws.com
# Returns: 203.101.190.205
```

**Add to terraform.tfvars:**
```hcl
my_ip = "203.101.190.205/32"
```

### Part B: Security Group

**Create security.tf:**
```hcl
resource "aws_security_group" "web_server" {
  name        = "${var.project_name}-${var.environment}-web-sg"
  description = "Security group for web server"
  vpc_id      = aws_vpc.main.id
  
  # SSH from your IP only
  ingress {
    description = "SSH from my IP"
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = [var.my_ip]
  }
  
  # HTTP from anywhere
  ingress {
    description = "HTTP from anywhere"
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
  
  # Allow all outbound
  egress {
    description = "Allow all outbound"
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
  
  tags = {
    Name        = "${var.project_name}-${var.environment}-web-sg"
    Environment = var.environment
  }
}
```

**Apply Security Group:**
```bash
terraform apply
```

### Part C: SSH Key Pair

**Generate Key Pair:**
```bash
# Using Terraform
resource "aws_key_pair" "deployer" {
  key_name   = "${var.project_name}-${var.environment}-key"
  public_key = file("~/.ssh/id_rsa.pub")  # Your existing public key
}

# OR generate new key locally
ssh-keygen -t rsa -b 4096 -f terraform-key -N ""
# Creates: terraform-key (private), terraform-key.pub (public)
```

**Create key.tf:**
```hcl
resource "aws_key_pair" "main" {
  key_name   = "${var.project_name}-${var.environment}-key"
  public_key = file("${path.module}/terraform-key.pub")
  
  tags = {
    Name        = "${var.project_name}-${var.environment}-key"
    Environment = var.environment
  }
}
```

### Part D: EC2 Instance Variables

**Add to variables.tf:**
```hcl
variable "instance_type" {
  description = "EC2 instance type"
  type        = string
  default     = "t2.micro"
}

variable "ami_id" {
  description = "AMI ID for EC2 instance"
  type        = string
  # Ubuntu 22.04 LTS in us-east-1
  default     = "ami-0c7217cdde317cfec"
}
```

### Part E: User Data Script (Nginx Installation)

**Create user-data.sh:**
```bash
#!/bin/bash
# Update system
apt-get update -y

# Install Nginx
apt-get install -y nginx

# Create custom page
cat > /var/www/html/index.html <<EOF
<!DOCTYPE html>
<html>
<head>
    <title>Terraform Deployed Server</title>
    <style>
        body {
            font-family: Arial, sans-serif;
            text-align: center;
            padding: 50px;
            background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
            color: white;
        }
        .container {
            background: rgba(255,255,255,0.1);
            padding: 40px;
            border-radius: 10px;
            backdrop-filter: blur(10px);
        }
