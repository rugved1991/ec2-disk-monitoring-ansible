# Architecture Overview

## Table of Contents
1. [System Architecture](#system-architecture)
2. [Component Details](#component-details)
3. [Data Flow](#data-flow)
4. [Security Architecture](#security-architecture)
5. [Scaling Design](#scaling-design)

## System Architecture

![AWS Multi-Account Monitoring Architecture](../images/architecture.png)

The solution implements a hub-and-spoke architecture where the Management Account acts as the central hub for monitoring multiple AWS accounts. This design enables centralized control while maintaining security and scalability.

## Component Details

### Management Account Components

#### 1. IAM Identity Center (formerly AWS SSO)
- **Purpose**: Centralized access management and authentication
- **Key Features**:
  - Single sign-on capabilities
  - Role-based access control
  - Cross-account permission management
- **Integration Points**:
  - AWS Organizations integration
  - Cross-account role assumption
  - User and group management

#### 2. Ansible Control Node
- **Purpose**: Automation and orchestration hub
- **Configuration**:
  - Deployed as EC2 instance
  - IAM role with required permissions
  - Systems Manager integration
- **Functions**:
  - Executes monitoring playbooks
  - Manages cross-account operations
  - Handles metric collection

#### 3. Central S3 Bucket
- **Purpose**: Centralized metric storage
- **Structure**:
```plaintext
central-disk-metrics/
├── metrics/
│   ├── account1/
│   │   ├── YYYY-MM-DD/
│   │   └── metrics.json
│   ├── account2/
│   └── account3/
```
- **Configuration**:
  - Versioning enabled
  - Encryption at rest
  - Cross-account access policies

#### 4. Lambda Function
- **Purpose**: Metric processing and aggregation
- **Triggers**:
  - S3 event notifications
  - Scheduled executions
- **Functions**:
  - Data normalization
  - Metric aggregation
  - CloudWatch integration

#### 5. CloudWatch Dashboard
- **Purpose**: Unified metric visualization
- **Features**:
  - Real-time monitoring
  - Cross-account views
  - Custom widgets
  - Automated updates

#### 6. Systems Manager
- **Purpose**: Secure cross-account management
- **Features**:
  - SSH-less access
  - Command execution
  - Session management
- **Security**:
  - IAM role-based access
  - Encrypted communications
  - Audit logging

### Member Account Components

#### 1. EC2 Instances
- **Configuration Requirements**:
  - Systems Manager Agent installed
  - Required IAM role attached
  - Appropriate tags for identification
- **Monitoring Scope**:
  - Disk utilization metrics
  - System performance
  - Resource allocation

#### 2. Systems Manager Agent
- **Purpose**: Enable secure management
- **Functions**:
  - Command execution
  - Metric collection
  - Status reporting
- **Security**:
  - Secure communication channel
  - Role-based authentication
  - Encrypted data transfer

## Data Flow

### 1. Access Management Flow

```markdown
IAM Identity Center
      │
      ├─────► AWS Organizations ─────► Member Accounts
      │
      ├─────► IAM Roles
              │
              └─────► AWS Resources
```
### 2. Metric Collection Flow
```plaintext
Ansible Control Node
└── Systems Manager
    └── SSM Agent
        └── EC2 Instance Metrics
            └── S3 Bucket
                └── Lambda Processing
                    └── CloudWatch
```

### 3. Processing Pipeline
1. **Collection Phase**
   - Ansible playbook execution
   - SSM command dispatch
   - Metric gathering

2. **Storage Phase**
   - S3 upload
   - Data organization
   - Version control

3. **Processing Phase**
   - Lambda trigger
   - Data normalization
   - Metric aggregation

4. **Visualization Phase**
   - CloudWatch ingestion
   - Dashboard update
   - Alert evaluation

## Security Architecture

### 1. Authentication & Authorization
- **IAM Identity Center**:
  - Centralized user management
  - Role-based access control
  - Multi-factor authentication

- **Cross-Account Access**:
  - Trust relationships
  - Minimum privilege principle
  - Role assumption policies

### 2. Network Security
- **Systems Manager**:
  - No inbound ports required
  - VPC endpoints
  - Encrypted communications

- **Data Transfer**:
  - TLS encryption in transit
  - S3 server-side encryption
  - CloudWatch encryption

### 3. Audit & Compliance
- **Logging**:
  - CloudTrail integration
  - S3 access logs
  - Systems Manager session logs

- **Monitoring**:
  - IAM activity monitoring
  - Resource access tracking
  - Compliance reporting

## Scaling Design

### 1. Account Scaling
- **Onboarding Process**:
  - Automated role creation
  - Systems Manager enablement
  - Metric collection integration

- **Management**:
  - Dynamic inventory management
  - Tag-based discovery
  - Automated configuration

### 2. Resource Scaling
- **Instance Discovery**:
  - Tag-based identification
  - Automatic integration
  - Dynamic resource management

- **Metric Processing**:
  - Lambda concurrency
  - S3 performance optimization
  - CloudWatch capacity management

### 3. Performance Considerations
- **Metric Collection**:
  - Batch processing
  - Concurrent execution
  - Rate limiting

- **Data Storage**:
  - S3 lifecycle policies
  - Data retention rules
  - Performance optimization


