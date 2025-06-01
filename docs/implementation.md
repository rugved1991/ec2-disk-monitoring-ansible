# Implementation Guide

## Table of Contents
1. [Directory Structure](#directory-structure)
2. [Configuration Files](#configuration-files)
3. [Role Implementation](#role-implementation)
4. [Deployment Configuration](#deployment-configuration)
5. [Error Handling](#error-handling)

## Directory Structure

```plaintext
disk-monitoring/
├── src/
│   ├── inventory/
│   │   └── inventory.ini          # Account inventory
│   ├── group_vars/
│   │   └── all.yml               # Global variables
│   ├── roles/
│   │   └── disk-monitoring/
│   │       ├── tasks/
│   │       │   ├── main.yml      # Main tasks
│   │       │   └── collect.yml   # Collection tasks
│   │       └── templates/
│   │           ├── cloudwatch-config.json.j2
│   │           └── dashboard.json.j2
│   └── site.yml                  # Main playbook
└── requirements.yml              # Dependencies
```

## Configuration Files

### 1. Inventory Configuration
```ini
# src/inventory/inventory.ini

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

### 2. Global Variables
```yaml
# src/group_vars/all.yml

# AWS Configuration
aws_region: us-east-1
central_s3_bucket: "disk-metrics-central"
metrics_namespace: "DiskUtilization"

# Monitoring Configuration
collection_interval: 300  # 5 minutes
metric_retention_days: 90
alert_threshold_warning: 70
alert_threshold_critical: 85

# S3 Configuration
s3_bucket_name: "{{ central_s3_bucket }}"
s3_metrics_prefix: "metrics"

# Lambda Configuration
lambda_function_name: "DiskMetricsProcessor"
lambda_runtime: "python3.9"
lambda_memory: 128
lambda_timeout: 300
```

## Role Implementation

### 1. Main Tasks
```yaml
# src/roles/disk-monitoring/tasks/main.yml

---
- name: Import collection tasks
  import_tasks: collect.yml
  tags: [collect, metrics]

- name: Ensure CloudWatch agent is installed
  amazon.aws.aws_ssm_send_command:
    instance_ids: "{{ instance_id }}"
    document_name: AWS-ConfigureAWSPackage
    parameters:
      action: Install
      name: AmazonCloudWatchAgent
  register: install_result
  tags: [install]
  failed_when: 
    - install_result.rc is defined 
    - install_result.rc != 0
    - "'already installed' not in install_result.stderr"

- name: Configure CloudWatch agent
  amazon.aws.aws_ssm_send_command:
    instance_ids: "{{ instance_id }}"
    document_name: AmazonCloudWatch-ManageAgent
    parameters:
      action: configure
      mode: ec2
      optionalConfigurationSource: ssm
      optionalConfigurationLocation: "{{ cloudwatch_config_path }}"
  tags: [configure]
  register: config_result
  until: config_result is success
  retries: 3
  delay: 10
```

### 2. Collection Tasks
```yaml
# src/roles/disk-monitoring/tasks/collect.yml

---
- name: Collect disk utilization metrics
  block:
    - name: Execute disk space check
      amazon.aws.aws_ssm_send_command:
        instance_ids: "{{ instance_id }}"
        document_name: "AWS-RunShellScript"
        parameters:
          commands:
            - |
              df -h --output=source,size,used,avail,pcent | 
              grep '^/dev/' | 
              awk '{
                printf "{\\"device\\":\\"%s\\",\\"size\\":\\"%s\\",\\"used\\":\\"%s\\",\\"available\\":\\"%s\\",\\"used_percent\\":\\"%s\\"}\n",
                $1, $2, $3, $4, $5
              }'
      register: disk_metrics
  
    - name: Format metrics with metadata
      set_fact:
        formatted_metrics:
          timestamp: "{{ ansible_date_time.iso8601 }}"
          account_id: "{{ aws_account_id }}"
          instance_id: "{{ instance_id }}"
          metrics: "{{ disk_metrics.stdout_lines }}"

    - name: Upload metrics to S3
      amazon.aws.aws_s3:
        bucket: "{{ central_s3_bucket }}"
        object: "metrics/{{ aws_account_id }}/{{ instance_id }}/{{ ansible_date_time.date }}/{{ ansible_date_time.hour }}.json"
        content: "{{ formatted_metrics | to_json }}"
        mode: put
  rescue:
    - name: Log collection failure
      debug:
        msg: "Failed to collect metrics from {{ instance_id }}"
    
    - name: Send failure notification
      amazon.aws.sns_topic:
        name: "{{ sns_topic_name }}"
        message: "Metric collection failed for instance {{ instance_id }}"
      when: sns_topic_name is defined
```

## Deployment Configuration

### 1. Main Playbook
```yaml
# src/site.yml

---
- name: Setup Management Account Resources
  hosts: management
  gather_facts: false
  tasks:
    - name: Create central S3 bucket
      amazon.aws.s3_bucket:
        name: "{{ central_s3_bucket }}"
        region: "{{ aws_region }}"
        versioning: yes
        encryption: "AES256"
      register: s3_bucket

    - name: Setup Lambda function
      community.aws.lambda:
        name: "{{ lambda_function_name }}"
        state: present
        zip_file: files/lambda_function.zip
        runtime: "{{ lambda_runtime }}"
        handler: index.handler
        memory_size: "{{ lambda_memory }}"
        timeout: "{{ lambda_timeout }}"
        role: "{{ lambda_role_arn }}"
        environment_variables:
          METRICS_NAMESPACE: "{{ metrics_namespace }}"
          RETENTION_DAYS: "{{ metric_retention_days }}"

    - name: Create CloudWatch dashboard
      community.aws.cloudwatch_dashboard:
        dashboard_name: "DiskUtilization"
        dashboard_body: "{{ lookup('template', 'templates/dashboard.json.j2') }}"

- name: Configure Member Accounts
  hosts: member_accounts
  gather_facts: false
  roles:
    - disk-monitoring
```

## Error Handling

### 1. Retry Logic
```yaml
# Example retry configuration for failed tasks
- name: Collection task with retry
  block:
    - name: Attempt metric collection
      [task details]
  rescue:
    - name: Retry collection
      [task details]
      register: retry_result
      until: retry_result is success
      retries: 3
      delay: 5
```

### 2. Error Notifications
```yaml
# Error notification configuration
- name: Handle collection errors
  block:
    - name: Main task
      [task details]
  rescue:
    - name: Send error notification
      amazon.aws.sns_topic:
        name: "monitoring-alerts"
        message: "Error collecting metrics: {{ error_details }}"
```

## Dependencies

```yaml
# requirements.yml

collections:
  - name: amazon.aws
    version: ">=3.0.0"
  - name: community.aws
    version: ">=3.0.0"

roles:
  - src: https://github.com/ansible-collections/community.aws
    name: community.aws
```

## Usage

1. Install dependencies:
```bash
ansible-galaxy collection install -r requirements.yml
```

2. Configure AWS credentials:
```bash
export AWS_ACCESS_KEY_ID="your_access_key"
export AWS_SECRET_ACCESS_KEY="your_secret_key"
```

3. Run the playbook:
```bash
ansible-playbook -i src/inventory/inventory.ini src/site.yml
```

4. Verify deployment:
```bash
ansible-playbook -i src/inventory/inventory.ini src/site.yml --check
```

## Common Issues and Solutions

1. SSM Connection Issues:
   - Verify SSM Agent is installed
   - Check IAM roles
   - Validate VPC endpoints

2. Metric Collection Failures:
   - Check instance permissions
   - Verify disk mount points
   - Review CloudWatch agent status

3. Dashboard Updates:
   - Verify metric namespace
   - Check Lambda execution logs
   - Validate CloudWatch permissions
