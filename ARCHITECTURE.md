# Architecture Documentation

## Overview

This document provides detailed architectural information about the EC2 CloudWatch Monitoring solution.

## Components

### 1. EC2 Instance

**Purpose**: Monitored compute resource

**Configuration**:
- AMI: Amazon Linux 2 (latest)
- Instance Type: t3.micro (configurable)
- Detailed Monitoring: Enabled
- IAM Role: CloudWatch Agent permissions

**Installed Software**:
- CloudWatch Agent (for custom metrics)
- Stress tool (for testing)
- System monitoring scripts

### 2. CloudWatch Agent

**Purpose**: Collect custom metrics and logs from EC2

**Metrics Collected**:
- CPU usage (idle, iowait, guest time)
- Memory (used %, available, total)
- Disk (used %, free, total)
- Disk I/O (read/write bytes, operations)
- Network statistics (connections)
- Swap usage

**Logs Collected**:
- `/var/log/messages` - System messages
- `/var/log/secure` - Security logs
- `/var/log/system-info.log` - Custom monitoring logs

**Collection Interval**: 60 seconds

### 3. CloudWatch Metrics

**Standard Metrics** (AWS/EC2 namespace):
- CPUUtilization
- NetworkIn/NetworkOut
- StatusCheckFailed
- StatusCheckFailed_Instance
- StatusCheckFailed_System

**Custom Metrics** (CWAgent namespace):
- mem_used_percent
- disk_used_percent
- diskio_read_bytes
- diskio_write_bytes
- cpu_usage_idle
- swap_used_percent

### 4. CloudWatch Alarms

#### High CPU Alarm
- **Metric**: CPUUtilization
- **Threshold**: 80% (configurable)
- **Evaluation**: 2 periods of 5 minutes
- **Actions**: SNS notification + Lambda invocation

#### High Memory Alarm
- **Metric**: mem_used_percent
- **Threshold**: 80% (configurable)
- **Evaluation**: 2 periods of 5 minutes
- **Actions**: SNS notification + Lambda invocation

#### High Disk Alarm
- **Metric**: disk_used_percent
- **Threshold**: 85% (configurable)
- **Evaluation**: 2 periods of 5 minutes
- **Actions**: SNS notification only

#### Status Check Failed Alarm
- **Metric**: StatusCheckFailed
- **Threshold**: > 0
- **Evaluation**: 2 periods of 1 minute
- **Actions**: SNS notification + Lambda invocation

### 5. CloudWatch Dashboard

**Widgets**:
1. CPU Utilization (time series)
2. Memory Utilization (time series)
3. Disk Utilization (time series)
4. Network Traffic (time series)
5. Disk I/O (time series)
6. Status Checks (time series)
7. Recent Logs (log widget)
8. Current CPU (single value)
9. Current Memory (single value)
10. Current Disk (single value)

**Refresh**: Auto-refresh every 1 minute

### 6. SNS Topic

**Purpose**: Notification distribution

**Subscriptions**:
- Email (user-configured)
- Lambda function (automatic)

**Message Format**:
- Subject: Alarm name and state
- Body: Detailed alarm information

### 7. Lambda Function

**Purpose**: Automated remediation

**Runtime**: Python 3.11
**Architecture**: x86_64
**Timeout**: 60 seconds
**Memory**: 128 MB (default)

**Triggers**:
- CloudWatch Alarms (via SNS)

**Permissions**:
- EC2: Describe, Reboot, Stop, Start instances
- CloudWatch: Describe alarms, Get metrics
- SNS: Publish messages
- Logs: Create log groups/streams, Put log events

**Remediation Logic**:
1. Parse alarm event
2. Extract instance ID
3. Check instance state
4. Verify cooldown period
5. Execute remediation action
6. Send notification
7. Record action in CloudWatch

**Cooldown Mechanism**:
- Uses custom CloudWatch metrics
- Prevents repeated actions within 30 minutes
- Per-instance and per-alarm tracking

### 8. VPC and Networking

**VPC**:
- CIDR: 10.0.0.0/16
- DNS hostnames: Enabled
- DNS support: Enabled

**Subnet**:
- Type: Public
- CIDR: 10.0.1.0/24
- Auto-assign public IP: Enabled

**Internet Gateway**:
- Attached to VPC
- Route to 0.0.0.0/0

**Security Group**:
- Ingress: SSH (port 22) from anywhere
- Egress: All traffic

### 9. IAM Roles and Policies

#### EC2 Role
**Managed Policies**:
- CloudWatchAgentServerPolicy
- AmazonSSMManagedInstanceCore

**Purpose**:
- Allow CloudWatch Agent to publish metrics/logs
- Enable SSM Session Manager access

#### Lambda Role
**Custom Policy**:
```json
{
  "EC2": [
    "DescribeInstances",
    "RebootInstances",
    "StopInstances",
    "StartInstances",
    "DescribeInstanceStatus"
  ],
  "CloudWatch": [
    "DescribeAlarms",
    "GetMetricStatistics",
    "PutMetricData"
  ],
  "SNS": [
    "Publish"
  ],
  "Logs": [
    "CreateLogGroup",
    "CreateLogStream",
    "PutLogEvents"
  ]
}
```

## Data Flow

### Monitoring Flow

```
EC2 Instance
    ↓ (CloudWatch Agent)
CloudWatch Metrics
    ↓ (Every 60 seconds)
Metric Data Points
    ↓ (Evaluation)
CloudWatch Alarms
    ↓ (If threshold exceeded)
[SNS Topic] ← [Lambda Function]
    ↓              ↓
Email          Remediation
```

### Remediation Flow

```
CloudWatch Alarm (ALARM state)
    ↓
SNS Topic
    ↓
Lambda Function
    ↓
1. Parse event
2. Get instance info
3. Check cooldown
4. Execute action (reboot/stop)
5. Send notification
6. Record metric
    ↓
EC2 Instance (rebooted/stopped)
    ↓
SNS Notification (success/failure)
```

### Log Flow

```
EC2 Instance
    ↓ (CloudWatch Agent)
CloudWatch Logs
    ↓
Log Groups
    ├── /aws/ec2/ec2-monitoring/messages
    ├── /aws/ec2/ec2-monitoring/secure
    └── /aws/ec2/ec2-monitoring/system-info
    ↓
Dashboard Log Widget
```

## Scaling Considerations

### Horizontal Scaling

To monitor multiple instances:

1. **Modify Terraform**:
   ```hcl
   resource "aws_instance" "monitored" {
     count = var.instance_count
     # ... configuration
   }
   ```

2. **Update Alarms**:
   ```hcl
   resource "aws_cloudwatch_metric_alarm" "cpu_high" {
     count = var.instance_count
     # ... configuration
     dimensions = {
       InstanceId = aws_instance.monitored[count.index].id
     }
   }
   ```

3. **Lambda Modification**:
   - No changes needed (handles any instance)

### Vertical Scaling

To handle higher loads:

1. **Increase Lambda resources**:
   ```hcl
   memory_size = 256
   timeout     = 120
   ```

2. **Adjust metric collection**:
   ```json
   "metrics_collection_interval": 30
   ```

3. **Optimize dashboard**:
   - Reduce time range
   - Increase period

## High Availability

### Current Setup
- Single AZ deployment
- Single instance

### HA Improvements

1. **Multi-AZ Deployment**:
   ```hcl
   resource "aws_subnet" "public" {
     count             = length(var.availability_zones)
     availability_zone = var.availability_zones[count.index]
     # ... configuration
   }
   ```

2. **Auto Scaling Group**:
   ```hcl
   resource "aws_autoscaling_group" "monitored" {
     min_size         = 2
     max_size         = 4
     desired_capacity = 2
     # ... configuration
   }
   ```

3. **Application Load Balancer**:
   ```hcl
   resource "aws_lb" "main" {
     load_balancer_type = "application"
     # ... configuration
   }
   ```

## Security Architecture

### Network Security

```
Internet
    ↓ (HTTPS/443)
CloudWatch API
    ↑ (Metrics/Logs)
EC2 Instance (Private IP)
    ↑ (SSH/22 - Optional)
Bastion Host / SSM
```

### IAM Security

- **Principle of Least Privilege**: Each role has minimal permissions
- **No Inline Policies**: All policies are managed
- **Service-Specific Roles**: Separate roles for EC2 and Lambda
- **No User Credentials**: All access via IAM roles

### Data Security

- **Encryption at Rest**: CloudWatch Logs encrypted
- **Encryption in Transit**: All API calls over HTTPS
- **No Sensitive Data**: No credentials in code or logs
- **Secure Parameter Store**: Can be used for sensitive config

## Cost Optimization

### Current Costs

- **EC2**: Pay for instance hours
- **CloudWatch**: Pay for metrics, alarms, logs, dashboard
- **Lambda**: Pay per invocation (usually free tier)
- **SNS**: Pay per notification (usually free tier)

### Optimization Strategies

1. **Use Reserved Instances**: Save up to 72% on EC2
2. **Reduce Metric Resolution**: Increase collection interval
3. **Log Filtering**: Only collect necessary logs
4. **Alarm Consolidation**: Combine related alarms
5. **Dashboard Optimization**: Remove unused widgets
6. **Auto-Stop**: Stop instances during off-hours

### Cost Monitoring

Add cost alarms:
```hcl
resource "aws_cloudwatch_metric_alarm" "billing" {
  alarm_name          = "high-billing"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = "1"
  metric_name         = "EstimatedCharges"
  namespace           = "AWS/Billing"
  period              = "21600"
  statistic           = "Maximum"
  threshold           = "50"
  alarm_description   = "Billing alarm"
}
```

## Disaster Recovery

### Backup Strategy

1. **Infrastructure**: Terraform state in S3 with versioning
2. **Configuration**: Git repository
3. **Logs**: CloudWatch Logs retention (7 days)
4. **Metrics**: CloudWatch retains for 15 months

### Recovery Procedure

1. **Complete Loss**:
   ```bash
   terraform apply
   ```

2. **Partial Loss**:
   ```bash
   terraform taint aws_instance.monitored
   terraform apply
   ```

3. **Configuration Rollback**:
   ```bash
   git checkout <previous-commit>
   terraform apply
   ```

## Monitoring the Monitor

### Lambda Monitoring

- **CloudWatch Logs**: `/aws/lambda/ec2-monitoring-auto-remediation`
- **Metrics**: Invocations, Errors, Duration, Throttles
- **Alarms**: Set on Lambda errors

### Agent Monitoring

- **Logs**: `/opt/aws/amazon-cloudwatch-agent/logs/`
- **Status**: Check via SSM or SSH
- **Metrics**: Agent publishes its own health metrics

### Alarm Monitoring

- **CloudWatch Console**: View alarm history
- **SNS**: Notifications for all state changes
- **Lambda**: Logs all remediation actions

## Future Enhancements

1. **Machine Learning**: Use CloudWatch Anomaly Detection
2. **Predictive Scaling**: Forecast metrics and scale proactively
3. **Cost Analysis**: Integrate with AWS Cost Explorer
4. **Compliance**: Add AWS Config rules
5. **Security**: Integrate with AWS Security Hub
6. **Automation**: Use AWS Systems Manager for patching
7. **Observability**: Add X-Ray tracing
8. **Alerting**: Integrate with PagerDuty or Slack

## References

- [AWS Well-Architected Framework](https://aws.amazon.com/architecture/well-architected/)
- [CloudWatch Best Practices](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/Best_Practice_Recommended_Alarms_AWS_Services.html)
- [Lambda Best Practices](https://docs.aws.amazon.com/lambda/latest/dg/best-practices.html)
