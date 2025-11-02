# EC2 Monitoring with CloudWatch & Automated Remediation

A comprehensive AWS monitoring solution that demonstrates CloudWatch metrics, logs, dashboards, alarms, and automated remediation using Lambda functions.

## 🎯 Project Goals

- **Monitor EC2 instances** with detailed system metrics (CPU, Memory, Disk, Network)
- **Visualize metrics** using CloudWatch Dashboards
- **Trigger alerts** via SNS when thresholds are exceeded
- **Automate remediation** with Lambda functions (e.g., reboot instance on high CPU)
- **Centralize logs** in CloudWatch Logs

## 🏗️ Architecture

```
┌─────────────┐
│   EC2       │
│  Instance   │──────┐
│ (CW Agent)  │      │
└─────────────┘      │
                     │ Metrics & Logs
                     ▼
              ┌──────────────┐
              │  CloudWatch  │
              │   Metrics    │
              └──────┬───────┘
                     │
                     │ Threshold Exceeded
                     ▼
              ┌──────────────┐
              │  CloudWatch  │
              │    Alarms    │
              └──────┬───────┘
                     │
         ┌───────────┴───────────┐
         │                       │
         ▼                       ▼
    ┌─────────┐           ┌──────────┐
    │   SNS   │           │  Lambda  │
    │  Topic  │           │ Function │
    └────┬────┘           └────┬─────┘
         │                     │
         │ Email               │ Auto-Remediation
         ▼                     ▼
    ┌─────────┐           ┌──────────┐
    │  User   │           │   EC2    │
    │  Email  │           │  Reboot  │
    └─────────┘           └──────────┘
```

## 📋 Features

### CloudWatch Metrics
- **Standard EC2 Metrics**: CPU, Network, Status Checks
- **Custom Metrics via CloudWatch Agent**:
  - Memory utilization (%)
  - Disk utilization (%)
  - Disk I/O operations
  - Network connections
  - Swap usage

### CloudWatch Alarms
- **High CPU Alarm**: Triggers when CPU > 80% for 10 minutes
- **High Memory Alarm**: Triggers when Memory > 80% for 10 minutes
- **High Disk Alarm**: Triggers when Disk > 85% for 10 minutes
- **Status Check Failed**: Triggers on instance or system check failures

### CloudWatch Dashboard
- Real-time visualization of all metrics
- Historical trend analysis
- Single-value widgets for current status
- Log insights integration

### Automated Remediation
- **Lambda Function** automatically responds to alarms
- **Reboot Instance**: On high CPU or memory issues
- **Cooldown Period**: Prevents repeated actions (30 minutes)
- **SNS Notifications**: Sends detailed remediation reports
- **Error Handling**: Graceful failure with notifications

### CloudWatch Logs
- System logs (`/var/log/messages`)
- Security logs (`/var/log/secure`)
- Custom application logs
- 7-day retention policy

## 🚀 Quick Start

### Prerequisites

- **AWS Account** with appropriate permissions
- **AWS CLI** configured with credentials
- **Terraform** >= 1.0
- **Python 3.11+** and pip3
- **macOS** (scripts optimized for Mac, but work on Linux too)

### Installation

1. **Clone or navigate to the project directory**:
   ```bash
   cd /Users/sandipdas/aws_project_4
   ```

2. **Configure your settings**:
   ```bash
   cp terraform.tfvars.example terraform.tfvars
   nano terraform.tfvars
   ```
   
   Update with your email and preferences:
   ```hcl
   aws_region       = "us-east-1"
   project_name     = "ec2-monitoring"
   instance_type    = "t3.micro"
   alarm_email      = "your-email@example.com"  # REQUIRED
   cpu_threshold    = 80
   memory_threshold = 80
   disk_threshold   = 85
   ```

3. **Deploy the infrastructure**:
   ```bash
   # Using the deploy script (recommended)
   chmod +x scripts/deploy.sh
   ./scripts/deploy.sh
   
   # Or using Make
   make deploy
   ```

4. **Confirm SNS subscription**:
   - Check your email for SNS subscription confirmation
   - Click the confirmation link

5. **Wait for metrics** (5-10 minutes):
   - CloudWatch Agent needs time to start reporting
   - Check the dashboard after this period

## 📊 Using the Dashboard

Access your CloudWatch Dashboard:
```bash
# Get the dashboard URL
terraform output dashboard_url

# Or open directly
open "https://console.aws.amazon.com/cloudwatch/home?region=us-east-1#dashboards:name=ec2-monitoring-dashboard"
```

The dashboard includes:
- **CPU Utilization**: Average and maximum over time
- **Memory Utilization**: Percentage used
- **Disk Utilization**: Percentage used per mount point
- **Network Traffic**: In/out bytes
- **Disk I/O**: Read/write operations
- **Status Checks**: Instance and system health
- **Recent Logs**: Last 100 log entries
- **Current Values**: Single-value widgets for quick status

## 🧪 Testing the Monitoring

### Test CPU Alarm

SSH to the instance and run:
```bash
# Using SSM Session Manager (no SSH key needed)
aws ssm start-session --target $(terraform output -raw instance_id)

# Or traditional SSH
ssh ec2-user@$(terraform output -raw instance_public_ip)

# Run CPU stress test
sudo stress --cpu $(nproc) --timeout 300s
```

Or use the provided script:
```bash
# Copy script to instance
scp scripts/test_cpu_stress.sh ec2-user@<instance-ip>:~

# Run on instance
chmod +x test_cpu_stress.sh
./test_cpu_stress.sh
```

### Test Memory Alarm

```bash
# On the EC2 instance
sudo stress --vm 1 --vm-bytes 512M --timeout 300s
```

Or use the provided script:
```bash
# Copy script to instance
scp scripts/test_memory_stress.sh ec2-user@<instance-ip>:~

# Run on instance
chmod +x test_memory_stress.sh
./test_memory_stress.sh
```

### Expected Behavior

1. **After 2 evaluation periods** (10 minutes), the alarm triggers
2. **SNS notification** sent to your email
3. **Lambda function** executes automatically
4. **Instance reboots** (if configured action)
5. **Second notification** sent with remediation details
6. **Cooldown period** prevents repeated actions for 30 minutes

## 🔧 Configuration

### Adjusting Thresholds

Edit `terraform.tfvars`:
```hcl
cpu_threshold    = 90  # Increase to 90%
memory_threshold = 85  # Increase to 85%
disk_threshold   = 90  # Increase to 90%
```

Then apply changes:
```bash
terraform apply
```

### Modifying Remediation Actions

Edit `lambda/lambda_function.py`:
```python
REMEDIATION_ACTIONS = {
    'high-cpu': 'reboot',        # Change to 'stop' or add custom action
    'high-memory': 'reboot',
    'status-check-failed': 'reboot',
}
```

Rebuild and redeploy:
```bash
make build
terraform apply
```

### Changing Cooldown Period

Edit `lambda/lambda_function.py`:
```python
COOLDOWN_PERIOD = 60  # Change to 60 minutes
```

## 📁 Project Structure

```
.
├── main.tf                      # Main Terraform configuration
├── variables.tf                 # Input variables
├── outputs.tf                   # Output values
├── terraform.tfvars.example     # Example configuration
├── deploy.sh                    # Automated deployment script
├── destroy.sh                   # Cleanup script
├── Makefile                     # Make targets for common tasks
├── README.md                    # This file
├── .gitignore                   # Git ignore rules
│
├── lambda/
│   ├── lambda_function.py       # Auto-remediation Lambda (Python 3.11)
│   └── requirements.txt         # Python dependencies
│
├── scripts/
│   ├── build_lambda.sh          # Build Lambda package for x86_64
│   ├── user_data.sh             # EC2 initialization script
│   ├── test_cpu_stress.sh       # CPU stress test script
│   └── test_memory_stress.sh    # Memory stress test script
│
└── configs/
    ├── cloudwatch-config.json   # CloudWatch Agent configuration
    └── dashboard.json           # Dashboard template
```

## 🛠️ Make Commands

```bash
make help       # Show all available commands
make init       # Initialize Terraform
make build      # Build Lambda package
make plan       # Create Terraform plan
make deploy     # Deploy infrastructure
make destroy    # Destroy all resources
make clean      # Clean up local files
make test       # Show test commands
make validate   # Validate Terraform config
make fmt        # Format Terraform files
make outputs    # Show outputs
```

## 📈 Monitoring Costs

Estimated monthly costs (us-east-1):

| Service | Usage | Cost |
|---------|-------|------|
| EC2 (t3.micro) | 730 hours | ~$7.50 |
| CloudWatch Metrics | 10 custom metrics | ~$3.00 |
| CloudWatch Alarms | 4 alarms | ~$0.40 |
| CloudWatch Logs | 5 GB ingestion | ~$2.50 |
| CloudWatch Dashboard | 1 dashboard | $3.00 |
| Lambda | 100 invocations | ~$0.00 |
| SNS | 100 notifications | ~$0.00 |
| **Total** | | **~$16.40/month** |

> **Note**: Costs may vary based on usage. Use AWS Cost Explorer to monitor actual costs.

## 🔒 Security Best Practices

1. **IAM Roles**: Uses least-privilege IAM roles
2. **No Hardcoded Credentials**: All credentials via IAM roles
3. **Security Groups**: Minimal ingress rules (SSH only)
4. **Encryption**: CloudWatch Logs encrypted at rest
5. **VPC**: Resources deployed in isolated VPC

## 🐛 Troubleshooting

### CloudWatch Agent Not Reporting Metrics

```bash
# SSH to instance
aws ssm start-session --target <instance-id>

# Check agent status
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl \
    -a query -m ec2 -c default -s

# View agent logs
sudo tail -f /opt/aws/amazon-cloudwatch-agent/logs/amazon-cloudwatch-agent.log
```

### Lambda Function Not Triggering

1. Check Lambda logs:
   ```bash
   aws logs tail /aws/lambda/ec2-monitoring-auto-remediation --follow
   ```

2. Verify Lambda permissions:
   ```bash
   aws lambda get-policy --function-name ec2-monitoring-auto-remediation
   ```

3. Test Lambda manually:
   ```bash
   aws lambda invoke \
       --function-name ec2-monitoring-auto-remediation \
       --payload file://test-event.json \
       response.json
   ```

### Alarms Not Triggering

1. Check alarm state:
   ```bash
   aws cloudwatch describe-alarms \
       --alarm-names ec2-monitoring-high-cpu
   ```

2. Verify metrics are being reported:
   ```bash
   aws cloudwatch get-metric-statistics \
       --namespace CWAgent \
       --metric-name mem_used_percent \
       --dimensions Name=InstanceId,Value=<instance-id> \
       --start-time $(date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%S) \
       --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
       --period 300 \
       --statistics Average
   ```

### SNS Subscription Not Confirmed

1. Check SNS subscriptions:
   ```bash
   aws sns list-subscriptions-by-topic \
       --topic-arn $(terraform output -raw sns_topic_arn)
   ```

2. Resend confirmation:
   ```bash
   aws sns subscribe \
       --topic-arn $(terraform output -raw sns_topic_arn) \
       --protocol email \
       --notification-endpoint your-email@example.com
   ```

## 🧹 Cleanup

To destroy all resources:

```bash
# Using the destroy script
./scripts/destroy.sh

# Or using Make
make destroy

# Or using Terraform directly
terraform destroy
```

This will remove:
- EC2 instance
- VPC and networking
- CloudWatch alarms and dashboard
- SNS topic
- Lambda function
- IAM roles and policies
- CloudWatch log groups

## 📚 Learning Objectives

This project demonstrates:

1. **CloudWatch Metrics**:
   - Standard vs custom metrics
   - CloudWatch Agent installation and configuration
   - Metric namespaces and dimensions

2. **CloudWatch Logs**:
   - Log groups and streams
   - Log collection from EC2
   - Log retention policies

3. **CloudWatch Dashboards**:
   - Widget types (line, number, log)
   - Dashboard JSON structure
   - Metric visualization

4. **CloudWatch Alarms**:
   - Threshold-based alarms
   - Evaluation periods and statistics
   - Alarm actions (SNS, Lambda)

5. **SNS Notifications**:
   - Topic creation and subscriptions
   - Email notifications
   - Integration with CloudWatch

6. **Lambda Functions**:
   - Event-driven architecture
   - CloudWatch alarm integration
   - EC2 API operations
   - Error handling and logging
   - Cross-platform deployment (x86_64)

7. **Infrastructure as Code**:
   - Terraform best practices
   - Resource dependencies
   - Output values
   - Variable management

8. **Automated Remediation**:
   - Self-healing infrastructure
   - Cooldown periods
   - Idempotent operations

## 🤝 Contributing

Feel free to enhance this project:

- Add more remediation actions (scale up, snapshot, etc.)
- Implement more sophisticated remediation logic
- Add CloudWatch Insights queries
- Create custom metrics
- Add more alarm types
- Implement multi-region support

## 📝 License

This project is provided as-is for educational purposes.

## 🔗 Resources

- [AWS CloudWatch Documentation](https://docs.aws.amazon.com/cloudwatch/)
- [CloudWatch Agent Configuration](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/CloudWatch-Agent-Configuration-File-Details.html)
- [Terraform AWS Provider](https://registry.terraform.io/providers/hashicorp/aws/latest/docs)
- [AWS Lambda Python](https://docs.aws.amazon.com/lambda/latest/dg/lambda-python.html)

## 💡 Next Steps

After completing this project, consider:

1. **Add Auto Scaling**: Implement Auto Scaling based on CloudWatch metrics
2. **Multi-Instance Monitoring**: Extend to monitor multiple instances
3. **Custom Metrics**: Create application-specific metrics
4. **Log Analysis**: Use CloudWatch Insights for log analysis
5. **Cost Optimization**: Implement cost-based alarms and actions
6. **Compliance**: Add compliance monitoring and reporting
7. **Integration**: Connect with other AWS services (EventBridge, Step Functions)

---

**Happy Monitoring! 🎉**
# ec2-monitoing-auto-remediation-terraform-lambda
