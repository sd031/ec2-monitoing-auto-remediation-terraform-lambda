# Quick Start Guide

Get up and running in 5 minutes!

## Prerequisites Check

```bash
# Check Terraform
terraform version

# Check AWS CLI
aws --version

# Check AWS credentials
aws sts get-caller-identity

# Check Python
python3 --version
pip3 --version
```

If any are missing, install them first.

## Step-by-Step Deployment

### 1. Configure Settings (2 minutes)

```bash
cd /Users/sandipdas/aws_project_4

# Copy example config
cp terraform.tfvars.example terraform.tfvars

# Edit with your email
nano terraform.tfvars
```

**Required**: Change `alarm_email` to your email address!

### 2. Deploy Infrastructure (3 minutes)

```bash
# Option 1: Using deploy script (recommended)
./scripts/deploy.sh

# Option 2: Using Make
make deploy

# Option 3: Manual steps
make build          # Build Lambda package
terraform init      # Initialize Terraform
terraform plan      # Review changes
terraform apply     # Deploy
```

### 3. Confirm Email Subscription (1 minute)

1. Check your email inbox
2. Look for "AWS Notification - Subscription Confirmation"
3. Click "Confirm subscription"

### 4. Access Dashboard (1 minute)

```bash
# Get dashboard URL
terraform output dashboard_url

# Or copy this (replace region if needed)
open "https://console.aws.amazon.com/cloudwatch/home?region=us-east-1#dashboards"
```

## Testing (Optional)

### Test CPU Alarm

```bash
# Get instance ID
INSTANCE_ID=$(terraform output -raw instance_id)

# Connect via SSM
aws ssm start-session --target $INSTANCE_ID

# Run stress test (on EC2 instance)
sudo stress --cpu $(nproc) --timeout 300s
```

Wait 10 minutes, then check:
- Email for alarm notification
- Dashboard for spike in CPU
- Email for remediation notification (instance reboot)

## Viewing Results

### CloudWatch Console

```bash
# Dashboard
terraform output dashboard_url

# Alarms
open "https://console.aws.amazon.com/cloudwatch/home?region=us-east-1#alarmsV2:"

# Logs
open "https://console.aws.amazon.com/cloudwatch/home?region=us-east-1#logsV2:log-groups"
```

### Command Line

```bash
# Check alarm status
aws cloudwatch describe-alarms --alarm-names ec2-monitoring-high-cpu

# View recent metrics
aws cloudwatch get-metric-statistics \
  --namespace AWS/EC2 \
  --metric-name CPUUtilization \
  --dimensions Name=InstanceId,Value=$(terraform output -raw instance_id) \
  --start-time $(date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%S) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
  --period 300 \
  --statistics Average

# View Lambda logs
aws logs tail /aws/lambda/ec2-monitoring-auto-remediation --follow
```

## Common Issues

### "No terraform.tfvars file"
```bash
cp terraform.tfvars.example terraform.tfvars
nano terraform.tfvars  # Add your email
```

### "AWS credentials not configured"
```bash
aws configure
# Enter your AWS Access Key ID
# Enter your AWS Secret Access Key
# Enter your default region (e.g., us-east-1)
```

### "Terraform not found"
```bash
# macOS
brew install terraform

# Or download from
open "https://www.terraform.io/downloads"
```

### "Python not found"
```bash
# macOS
brew install python3
```

### "Metrics not showing in dashboard"
- Wait 5-10 minutes for CloudWatch Agent to start
- Check EC2 instance is running
- SSH to instance and check agent status:
  ```bash
  sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl \
    -a query -m ec2 -c default -s
  ```

### "Alarm not triggering"
- Verify threshold is set correctly
- Check metrics are being reported
- Ensure 2 evaluation periods have passed (10 minutes)
- View alarm history in CloudWatch console

## Cleanup

When you're done:

```bash
# Option 1: Using destroy script
./scripts/destroy.sh

# Option 2: Using Make
make destroy

# Option 3: Using Terraform
terraform destroy
```

This removes all resources and stops billing.

## Next Steps

1. **Explore Dashboard**: View all metrics and logs
2. **Test Alarms**: Run stress tests to trigger alarms
3. **Customize**: Adjust thresholds and actions
4. **Learn**: Read README.md and ARCHITECTURE.md
5. **Extend**: Add more instances or metrics

## Getting Help

- **README.md**: Comprehensive documentation
- **ARCHITECTURE.md**: Detailed architecture info
- **Makefile**: Run `make help` for all commands

## Estimated Costs

- **Development/Testing**: ~$0.50/day
- **Production**: ~$16/month

Use AWS Cost Explorer to monitor actual costs.

---

**You're all set! Happy monitoring! 🎉**
