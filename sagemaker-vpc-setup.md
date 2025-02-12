# SageMaker VPC and Network Setup Guide

   ```bash
   export AWS_REGION=eu-west-1
   ```

## 1. Create VPC

```bash
# Create VPC with DNS hostname and DNS resolution support
VPC_ID=$(aws ec2 create-vpc \
    --cidr-block 10.0.0.0/16 \
    --instance-tenancy default \
    --tag-specifications 'ResourceType=vpc,Tags=[{Key=Name,Value=sagemaker-vpc}]' \
    --query 'Vpc.VpcId' \
    --output text)

# Enable DNS hostname and DNS resolution support
aws ec2 modify-vpc-attribute \
    --vpc-id $VPC_ID \
    --enable-dns-hostnames '{"Value":true}'

aws ec2 modify-vpc-attribute \
    --vpc-id $VPC_ID \
    --enable-dns-support '{"Value":true}'

echo "Created VPC: $VPC_ID"
```

## Created VPC: vpc-0d721cad0c3776aee

## 2. Create Internet Gateway

```bash
# Create Internet Gateway
IGW_ID=$(aws ec2 create-internet-gateway \
    --tag-specifications 'ResourceType=internet-gateway,Tags=[{Key=Name,Value=sagemaker-igw}]' \
    --query 'InternetGateway.InternetGatewayId' \
    --output text)

# Attach Internet Gateway to VPC
aws ec2 attach-internet-gateway \
    --vpc-id $VPC_ID \
    --internet-gateway-id $IGW_ID

echo "Created and attached Internet Gateway: $IGW_ID"
```

## Created and attached Internet Gateway: igw-0186b20ba38b89f14

## 3. Create Subnets

```bash
# Create public subnet in availability zone 1
PUBLIC_SUBNET_1_ID=$(aws ec2 create-subnet \
    --vpc-id $VPC_ID \
    --cidr-block 10.0.1.0/24 \
    --availability-zone ${AWS_REGION}a \
    --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=sagemaker-public-1}]' \
    --query 'Subnet.SubnetId' \
    --output text)

# Create public subnet in availability zone 2
PUBLIC_SUBNET_2_ID=$(aws ec2 create-subnet \
    --vpc-id $VPC_ID \
    --cidr-block 10.0.2.0/24 \
    --availability-zone ${AWS_REGION}b \
    --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=sagemaker-public-2}]' \
    --query 'Subnet.SubnetId' \
    --output text)

# Enable auto-assign public IP for public subnets
aws ec2 modify-subnet-attribute \
    --subnet-id $PUBLIC_SUBNET_1_ID \
    --map-public-ip-on-launch

aws ec2 modify-subnet-attribute \
    --subnet-id $PUBLIC_SUBNET_2_ID \
    --map-public-ip-on-launch

echo "Created Public Subnet 1: $PUBLIC_SUBNET_1_ID"
echo "Created Public Subnet 2: $PUBLIC_SUBNET_2_ID"
```

## Created Public Subnet 1: subnet-02d618bb12996d695
## Created Public Subnet 2: subnet-0ebd8402edace0b0a

## 4. Create Route Table

```bash
# Create route table
ROUTE_TABLE_ID=$(aws ec2 create-route-table \
    --vpc-id $VPC_ID \
    --tag-specifications 'ResourceType=route-table,Tags=[{Key=Name,Value=sagemaker-rt}]' \
    --query 'RouteTable.RouteTableId' \
    --output text)

# Add route to Internet Gateway
aws ec2 create-route \
    --route-table-id $ROUTE_TABLE_ID \
    --destination-cidr-block 0.0.0.0/0 \
    --gateway-id $IGW_ID

# Associate public subnets with route table
aws ec2 associate-route-table \
    --subnet-id $PUBLIC_SUBNET_1_ID \
    --route-table-id $ROUTE_TABLE_ID

aws ec2 associate-route-table \
    --subnet-id $PUBLIC_SUBNET_2_ID \
    --route-table-id $ROUTE_TABLE_ID

echo "Created and configured Route Table: $ROUTE_TABLE_ID"
```

## Created and configured Route Table: rtb-0003466e7f1d2f979

## 5. Create Security Group

```bash
# Create security group
SECURITY_GROUP_ID=$(aws ec2 create-security-group \
    --group-name sagemaker-sg \
    --description "Security group for SageMaker" \
    --vpc-id $VPC_ID \
    --query 'GroupId' \
    --output text)

# Add inbound rules
aws ec2 authorize-security-group-ingress \
    --group-id $SECURITY_GROUP_ID \
    --protocol tcp \
    --port 443 \
    --cidr 0.0.0.0/0

# Add necessary tags
aws ec2 create-tags \
    --resources $SECURITY_GROUP_ID \
    --tags Key=Name,Value=sagemaker-sg

echo "Created Security Group: $SECURITY_GROUP_ID"
```

## Created Security Group: sg-0941cfa855712ce32

## 6. Create VPC Endpoints (Required for SageMaker)

```bash
# Create endpoints for SageMaker API and Runtime
aws ec2 create-vpc-endpoint \
    --vpc-id $VPC_ID \
    --vpc-endpoint-type Interface \
    --service-name com.amazonaws.${AWS_REGION}.sagemaker.api \
    --subnet-ids $PUBLIC_SUBNET_1_ID $PUBLIC_SUBNET_2_ID \
    --security-group-ids $SECURITY_GROUP_ID

aws ec2 create-vpc-endpoint \
    --vpc-id $VPC_ID \
    --vpc-endpoint-type Interface \
    --service-name com.amazonaws.${AWS_REGION}.sagemaker.runtime \
    --subnet-ids $PUBLIC_SUBNET_1_ID $PUBLIC_SUBNET_2_ID \
    --security-group-ids $SECURITY_GROUP_ID

# Create S3 endpoint (Gateway type)
aws ec2 create-vpc-endpoint \
    --vpc-id $VPC_ID \
    --vpc-endpoint-type Gateway \
    --service-name com.amazonaws.${AWS_REGION}.s3 \
    --route-table-ids $ROUTE_TABLE_ID
```

## 7. Create Execution Role

# First, create a trust policy JSON file

```bash
cat << EOF > trust-policy.json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "Service": "sagemaker.amazonaws.com"
            },
            "Action": "sts:AssumeRole"
        }
    ]
}
EOF
```

```bash
# Create the IAM role
aws iam create-role \
    --role-name SageMakerExecutionRole \
    --assume-role-policy-document file://trust-policy.json

# Attach the necessary managed policies
aws iam attach-role-policy \
    --role-name SageMakerExecutionRole \
    --policy-arn arn:aws:iam::aws:policy/AmazonSageMakerFullAccess

aws iam attach-role-policy \
    --role-name SageMakerExecutionRole \
    --policy-arn arn:aws:iam::aws:policy/AmazonS3FullAccess
    
ROLE_ARN=$(aws iam get-role \
    --role-name SageMakerExecutionRole \
    --query Role.Arn \
    --output text)    
    
echo "Created ROLE: $ROLE_ARN"    
```

## 7. Save Infrastructure IDs

```bash
# Save the IDs for later use
cat << EOF > sagemaker-infrastructure.txt
VPC_ID=$VPC_ID
PUBLIC_SUBNET_1_ID=$PUBLIC_SUBNET_1_ID
PUBLIC_SUBNET_2_ID=$PUBLIC_SUBNET_2_ID
SECURITY_GROUP_ID=$SECURITY_GROUP_ID
ROUTE_TABLE_ID=$ROUTE_TABLE_ID
$ROLE_ID= ####
EOF
```

## VPC_ID=vpc-0d721cad0c3776aee
## PUBLIC_SUBNET_1_ID=subnet-02d618bb12996d695
## PUBLIC_SUBNET_2_ID=subnet-0ebd8402edace0b0a
## SECURITY_GROUP_ID=sg-0941cfa855712ce32
## ROUTE_TABLE_ID=rtb-0003466e7f1d2f979
## AWS_ACCOUNT_ID= 992382645889

## 9. Now Create SageMaker Domain

```bash
# Set your AWS account ID

AWS_ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)

# Create the domain with specified instance type and app settings
aws sagemaker create-domain \
    --region ${AWS_REGION} \
    --domain-name my-sagemaker-domain \
    --vpc-id $VPC_ID \
    --subnet-ids $PUBLIC_SUBNET_1_ID $PUBLIC_SUBNET_2_ID \
    --auth-mode IAM \
    --default-user-settings "{
        \"ExecutionRole\": \"arn:aws:iam::${AWS_ACCOUNT_ID}:role/SageMakerExecutionRole\${ROLE_ID}",
        \"JupyterServerAppSettings\": {
            \"DefaultResourceSpec\": {
                \"InstanceType\": \"system\"
            }
        }
    }" \
    --query DomainArn \
    --output text
    
    
## arn:aws:sagemaker:eu-west-1:992382645889:domain/d-6kxgoifrsnsk

## Important Notes:

1. **Region Configuration:**
   - Before running these commands, ensure you have set your AWS region:
   ```bash
   export AWS_REGION=$(aws configure get region) ##eu-west-1
   ```

2. **Subnet Configuration:**
   - The example uses two availability zones for high availability
   - Adjust CIDR blocks if they conflict with existing networks

3. **Security:**
   - The security group allows HTTPS (443) from anywhere
   - Modify security group rules based on your security requirements

4. **Costs:**
   - VPC endpoints are not free
   - Consider cleaning up resources when not in use

5. **Prerequisites:**
   - AWS CLI installed and configured
   - Appropriate IAM permissions to create these resources

## Cleanup Commands

```bash
# If you need to delete everything:
aws sagemaker delete-domain --domain-id <domain-id>
aws ec2 delete-vpc-endpoints --vpc-endpoint-ids <endpoint-ids>
aws ec2 delete-security-group --group-id $SECURITY_GROUP_ID
aws ec2 delete-subnet --subnet-id $PUBLIC_SUBNET_1_ID
aws ec2 delete-subnet --subnet-id $PUBLIC_SUBNET_2_ID
aws ec2 delete-route-table --route-table-id $ROUTE_TABLE_ID
aws ec2 detach-internet-gateway --internet-gateway-id $IGW_ID --vpc-id $VPC_ID
aws ec2 delete-internet-gateway --internet-gateway-id $IGW_ID
aws ec2 delete-vpc --vpc-id $VPC_ID
```
