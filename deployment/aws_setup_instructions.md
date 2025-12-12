# Infrastructure Setup for Question Bank API

This document provides step-by-step instructions for configuring AWS S3 and EC2/IAM roles for deploying the application

## Configuration Overview

- **EC2 Instance**: Your EC2 IP
- **SSH Key**: Your SSH Key
- **Application Path**: /var/www/question-bank-api
- **Service Name**: question-bank-api.service
- **S3 Bucket**: your-app-bucket
- **AWS Region**: your-region (e.g., us-east-1, ap-south-1)
- **Application User**: ubuntu

## Step 1: Create S3 Bucket

### Create Bucket
```bash
# Create the bucket in your chosen region
aws s3 mb s3://your-app-bucket --region your-region
```

### Enable Versioning
```bash
aws s3api put-bucket-versioning \
  --bucket your-app-bucket \
  --versioning-configuration Status=Enabled
```

### Enable Encryption
```bash
aws s3api put-bucket-encryption \
  --bucket your-app-bucket \
  --server-side-encryption-configuration '{
    "Rules": [
      {
        "ApplyServerSideEncryptionByDefault": {
          "SSEAlgorithm": "AES256"
        },
        "BucketKeyEnabled": true
      }
    ]
  }'
```

## Step 2: Create IAM Policy for S3 Access

Create `s3-upload-policy.json`:
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:PutObject",
        "s3:PutObjectAcl",
        "s3:GetObject",
        "s3:DeleteObject"
      ],
      "Resource": "arn:aws:s3:::your-app-bucket/*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "s3:ListBucket",
        "s3:GetBucketLocation"
      ],
      "Resource": "arn:aws:s3:::your-app-bucket"
    }
  ]
}
```

### Create the Policy
```bash
aws iam create-policy \
  --policy-name AppS3UploadPolicy \
  --policy-document file://s3-upload-policy.json \
  --description "Policy for application to upload result files to S3"
```

## Step 3: Create IAM Role for EC2

Create `ec2-trust-policy.json`:
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "ec2.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
```

### Create IAM Role
```bash
aws iam create-role \
  --role-name AppEC2Role \
  --assume-role-policy-document file://ec2-trust-policy.json \
  --description "IAM role for EC2 instances to access S3"
```

### Attach Policy to Role
```bash
# Get your AWS Account ID
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)

# Attach the S3 policy to the role
aws iam attach-role-policy \
  --role-name AppEC2Role \
  --policy-arn arn:aws:iam::${ACCOUNT_ID}:policy/AppS3UploadPolicy
```

## Step 4: Create Instance Profile

### Create Instance Profile
```bash
aws iam create-instance-profile \
  --instance-profile-name AppEC2InstanceProfile
```

### Add Role to Instance Profile
```bash
aws iam add-role-to-instance-profile \
  --instance-profile-name AppEC2InstanceProfile \
  --role-name AppEC2Role
```

## Step 5: Launch EC2 Instance

```bash
aws ec2 run-instances \
  --image-id ami-xxxxxxxxx \
  --count 1 \
  --instance-type t3.medium \
  --key-name your-ssh-key \
  --security-group-ids sg-xxxxxxxxx \
  --subnet-id subnet-xxxxxxxxx \
  --iam-instance-profile Name=AppEC2InstanceProfile
```

### Attach IAM Role to Existing Instance (if needed)
```bash
# Get instance ID
INSTANCE_ID=i-xxxxxxxxx

# Associate IAM instance profile
aws ec2 associate-iam-instance-profile \
  --instance-id $INSTANCE_ID \
  --iam-instance-profile Name=AppEC2InstanceProfile
```

## Step 6: Configure Environment Variables

### Option 1: Environment File
```bash
# Add to .env file or environment configuration
export S3_BUCKET_NAME=your-app-bucket
export AWS_DEFAULT_REGION=your-region
```

### Option 2: Docker Compose
```yaml
# docker-compose.yml or Kubernetes deployment
environment:
  - S3_BUCKET_NAME=your-app-bucket
  - AWS_DEFAULT_REGION=your-region
```

### Option 3: Systemd Service
```ini
# In service file
[Service]
Environment=S3_BUCKET_NAME=your-app-bucket
Environment=AWS_DEFAULT_REGION=your-region
```

## Step 7: Configure Bucket Policy (Optional)

Create `bucket-policy.json`:
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::_ACCOUNT_ID:role/AppEC2Role"
      },
      "Action": [
        "s3:PutObject",
        "s3:GetObject",
        "s3:DeleteObject"
      ],
      "Resource": "arn:aws:s3:::your-app-bucket/*"
    },
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::_ACCOUNT_ID:role/AppEC2Role"
      },
      "Action": [
        "s3:ListBucket",
        "s3:GetBucketLocation"
      ],
      "Resource": "arn:aws:s3:::your-app-bucket"
    }
  ]
}
```

### Apply Bucket Policy
```bash
aws s3api put-bucket-policy \
  --bucket your-app-bucket \
  --policy file://bucket-policy.json
```

## Testing

### Test S3 Access
```bash
# SSH to EC2 instance and test S3 access
aws s3 ls s3://your-app-bucket/

# Test upload
echo "test file" > test.txt
aws s3 cp test.txt s3://your-app-bucket/BulkUploadResponse/test/
```

## Security Best Practices

### Enable CloudTrail (Optional)
```bash
aws cloudtrail create-trail \
  --name app-s3-trail \
  --s3-bucket-name your-cloudtrail-bucket \
  --include-global-service-events \
  --is-multi-region-trail
```

### Enable Access Logging (Optional)
```bash
aws s3api put-bucket-logging \
  --bucket your-app-bucket \
  --bucket-logging-status '{
    "LoggingEnabled": {
      "TargetBucket": "your-access-logs-bucket",
      "TargetPrefix": "app-s3-access-logs/"
    }
  }'
```

### Lifecycle Policy (Optional)
```bash
# Create lifecycle policy to automatically delete old files
aws s3api put-bucket-lifecycle-configuration \
  --bucket your-app-bucket \
  --lifecycle-configuration '{
    "Rules": [
      {
        "ID": "DeleteOldFiles",
        "Status": "Enabled",
        "Filter": {
          "Prefix": "BulkUploadResponse/"
        },
        "Expiration": {
          "Days": 30
        }
      }
    ]
  }'
```

## Verification Commands

```bash
# Check IAM role
aws iam get-role --role-name AppEC2Role

# Check role policies
aws iam list-attached-role-policies --role-name AppEC2Role

# Test S3 access from application server
curl -s http://169.254.169.254/latest/meta-data/iam/security-credentials/AppEC2Role
```

## Deployment Commands

```bash
# SSH to instance
ssh -i "your-ssh-key.pem" ubuntu@your-ec2-ip

# Navigate to application directory
cd /var/www/question-bank-api

# Activate virtual environment
source venv/bin/activate

# Restart the service
sudo systemctl restart question-bank-api.service
```

### Update Service Configuration
```bash
# Add to systemd service file
sudo nano /etc/systemd/system/question-bank-api.service

# Add these lines under [Service]:
Environment=S3_BUCKET_NAME=your-app-bucket
Environment=AWS_DEFAULT_REGION=your-region

# Reload and restart
sudo systemctl daemon-reload
sudo systemctl restart question-bank-api.service
```

### Service Status
```bash
# Check service status
sudo systemctl status question-bank-api.service

# Test S3 access (if IAM role is configured)
aws s3 ls s3://your-app-bucket/
```

## Replace Placeholders

Before using this guide, replace the following placeholders with your actual values:

- `your-app-bucket` → Your S3 bucket name
- `your-region` → Your AWS region (e.g., us-east-1, ap-south-1)
- `your-ssh-key` → Your SSH key name
- `your-ec2-ip` → Your EC2 instance IP address
- `_ACCOUNT_ID` → Your AWS Account ID
- `ami-xxxxxxxxx` → Your preferred AMI ID
- `sg-xxxxxxxxx` → Your security group ID
- `subnet-xxxxxxxxx` → Your subnet ID
- `i-xxxxxxxxx` → Your EC2 instance ID