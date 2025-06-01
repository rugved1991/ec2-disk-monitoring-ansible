# Deployment Guide

## Table of Contents
1. [Prerequisites](#prerequisites)
2. [Initial Setup](#initial-setup)
3. [Installation Steps](#installation-steps)
4. [Verification](#verification)
5. [Adding New Accounts](#adding-new-accounts)
6. [Troubleshooting](#troubleshooting)

## Prerequisites

### AWS Account Requirements
- Management account with administrative access
- Member accounts configured in AWS Organizations
- IAM permissions to create:
  - IAM roles and policies
  - S3 buckets
  - Lambda functions
  - CloudWatch dashboards

### System Requirements
```plaintext
- Python 3.8 or higher
- Ansible 2.9 or higher
- AWS CLI v2
- boto3 library
```

### Network Requirements
- VPC endpoints for:
  - Systems Manager
  - S3
  - CloudWatch
- Internet access for AWS API calls

## Initial Setup

### 1. AWS CLI Configuration
```bash
# Configure AWS CLI for management account
aws configure set region us-east-1
aws configure set output json

# Verify configuration
aws sts get-caller-identity
```

### 2. Install Required Packages
```bash
# Install Python dependencies
pip install -r requirements.txt

# Install Ansible collections
ansible-galaxy collection install -r requirements.yml
```

### 3. Environment Variables
```bash
# Set required environment variables
export AWS_DEFAULT_REGION=us-east-1
export ANSIBLE_HOST_KEY_CHECKING=False
```

## Installation Steps

### 1. Clone Repository
```bash
git clone https://github.com/rugved1991/ec2-disk-monitoring-ansible.git
cd disk-monitoring
```

### 2. Configure Inventory
Edit `src/inventory/inventory.ini`:
```ini
[management]
localhost ansible_connection=local

[member_accounts]
account1 aws_account_id=111111111111
account2 aws_account_id=222222222222
account3 aws_account_id=333333333333

[member_accounts:vars]
ansible_connection=aws_ssm
ansible_aws_ssm_region=us-east-1
```

### 3. Update Variables
Edit `src/group_vars/all.yml`:
```yaml
# AWS Configuration
aws_region: us-east-1
central_s3_bucket: "your-bucket-name"
metrics_namespace: "DiskUtilization"

# Alert Configuration
alert_threshold_warning: 70
alert_threshold_critical: 85
```

### 4. Deploy Management Account Resources
```bash
# Deploy base infrastructure
ansible-playbook -i src/inventory/inventory.ini src/site.yml --tags management

# Verify deployment
aws cloudformation list-stacks
```

### 5. Configure Member Accounts
```bash
# Deploy monitoring configuration to member accounts
ansible-playbook -i src/inventory/inventory.ini src/site.yml --tags member_accounts
```

## Verification

### 1. Check Infrastructure
```bash
# Verify S3 bucket creation
aws s3 ls s3://${CENTRAL_S3_BUCKET}

# Check Lambda function
aws lambda get-function --function-name DiskMetricsProcessor

# Verify CloudWatch dashboard
aws cloudwatch list-dashboards
```

### 2. Validate Metric Collection
```bash
# Check metric flow
ansible-playbook -i src/inventory/inventory.ini src/site.yml --tags verify

# View metrics in CloudWatch
aws cloudwatch get-metric-statistics \
    --namespace "DiskUtilization" \
    --metric-name "disk_used_percent" \
    --dimensions Name=InstanceId,Value=i-1234567890abcdef0 \
    --start-time $(date -d '1 hour ago' -u +%FT%TZ) \
    --end-time $(date -u +%FT%TZ) \
    --period 300 \
    --statistics Average
```

### 3. Test Alerts
```bash
# Simulate high disk usage
ansible-playbook -i src/inventory/inventory.ini tests/simulate_alerts.yml

# Verify alert generation
aws cloudwatch describe-alarms --state-value ALARM
```

## Adding New Accounts

### 1. Update Inventory
Add new account to `src/inventory/inventory.ini`:
```ini
[member_accounts]
account4 aws_account_id=444444444444
```

### 2. Deploy Configuration
```bash
# Deploy to new account
ansible-playbook -i src/inventory/inventory.ini src/site.yml \
    --limit account4 \
    --tags member_accounts
```

### 3. Verify Integration
```bash
# Check cross-account access
aws sts assume-role \
    --role-arn arn:aws:iam::444444444444:role/MonitoringRole \
    --role-session-name verification

# Verify metric collection
ansible-playbook -i src/inventory/inventory.ini src/site.yml \
    --limit account4 \
    --tags verify
```

## Troubleshooting

### Common Issues

#### 1. SSM Connection Failures
```bash
# Check SSM agent status
aws ssm describe-instance-information

# Verify role trust relationship
aws iam get-role --role-name MonitoringRole
```

#### 2. Metric Collection Issues
```bash
# Check CloudWatch agent status
aws ssm send-command \
    --targets Key=instanceids,Values=i-1234567890abcdef0 \
    --document-name AWS-RunShellScript \
    --parameters commands=['systemctl status amazon-cloudwatch-agent']

# View CloudWatch agent logs
aws logs get-log-events \
    --log-group-name /aws/cloudwatch-agent \
    --log-stream-name i-1234567890abcdef0
```

#### 3. Dashboard Issues
```bash
# Verify metric existence
aws cloudwatch list-metrics \
    --namespace "DiskUtilization"

# Check Lambda execution
aws logs get-log-events \
    --log-group-name /aws/lambda/DiskMetricsProcessor
```

### Recovery Procedures

#### 1. Reset Monitoring Configuration
```bash
ansible-playbook -i src/inventory/inventory.ini src/site.yml \
    --tags reset_monitoring
```

#### 2. Rebuild Dashboard
```bash
ansible-playbook -i src/inventory/inventory.ini src/site.yml \
    --tags rebuild_dashboard
```

#### 3. Force Metric Collection
```bash
ansible-playbook -i src/inventory/inventory.ini src/site.yml \
    --tags force_collection
```
