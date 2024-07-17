# Infrastructure as Code (IaC) for AWS EKS Cluster Provisioning using Terraform

This project demonstrates how to deploy and manage an AWS EKS cluster using Terraform, with S3 and DynamoDB for state management.

## Prerequisites

- AWS account
- AWS CLI installed and configured
- Terraform installed
- S3 bucket and DynamoDB table for Terraform state management

## Read the medium file for the better understanding of the project:
[IaC for AWS EKS sluster provising using Terraform](https://carl-writes.medium.com/infrastructure-as-code-iac-for-aws-eks-cluster-provisioning-using-terraform-95ce0d0a890f)


## Necessary Details of the Terraform Script

### Provider Configuration

Configures the AWS provider with the specified region.

```hcl
provider "aws" {
  region = "REPLACE-WITH-YOUR-AWS-REGION"
}
```

### VPC and Networking

- **VPC**: Creates a Virtual Private Cloud.
- **Subnets**: Creates two public subnets.
- **Internet Gateway**: Creates an internet gateway for the VPC.
- **Route Table**: Creates a route table for the public subnets and associates it with the subnets.

```hcl
resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"
}

resource "aws_subnet" "public1" {
  vpc_id     = aws_vpc.main.id
  cidr_block = "10.0.1.0/24"
  map_public_ip_on_launch = true
}

resource "aws_subnet" "public2" {
  vpc_id     = aws_vpc.main.id
  cidr_block = "10.0.2.0/24"
  map_public_ip_on_launch = true
}

resource "aws_internet_gateway" "main" {
  vpc_id = aws_vpc.main.id
}

resource "aws_route_table" "public" {
  vpc_id = aws_vpc.main.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.main.id
  }
}

resource "aws_route_table_association" "public1" {
  subnet_id      = aws_subnet.public1.id
  route_table_id = aws_route_table.public.id
}

resource "aws_route_table_association" "public2" {
  subnet_id      = aws_subnet.public2.id
  route_table_id = aws_route_table.public.id
}
```

### IAM Roles and Policies

- **EKS Cluster Role**: Creates IAM roles and attaches policies for the EKS cluster.
- **Worker Node Role**: Creates IAM roles and attaches policies for the worker nodes.

```hcl
resource "aws_iam_role" "eks_cluster" {
  name = "eks_cluster_role"
  assume_role_policy = data.aws_iam_policy_document.eks_assume_role.json
}

resource "aws_iam_role_policy_attachment" "eks_cluster_AmazonEKSClusterPolicy" {
  role       = aws_iam_role.eks_cluster.name
  policy_arn = "arn:aws:iam::aws:policy/AmazonEKSClusterPolicy"
}

resource "aws_iam_role" "eks_worker_node" {
  name = "eks_worker_node_role"
  assume_role_policy = data.aws_iam_policy_document.eks_assume_role.json
}

resource "aws_iam_role_policy_attachment" "eks_worker_node_AmazonEKSWorkerNodePolicy" {
  role       = aws_iam_role.eks_worker_node.name
  policy_arn = "arn:aws:iam::aws:policy/AmazonEKSWorkerNodePolicy"
}
```

### EKS Cluster and Node Group

- **EKS Cluster**: Creates an EKS cluster with the specified settings.
- **Worker Nodes**: Creates a node group with the specified instance types and scaling configuration.

```hcl
resource "aws_eks_cluster" "eks" {
  name     = "my-eks-cluster"
  role_arn = aws_iam_role.eks_cluster.arn

  vpc_config {
    subnet_ids = [aws_subnet.public1.id, aws_subnet.public2.id]
  }
}

resource "aws_eks_node_group" "eks_node_group" {
  cluster_name    = aws_eks_cluster.eks.name
  node_group_name = "eks-node-group"
  node_role_arn   = aws_iam_role.eks_worker_node.arn

  scaling_config {
    desired_size = 2
    max_size     = 3
    min_size     = 1
  }

  instance_types = ["t3.medium"]
  subnet_ids     = [aws_subnet.public1.id, aws_subnet.public2.id]
}
```

### Terraform Backend Configuration

Configures Terraform to use an S3 bucket and DynamoDB table for state management.

```hcl
terraform {
  backend "s3" {
    bucket         = "my-bucket"  # REPLACE WITH your-bucket-name
    key            = "eks-cluster/terraform.tfstate"
    region         = "YOUR-DESIRED-AWS-REGION"
    dynamodb_table = "terraform-lock"
    encrypt        = true
  }
}
```

### Instructions to Provision the EKS Cluster

1. **Initialize Terraform**:
    ```sh
    terraform init
    ```

2. **Validate the Configuration**:
    ```sh
    terraform validate
    ```

3. **Plan the Deployment**:
    ```sh
    terraform plan
    ```

4. **Apply the Configuration**:
    ```sh
    terraform apply
    ```

5. **Destroy the Infrastructure** (when no longer needed):
    ```sh
    terraform destroy
    ```


