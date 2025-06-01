# Maintenance Guide

## Table of Contents
1. [Routine Maintenance](#routine-maintenance)
2. [Monitoring and Alerts](#monitoring-and-alerts)
3. [Backup and Recovery](#backup-and-recovery)
4. [Updates and Upgrades](#updates-and-upgrades)
5. [Troubleshooting Guide](#troubleshooting-guide)

## Routine Maintenance

### Daily Operations

#### 1. Health Check Procedures
```bash
# Check metric collection status
aws cloudwatch list-metrics \
    --namespace "DiskUtilization" \
    --start-time $(date -d '24 hours ago' -u +%FT%TZ)

# Verify Lambda function health
aws lambda get-function-statistics \
    --function-name DiskMetricsProcessor

# Review CloudWatch agent status
ansible-playbook maintenance/health_check.yml --tags daily
```

#### 2. Alert Review
```bash
# Check active alerts
aws cloudwatch describe-alarms --state-value ALARM

# Review recent alert history
aws cloudwatch describe-alarm-history \
    --start-date $(date -d '24 hours ago' -u +%FT%TZ)
```

#### 3. Error Log Analysis
```bash
# Check Lambda errors
aws logs filter-log-events \
    --log-group-name /aws/lambda/DiskMetricsProcessor \
    --filter-pattern "ERROR"

# Review SSM command failures
aws ssm list-command-invocations \
    --filters Key=Status,Values=Failed
```

### Weekly Tasks

#### 1. Performance Review
```bash
# Generate performance report
ansible-playbook maintenance/performance_report.yml

# Review metric collection statistics
aws cloudwatch get-metric-statistics \
    --namespace AWS/Lambda \
    --metric-name Duration \
    --dimensions Name=FunctionName,Value=DiskMetricsProcessor \
    --start-time $(date -d '7 days ago' -u +%FT%TZ) \
    --period 86400 \
    --statistics Average
```

#### 2. Resource Cleanup
```bash
# Clean old metric data
aws s3 ls s3://${CENTRAL_S3_BUCKET}/metrics/ --recursive \
    | awk '$1 < "$(date -d '30 days ago' +%Y-%m-%d)"' \
    | xargs -I {} aws s3 rm s3://${CENTRAL_S3_BUCKET}/{}

# Archive old logs
ansible-playbook maintenance/archive_logs.yml
```

#### 3. Permission Audit
```bash
# Review IAM roles
aws iam list-role-policies --role-name MonitoringRole

# Check cross-account access
ansible-playbook maintenance/verify_permissions.yml
```

### Monthly Tasks

#### 1. Configuration Review
```bash
# Validate monitoring configuration
ansible-playbook maintenance/validate_config.yml

# Update thresholds if needed
ansible-playbook maintenance/update_thresholds.yml \
    --extra-vars "warning_threshold=75 critical_threshold=90"
```

#### 2. Compliance Check
```bash
# Generate compliance report
ansible-playbook maintenance/compliance_check.yml

# Review security configurations
ansible-playbook maintenance/security_audit.yml
```

#### 3. Documentation Update
- Review and update operational procedures
- Update runbooks with any new issues/solutions
- Validate contact information and escalation procedures

## Monitoring and Alerts

### Alert Configuration

#### 1. Disk Space Alerts
```yaml
# Alert thresholds configuration
disk_alerts:
  warning:
    threshold: 70
    evaluation_periods: 3
  critical:
    threshold: 85
    evaluation_periods: 1
```

#### 2. Performance Alerts
```yaml
# Performance monitoring thresholds
performance_alerts:
  collection_delay:
    threshold: 600  # seconds
    evaluation_periods: 2
  lambda_errors:
    threshold: 5
    evaluation_periods: 1
```

### Alert Response Procedures

#### 1. High Disk Usage Alert
```bash
# Investigate disk usage
ansible-playbook maintenance/disk_investigation.yml \
    --extra-vars "instance_id=${INSTANCE_ID}"

# Clean up if necessary
ansible-playbook maintenance/disk_cleanup.yml \
    --extra-vars "instance_id=${INSTANCE_ID}"
```

#### 2. Collection Failure Alert
```bash
# Check collection status
ansible-playbook maintenance/verify_collection.yml \
    --extra-vars "account_id=${ACCOUNT_ID}"

# Reset collection if needed
ansible-playbook maintenance/reset_collection.yml \
    --extra-vars "account_id=${ACCOUNT_ID}"
```

## Backup and Recovery

### Backup Procedures

#### 1. Configuration Backup
```bash
# Backup current configuration
ansible-playbook maintenance/backup_config.yml

# Archive old backups
ansible-playbook maintenance/archive_backups.yml
```

#### 2. Metric Data Backup
```bash
# Backup metric data to secondary bucket
aws s3 sync \
    s3://${PRIMARY_BUCKET}/metrics/ \
    s3://${BACKUP_BUCKET}/metrics/ \
    --only-show-errors
```

### Recovery Procedures

#### 1. Configuration Recovery
```bash
# Restore configuration from backup
ansible-playbook maintenance/restore_config.yml \
    --extra-vars "backup_date=YYYY-MM-DD"

# Verify restoration
ansible-playbook maintenance/verify_config.yml
```

#### 2. Service Recovery
```bash
# Reset monitoring services
ansible-playbook maintenance/reset_services.yml

# Rebuild dashboards
ansible-playbook maintenance/rebuild_dashboards.yml
```

## Updates and Upgrades

### Component Updates

#### 1. CloudWatch Agent Update
```bash
# Update CloudWatch agent
ansible-playbook maintenance/update_cloudwatch_agent.yml

# Verify agent version
ansible-playbook maintenance/verify_agent_version.yml
```

#### 2. Lambda Function Update
```bash
# Deploy new Lambda code
ansible-playbook maintenance/update_lambda.yml \
    --extra-vars "version=1.2.3"

# Verify deployment
ansible-playbook maintenance/verify_lambda.yml
```

### Version Control

#### 1. Configuration Version Control
```bash
# Store current configuration version
ansible-playbook maintenance/version_config.yml

# Roll back if needed
ansible-playbook maintenance/rollback_config.yml \
    --extra-vars "version=1.1.0"
```

## Troubleshooting Guide

### Common Issues

#### 1. Metric Collection Failures
```bash
# Check SSM agent status
aws ssm describe-instance-information

# Verify CloudWatch agent
aws ssm send-command \
    --document-name AWS-RunShellScript \
    --targets Key=instanceids,Values=${INSTANCE_ID} \
    --parameters commands=['systemctl status amazon-cloudwatch-agent']
```

#### 2. Dashboard Issues
```bash
# Verify metric availability
aws cloudwatch list-metrics \
    --namespace "DiskUtilization"

# Check dashboard configuration
aws cloudwatch get-dashboard \
    --dashboard-name "DiskUtilization"
```

### Resolution Steps

#### 1. Reset Collection
```bash
# Reset metric collection
ansible-playbook maintenance/reset_collection.yml

# Verify data flow
ansible-playbook maintenance/verify_metrics.yml
```

#### 2. Rebuild Integration
```bash
# Rebuild monitoring integration
ansible-playbook maintenance/rebuild_integration.yml

# Validate setup
ansible-playbook maintenance/validate_setup.yml
```
