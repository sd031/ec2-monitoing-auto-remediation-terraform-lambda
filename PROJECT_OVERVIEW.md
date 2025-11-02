# Project Overview: EC2 Monitoring with CloudWatch & Automated Remediation

## 🎯 Learning Objectives

This project demonstrates comprehensive AWS monitoring and automation capabilities:

### CloudWatch Concepts Covered

1. **CloudWatch Metrics**
   - Standard EC2 metrics (CPU, Network, Status Checks)
   - Custom metrics via CloudWatch Agent
   - Metric namespaces and dimensions
   - Metric collection intervals and retention

2. **CloudWatch Logs**
   - Log groups and streams
   - Log collection from EC2 instances
   - Log retention policies
   - Log insights and queries

3. **CloudWatch Dashboards**
   - Time series widgets
   - Single value widgets
   - Log widgets
   - Dashboard JSON structure
   - Metric visualization best practices

4. **CloudWatch Alarms**
   - Threshold-based alarms
   - Evaluation periods and statistics
   - Alarm states (OK, ALARM, INSUFFICIENT_DATA)
   - Alarm actions (SNS, Lambda)
   - Composite alarms (future enhancement)

### AWS Services Integration

- **EC2**: Compute instance with detailed monitoring
- **CloudWatch**: Metrics, logs, dashboards, alarms
- **SNS**: Email notifications
- **Lambda**: Automated remediation (Python 3.11)
- **IAM**: Roles and policies for secure access
- **VPC**: Network isolation and security
- **Systems Manager**: SSM Session Manager for secure access

### Automation & DevOps

- **Infrastructure as Code**: Terraform for reproducible deployments
- **Automated Remediation**: Lambda-based self-healing
- **CI/CD Ready**: Scripts for automated deployment
- **Testing**: Stress test scripts included
- **Documentation**: Comprehensive guides and architecture docs

## 🏗️ What Gets Deployed

### Infrastructure Components

```
1 VPC
  ├── 1 Public Subnet
  ├── 1 Internet Gateway
  ├── 1 Route Table
  └── 1 Security Group

1 EC2 Instance (t3.micro)
  ├── CloudWatch Agent installed
  ├── Detailed monitoring enabled
  └── IAM role attached

4 CloudWatch Alarms
  ├── High CPU (>80%)
  ├── High Memory (>80%)
  ├── High Disk (>85%)
  └── Status Check Failed

1 CloudWatch Dashboard
  └── 10 widgets (metrics + logs)

1 CloudWatch Log Group
  └── 3 log streams

1 SNS Topic
  └── Email subscription

1 Lambda Function
  ├── Python 3.11 runtime
  ├── x86_64 architecture
  └── Auto-remediation logic

3 IAM Roles
  ├── EC2 role (CloudWatch Agent)
  ├── Lambda role (Remediation)
  └── Instance profile
```

## 📊 Monitoring Capabilities

### Metrics Monitored

**Standard Metrics (5-minute intervals)**:
- CPU Utilization (%)
- Network In/Out (bytes)
- Status Checks (pass/fail)

**Custom Metrics (1-minute intervals)**:
- Memory Used (%)
- Memory Available (MB)
- Disk Used (%)
- Disk Free (GB)
- Disk I/O Read/Write (bytes)
- Network Connections (count)
- Swap Usage (%)

### Logs Collected

- **System Logs**: `/var/log/messages`
- **Security Logs**: `/var/log/secure`
- **Custom Logs**: `/var/log/system-info.log`

### Alarms Configured

| Alarm | Metric | Threshold | Period | Action |
|-------|--------|-----------|--------|--------|
| High CPU | CPUUtilization | 80% | 10 min | SNS + Lambda |
| High Memory | mem_used_percent | 80% | 10 min | SNS + Lambda |
| High Disk | disk_used_percent | 85% | 10 min | SNS only |
| Status Failed | StatusCheckFailed | > 0 | 2 min | SNS + Lambda |

## 🤖 Automated Remediation

### How It Works

1. **Alarm Triggers**: CloudWatch alarm enters ALARM state
2. **SNS Notification**: Sends message to SNS topic
3. **Lambda Invocation**: SNS triggers Lambda function
4. **Remediation Logic**:
   - Parse alarm details
   - Identify affected instance
   - Check cooldown period (30 min)
   - Execute remediation action (reboot)
   - Send success/failure notification
5. **Cooldown**: Prevents repeated actions

### Remediation Actions

- **High CPU**: Reboot instance
- **High Memory**: Reboot instance
- **Status Check Failed**: Reboot instance
- **High Disk**: Notification only (manual intervention)

### Safety Features

- **Cooldown Period**: 30 minutes between actions
- **State Validation**: Only acts on running instances
- **Error Handling**: Graceful failures with notifications
- **Audit Trail**: All actions logged to CloudWatch

## 🚀 Deployment Options

### Quick Deploy (Recommended)
```bash
./scripts/deploy.sh
```

### Make Commands
```bash
make deploy    # Full deployment
make destroy   # Cleanup
make test      # Show test commands
```

### Manual Terraform
```bash
terraform init
terraform plan
terraform apply
```

## 🧪 Testing Scenarios

### 1. CPU Stress Test
```bash
# Triggers: High CPU alarm
# Expected: Instance reboots after 10 minutes
sudo stress --cpu $(nproc) --timeout 300s
```

### 2. Memory Stress Test
```bash
# Triggers: High Memory alarm
# Expected: Instance reboots after 10 minutes
sudo stress --vm 1 --vm-bytes 512M --timeout 300s
```

### 3. Disk Fill Test
```bash
# Triggers: High Disk alarm
# Expected: Email notification (no auto-remediation)
dd if=/dev/zero of=/tmp/bigfile bs=1M count=1000
```

## 📈 Dashboard Features

### Real-Time Monitoring
- CPU utilization trends
- Memory usage patterns
- Disk space consumption
- Network traffic analysis
- Disk I/O performance
- Status check results

### Historical Analysis
- 1-hour, 3-hour, 12-hour, 1-day views
- Metric comparison
- Anomaly detection
- Trend identification

### Log Integration
- Recent log entries
- Log search and filter
- Log insights queries

## 💰 Cost Breakdown

### Monthly Costs (us-east-1)

| Service | Usage | Cost |
|---------|-------|------|
| EC2 t3.micro | 730 hours | $7.50 |
| CloudWatch Metrics | 10 custom | $3.00 |
| CloudWatch Alarms | 4 alarms | $0.40 |
| CloudWatch Logs | 5 GB | $2.50 |
| CloudWatch Dashboard | 1 dashboard | $3.00 |
| Lambda | 100 invocations | $0.00 |
| SNS | 100 emails | $0.00 |
| Data Transfer | 1 GB | $0.09 |
| **Total** | | **~$16.49** |

### Cost Optimization Tips

1. **Use Reserved Instances**: Save 30-70% on EC2
2. **Reduce Metric Frequency**: Change to 5-minute intervals
3. **Log Filtering**: Only collect critical logs
4. **Alarm Consolidation**: Combine related alarms
5. **Auto-Stop**: Stop instances during off-hours

## 🔒 Security Features

### Network Security
- VPC isolation
- Security group restrictions (SSH only)
- No public database access
- Private subnet option available

### IAM Security
- Least privilege access
- Service-specific roles
- No hardcoded credentials
- Managed policies only

### Data Security
- Logs encrypted at rest
- HTTPS for all API calls
- No sensitive data in logs
- Secure parameter store ready

### Compliance
- CloudTrail integration ready
- AWS Config rules ready
- Security Hub integration ready

## 📚 Documentation Structure

```
README.md           - Main documentation (comprehensive)
QUICKSTART.md       - 5-minute setup guide
ARCHITECTURE.md     - Detailed architecture documentation
PROJECT_OVERVIEW.md - This file (high-level overview)
```

## 🎓 Skills Demonstrated

### AWS Services
✅ EC2 instance management
✅ CloudWatch metrics, logs, dashboards, alarms
✅ SNS notifications
✅ Lambda functions (Python)
✅ IAM roles and policies
✅ VPC networking
✅ Systems Manager

### DevOps Practices
✅ Infrastructure as Code (Terraform)
✅ Automated deployment scripts
✅ Configuration management
✅ Monitoring and alerting
✅ Incident response automation
✅ Documentation

### Programming
✅ Python 3.11 (Lambda function)
✅ Bash scripting (deployment automation)
✅ JSON (configuration files)
✅ HCL (Terraform)

### Best Practices
✅ Idempotent operations
✅ Error handling
✅ Logging and observability
✅ Security by design
✅ Cost optimization
✅ Scalability considerations

## 🔄 Workflow

### Development Workflow
```
1. Modify Terraform/Lambda code
2. Run: make build
3. Run: terraform plan
4. Review changes
5. Run: terraform apply
6. Test changes
7. Commit to Git
```

### Testing Workflow
```
1. Deploy infrastructure
2. Wait for metrics (5-10 min)
3. Run stress tests
4. Monitor dashboard
5. Verify alarms trigger
6. Check email notifications
7. Verify remediation
8. Review logs
```

### Troubleshooting Workflow
```
1. Check CloudWatch dashboard
2. Review alarm history
3. Check Lambda logs
4. Verify EC2 instance state
5. Check CloudWatch Agent status
6. Review SNS subscriptions
7. Validate IAM permissions
```

## 🌟 Key Features

### Self-Healing Infrastructure
- Automatic problem detection
- Intelligent remediation
- Minimal downtime
- Audit trail

### Comprehensive Monitoring
- System-level metrics
- Application-ready
- Real-time dashboards
- Historical analysis

### Proactive Alerting
- Multiple notification channels
- Configurable thresholds
- Evaluation periods
- State tracking

### Production-Ready
- Terraform managed
- Version controlled
- Documented
- Tested

## 🚀 Future Enhancements

### Short Term
- [ ] Add more remediation actions (scale, snapshot)
- [ ] Implement Slack notifications
- [ ] Add CloudWatch Anomaly Detection
- [ ] Create custom metrics for applications

### Medium Term
- [ ] Multi-region deployment
- [ ] Auto Scaling integration
- [ ] Application Load Balancer
- [ ] RDS monitoring

### Long Term
- [ ] Machine learning predictions
- [ ] Cost optimization automation
- [ ] Compliance monitoring
- [ ] Full observability stack

## 📞 Support Resources

### AWS Documentation
- [CloudWatch User Guide](https://docs.aws.amazon.com/cloudwatch/)
- [Lambda Developer Guide](https://docs.aws.amazon.com/lambda/)
- [Terraform AWS Provider](https://registry.terraform.io/providers/hashicorp/aws/)

### Troubleshooting
- Check CloudWatch Logs
- Review Terraform state
- Validate AWS credentials
- Verify IAM permissions

### Community
- AWS Forums
- Stack Overflow
- Terraform Community
- GitHub Issues

## ✅ Success Criteria

You've successfully completed this project when you can:

1. ✅ Deploy infrastructure with one command
2. ✅ View metrics in CloudWatch dashboard
3. ✅ Trigger alarms with stress tests
4. ✅ Receive email notifications
5. ✅ Observe automatic remediation
6. ✅ Review Lambda logs
7. ✅ Understand the architecture
8. ✅ Modify thresholds and redeploy
9. ✅ Clean up all resources

## 🎉 Conclusion

This project provides a **production-ready foundation** for AWS monitoring and automation. It demonstrates:

- **Real-world scenarios**: Actual problems and solutions
- **Best practices**: AWS Well-Architected Framework
- **Automation**: Infrastructure as Code and self-healing
- **Observability**: Comprehensive monitoring and logging
- **Cost-effective**: Optimized for learning and small workloads

**Perfect for**: DevOps engineers, Cloud architects, AWS learners, and anyone building reliable cloud infrastructure.

---

**Ready to deploy? Start with [QUICKSTART.md](QUICKSTART.md)!**
