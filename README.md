# NT548 Lab1 - AWS Infrastructure với Terraform và CloudFormation

Triển khai hạ tầng AWS gồm VPC, Subnets, NAT Gateway, Security Groups và EC2 instances bằng cả Terraform và CloudFormation.

## Kiến trúc tổng quan

```
Internet
    │
    ▼
Internet Gateway
    │
    ▼
┌─────────────────────────────────────────┐
│  VPC (10.0.0.0/16)                      │
│                                         │
│  ┌───────────────────┐                  │
│  │  Public Subnet    │  ◄── Public RT   │
│  │  10.0.1.0/24      │      (→ IGW)     │
│  │                   │                  │
│  │  [Public EC2]     │                  │
│  │  [NAT Gateway]    │                  │
│  └───────────────────┘                  │
│           │ NAT                         │
│           ▼                             │
│  ┌───────────────────┐                  │
│  │  Private Subnet   │  ◄── Private RT  │
│  │  10.0.2.0/24      │      (→ NAT GW)  │
│  │                   │                  │
│  │  [Private EC2]    │                  │
│  └───────────────────┘                  │
└─────────────────────────────────────────┘
```

## Yêu cầu hệ thống

- [Terraform](https://developer.hashicorp.com/terraform/downloads) >= 1.0
- [AWS CLI](https://aws.amazon.com/cli/) đã cấu hình (`aws configure`)
- AWS account với IAM permissions phù hợp (xem bên dưới)

## IAM Permissions cần thiết

| AWS Managed Policy | Dùng cho | Bắt buộc |
|-------------------|---------|----------|
| `AmazonEC2FullAccess` | EC2 instances, Key Pairs, Security Groups, AMI | ✅ Cả hai |
| `AmazonVPCFullAccess` | VPC, Subnets, IGW, NAT Gateway, Route Tables, EIP | ✅ Cả hai |
| `AWSCloudFormationFullAccess` | Tạo/xóa CloudFormation stacks | ✅ Chỉ CloudFormation |

> **Terraform** chỉ cần `AmazonEC2FullAccess` + `AmazonVPCFullAccess` vì gọi thẳng AWS API.
> **CloudFormation** cần thêm `AWSCloudFormationFullAccess` để tạo và quản lý stacks.

### Cách thêm policy trong IAM Console

1. Vào **IAM Console** → **Users** → chọn user của bạn
2. Tab **Permissions** → **Add permissions** → **Attach policies directly**
3. Tìm và chọn từng policy trong bảng trên → **Next** → **Add permissions**

---

## Phần 1: Terraform

### Cấu trúc thư mục

```
Terraform/
├── main.tf                    # Root module - gọi các module con
├── variables.tf               # Khai báo biến
├── outputs.tf                 # Kết quả đầu ra
├── terraform.tfvars           # Giá trị biến (có thể chỉnh sửa)
└── modules/
    ├── vpc/                   # Module VPC, Subnets, IGW, NAT, Route Tables
    │   ├── main.tf
    │   ├── variables.tf
    │   └── outputs.tf
    ├── security_groups/       # Module Security Groups
    │   ├── main.tf
    │   ├── variables.tf
    │   └── outputs.tf
    └── ec2/                   # Module EC2 Instances + Key Pair
        ├── main.tf
        ├── variables.tf
        └── outputs.tf
```

### Các biến có thể cấu hình (`terraform.tfvars`)

| Biến | Mặc định | Mô tả |
|------|----------|-------|
| `region` | `ap-southeast-1` | AWS Region |
| `project_name` | `NT548-Lab` | Tên project |
| `vpc_cidr` | `10.0.0.0/16` | CIDR block của VPC |
| `public_subnet_cidr` | `10.0.1.0/24` | CIDR của Public Subnet |
| `private_subnet_cidr` | `10.0.2.0/24` | CIDR của Private Subnet |
| `availability_zone` | `ap-southeast-1a` | Availability Zone |
| `ssh_allowed_cidr` | `0.0.0.0/0` | CIDR được phép SSH vào Public EC2 |
| `ami` | _(tự động)_ | Để trống để tự resolve Ubuntu Server 24.04 LTS mới nhất |
| `instance_type` | `t3.micro` | Loại EC2 instance |


### Các bước chạy Terraform

```bash
# 1. Di chuyển vào thư mục Terraform
cd Terraform

# 2. Khởi tạo Terraform (tải providers và modules)
terraform init

# 3. Xem kế hoạch triển khai
terraform plan

# 4. Triển khai hạ tầng
terraform apply

# Gõ 'yes' khi được hỏi xác nhận
```

### Kết quả sau khi apply

Terraform sẽ in ra:
- `vpc_id` - ID của VPC
- `public_ec2_ip` - Public IP của EC2 trong Public Subnet
- `private_ec2_private_ip` - Private IP của EC2 trong Private Subnet
- `ssh_to_public` - Lệnh SSH vào Public EC2
- `ssh_to_private_via_proxy` - Lệnh SSH vào Private EC2 qua Public EC2 (proxy)

File `NT548-Lab1-key.pem` sẽ được tạo tự động trong thư mục `Terraform/`.

### Kết nối SSH

```powershell
# SSH vào Public EC2
ssh -i NT548-Lab-key.pem ubuntu@<public_ec2_ip>

# SSH vào Private EC2 qua Public EC2 (dùng ProxyCommand thay -J để forward key cho cả 2 hop)
ssh -i NT548-Lab-key.pem -o "ProxyCommand=ssh -i NT548-Lab-key.pem -W %h:%p ubuntu@<public_ec2_ip>" ubuntu@<private_ec2_private_ip>
```

### Dọn dẹp tài nguyên 

```bash
terraform destroy
# Gõ 'yes' khi được hỏi xác nhận
```

---

## Phần 2: CloudFormation

### Cấu trúc thư mục

```
CloudFormation/
├── 01-vpc.yaml              # VPC, Subnets, IGW, NAT Gateway, Route Tables
├── 02-security-groups.yaml  # Security Groups
└── 03-ec2.yaml              # EC2 Instances
```

> **3 stack riêng biệt** kết nối với nhau qua `Outputs` / `!ImportValue`. Stack sau import giá trị từ stack trước (VPC ID, Subnet ID, Security Group ID). Đây là phương pháp tiêu chuẩn để tổ chức CloudFormation theo dạng module.

### Yêu cầu trước khi chạy

Tạo EC2 Key Pair trên AWS Console hoặc CLI trước:

**Linux/macOS:**
```bash
aws ec2 create-key-pair \
  --key-name NT548-Lab1-Cloudformation-key \
  --region ap-southeast-1 \
  --query 'KeyMaterial' \
  --output text > NT548-Lab1-Cloudformation-key.pem

chmod 400 NT548-Lab1-Cloudformation-key.pem
```

**Windows (PowerShell):**
```powershell
aws ec2 create-key-pair `
  --key-name NT548-Lab1-Cloudformation-key `
  --region ap-southeast-1 `
  --query 'KeyMaterial' `
  --output text | Out-File -Encoding ascii NT548-Lab1-Cloudformation-key.pem

icacls NT548-Lab1-Cloudformation-key.pem /inheritance:r /grant:r "${env:USERNAME}:R"
```

### Các bước triển khai CloudFormation

**Bước 1: Deploy VPC stack**

```powershell
aws cloudformation deploy `
  --template-file CloudFormation/01-vpc.yaml `
  --stack-name NT548-Lab1-Cloudformation-vpc `
  --region ap-southeast-1 `
  --parameter-overrides `
    ProjectName=NT548-Lab1-Cloudformation `
    VpcCidr=10.0.0.0/16 `
    PublicSubnetCidr=10.0.1.0/24 `
    PrivateSubnetCidr=10.0.2.0/24 `
    AvailabilityZone=ap-southeast-1a
```

**Bước 2: Deploy Security Groups stack** (sau khi bước 1 hoàn thành)

```powershell
aws cloudformation deploy `
  --template-file CloudFormation/02-security-groups.yaml `
  --stack-name NT548-Lab1-Cloudformation-security-groups `
  --region ap-southeast-1 `
  --parameter-overrides `
    ProjectName=NT548-Lab1-Cloudformation `
    SshAllowedCidr=0.0.0.0/0
```

**Bước 3: Deploy EC2 stack** (sau khi bước 2 hoàn thành)

```powershell
aws cloudformation deploy `
  --template-file CloudFormation/03-ec2.yaml `
  --stack-name NT548-Lab1-Cloudformation-ec2 `
  --region ap-southeast-1 `
  --parameter-overrides `
    ProjectName=NT548-Lab1-Cloudformation `
    KeyPairName=NT548-Lab1-Cloudformation-key `
    InstanceType=t3.micro
```

### Xem kết quả

```powershell
# Xem IP của Public EC2
aws cloudformation describe-stacks `
  --stack-name NT548-Lab1-Cloudformation-ec2 `
  --region ap-southeast-1 `
  --query 'Stacks[0].Outputs'
```

### Kết nối SSH (CloudFormation)

```powershell
# SSH vào Public EC2
ssh -i NT548-Lab1-Cloudformation-key.pem ubuntu@<PublicEC2PublicIp>

# SSH vào Private EC2 qua Public EC2 (dùng ProxyCommand thay -J để forward key cho cả 2 hop)
ssh -i NT548-Lab1-Cloudformation-key.pem -o "ProxyCommand=ssh -i NT548-Lab1-Cloudformation-key.pem -W %h:%p ubuntu@<PublicEC2PublicIp>" ubuntu@<PrivateEC2PrivateIp>
```

### Dọn dẹp tài nguyên CloudFormation (theo thứ tự ngược)

```powershell
# Bước 1: Xóa EC2 stack trước
aws cloudformation delete-stack --stack-name NT548-Lab1-Cloudformation-ec2 --region ap-southeast-1

# Kiểm tra trạng thái (chờ đến khi không còn stack hoặc DELETE_COMPLETE)
aws cloudformation describe-stacks --stack-name NT548-Lab1-Cloudformation-ec2 --region ap-southeast-1 --query 'Stacks[0].StackStatus'

# Bước 2: Xóa Security Groups stack
aws cloudformation delete-stack --stack-name NT548-Lab1-Cloudformation-security-groups --region ap-southeast-1

# Bước 3: Xóa VPC stack
aws cloudformation delete-stack --stack-name NT548-Lab1-Cloudformation-vpc --region ap-southeast-1
```
