# Troubleshooting Guide

## Memory and Disk Utilization Not Showing in Dashboard

### Root Cause
Memory and disk metrics are **custom metrics** collected by the CloudWatch Agent, not standard EC2 metrics. They require the agent to be properly installed and running.

### Quick Checks

#### 1. Wait for CloudWatch Agent to Start
**Time Required**: 5-10 minutes after instance launch

The CloudWatch Agent needs time to:
- Install via user_data script
- Start collecting metrics
- Send first data points to CloudWatch

**Solution**: Wait 10 minutes after deployment, then refresh the dashboard.

#### 2. Verify CloudWatch Agent is Running

SSH to your EC2 instance:
```bash
# Using SSM Session Manager (recommended)
aws ssm start-session --target $(terraform output -raw instance_id)

# Or traditional SSH
ssh ec2-user@$(terraform output -raw instance_public_ip)
```

Check agent status:
```bash
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl \
    -a query -m ec2 -c default -s
```

**Expected Output**:
```json
{
  "status": "running",
  "starttime": "...",
  "version": "..."
}
```

If status is **not running**:
```bash
# Start the agent
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl \
    -a fetch-config -m ec2 -s \
    -c file:/opt/aws/amazon-cloudwatch-agent/etc/config.json
```

#### 3. Check Agent Logs

```bash
# View agent logs
sudo tail -f /opt/aws/amazon-cloudwatch-agent/logs/amazon-cloudwatch-agent.log

# Check for errors
sudo grep -i error /opt/aws/amazon-cloudwatch-agent/logs/amazon-cloudwatch-agent.log
```

**Common Errors**:
- Permission issues → Check IAM role
- Configuration errors → Validate config.json
- Network issues → Check security groups

#### 4. Verify Metrics in CloudWatch

Check if metrics are being published:
```bash
# List all CWAgent metrics for your instance
aws cloudwatch list-metrics \
    --namespace CWAgent \
    --dimensions Name=InstanceId,Value=$(terraform output -raw instance_id)
```

**Expected Output**: List of metrics including:
- `mem_used_percent`
- `disk_used_percent`
- `diskio_read_bytes`
- `diskio_write_bytes`

If **no metrics** appear:
- Agent is not running
- IAM role missing CloudWatchAgentServerPolicy
- Configuration file has errors

#### 5. Check Specific Metric Data

```bash
# Check memory metrics
aws cloudwatch get-metric-statistics \
    --namespace CWAgent \
    --metric-name mem_used_percent \
    --dimensions Name=InstanceId,Value=$(terraform output -raw instance_id) \
    --start-time $(date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%S) \
    --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
    --period 300 \
    --statistics Average

# Check disk metrics
aws cloudwatch get-metric-statistics \
    --namespace CWAgent \
    --metric-name disk_used_percent \
    --dimensions Name=InstanceId,Value=$(terraform output -raw instance_id) Name=path,Value=/ \
    --start-time $(date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%S) \
    --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
    --period 300 \
    --statistics Average
```

### Common Issues and Solutions

#### Issue 1: "No data available" in Dashboard

**Causes**:
1. CloudWatch Agent not started yet (wait 5-10 minutes)
2. Agent crashed or stopped
3. Wrong metric dimensions in dashboard
4. IAM permissions missing

**Solutions**:
```bash
# 1. Check if agent is installed
ls -la /opt/aws/amazon-cloudwatch-agent/

# 2. Check if config exists
cat /opt/aws/amazon-cloudwatch-agent/etc/config.json

# 3. Restart agent
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl \
    -a stop -m ec2 -c default
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl \
    -a fetch-config -m ec2 -s \
    -c file:/opt/aws/amazon-cloudwatch-agent/etc/config.json

# 4. Check IAM role
aws iam list-attached-role-policies \
    --role-name ec2-monitoring-ec2-cloudwatch-role
```

#### Issue 2: Disk Metrics Show Wrong Device

**Cause**: Dashboard was configured for specific device names (nvme0n1p1, xfs) which may not match your instance.

**Solution**: ✅ **Already Fixed!** The dashboard now uses only `path: "/"` dimension, which works across all device types.

**To verify your actual disk setup**:
```bash
# On EC2 instance
df -h
lsblk
mount | grep "on / "
```

#### Issue 3: Memory Metrics Not Appearing

**Cause**: Agent configuration issue or permissions

**Solutions**:
```bash
# 1. Verify agent is collecting memory metrics
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl \
    -a query -m ec2 -c default -s

# 2. Check if metrics are in config
cat /opt/aws/amazon-cloudwatch-agent/etc/config.json | grep -A 10 '"mem"'

# 3. Manually test metric collection
free -m  # Should show memory info

# 4. Check agent can write to CloudWatch
aws cloudwatch put-metric-data \
    --namespace TestNamespace \
    --metric-name TestMetric \
    --value 1
```

#### Issue 4: Alarms Not Triggering

**Cause**: Alarm dimensions don't match actual metric dimensions

**Solution**: ✅ **Already Fixed!** Alarms now use simplified dimensions.

**To verify alarm configuration**:
```bash
# Check alarm details
aws cloudwatch describe-alarms \
    --alarm-names ec2-monitoring-high-memory ec2-monitoring-high-disk

# Check alarm history
aws cloudwatch describe-alarm-history \
    --alarm-name ec2-monitoring-high-memory \
    --max-records 10
```

### Dashboard Dimension Issues

#### Old Configuration (Problematic):
```json
["CWAgent", "disk_used_percent", "InstanceId", "i-xxx", "path", "/", "device", "nvme0n1p1", "fstype", "xfs"]
```
**Problem**: Too specific - only works with exact device/filesystem match

#### New Configuration (Fixed):
```json
["CWAgent", "disk_used_percent", "InstanceId", "i-xxx", "path", "/"]
```
**Solution**: Uses only essential dimensions - works with any device/filesystem

### Verification Checklist

After deployment, verify:

- [ ] **Wait 10 minutes** for agent to start and send metrics
- [ ] **Check agent status** - should be "running"
- [ ] **List CWAgent metrics** - should see mem_used_percent, disk_used_percent
- [ ] **View dashboard** - all widgets should show data
- [ ] **Check alarms** - should be in OK or ALARM state (not INSUFFICIENT_DATA)
- [ ] **Test stress** - run CPU/memory stress to verify metrics update

### Quick Verification Commands

```bash
# All-in-one verification script
INSTANCE_ID=$(terraform output -raw instance_id)

echo "1. Checking if metrics exist..."
aws cloudwatch list-metrics --namespace CWAgent \
    --dimensions Name=InstanceId,Value=$INSTANCE_ID \
    --query 'Metrics[*].MetricName' --output table

echo "2. Checking recent memory data..."
aws cloudwatch get-metric-statistics \
    --namespace CWAgent \
    --metric-name mem_used_percent \
    --dimensions Name=InstanceId,Value=$INSTANCE_ID \
    --start-time $(date -u -d '30 minutes ago' +%Y-%m-%dT%H:%M:%S) \
    --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
    --period 300 \
    --statistics Average \
    --query 'Datapoints[*].[Timestamp,Average]' --output table

echo "3. Checking recent disk data..."
aws cloudwatch get-metric-statistics \
    --namespace CWAgent \
    --metric-name disk_used_percent \
    --dimensions Name=InstanceId,Value=$INSTANCE_ID Name=path,Value=/ \
    --start-time $(date -u -d '30 minutes ago' +%Y-%m-%dT%H:%M:%S) \
    --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
    --period 300 \
    --statistics Average \
    --query 'Datapoints[*].[Timestamp,Average]' --output table

echo "4. Checking alarm states..."
aws cloudwatch describe-alarms \
    --alarm-name-prefix ec2-monitoring \
    --query 'MetricAlarms[*].[AlarmName,StateValue]' --output table
```

### Still Not Working?

If metrics still don't appear after following all steps:

1. **Redeploy the instance**:
   ```bash
   terraform taint aws_instance.monitored
   terraform apply
   ```

2. **Check CloudWatch Agent installation**:
   ```bash
   # SSH to instance
   aws ssm start-session --target $INSTANCE_ID
   
   # Check user_data execution
   sudo cat /var/log/cloud-init-output.log | grep -i cloudwatch
   ```

3. **Manually install agent** (if user_data failed):
   ```bash
   # Download and install
   wget https://s3.amazonaws.com/amazoncloudwatch-agent/amazon_linux/amd64/latest/amazon-cloudwatch-agent.rpm
   sudo rpm -U ./amazon-cloudwatch-agent.rpm
   
   # Copy config (from local machine)
   # Then on instance:
   sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl \
       -a fetch-config -m ec2 -s \
       -c file:/opt/aws/amazon-cloudwatch-agent/etc/config.json
   ```

4. **Check AWS Service Health**:
   - Visit AWS Service Health Dashboard
   - Check for CloudWatch service issues in your region

### Getting Help

If you're still experiencing issues:

1. **Collect diagnostic information**:
   ```bash
   # Save to file
   {
     echo "=== Instance Info ==="
     aws ec2 describe-instances --instance-ids $INSTANCE_ID
     
     echo "=== IAM Role ==="
     aws ec2 describe-instances --instance-ids $INSTANCE_ID \
         --query 'Reservations[0].Instances[0].IamInstanceProfile'
     
     echo "=== CloudWatch Metrics ==="
     aws cloudwatch list-metrics --namespace CWAgent \
         --dimensions Name=InstanceId,Value=$INSTANCE_ID
     
     echo "=== Alarms ==="
     aws cloudwatch describe-alarms --alarm-name-prefix ec2-monitoring
   } > diagnostics.txt
   ```

2. **Review logs**:
   - CloudWatch Agent logs: `/opt/aws/amazon-cloudwatch-agent/logs/`
   - User data logs: `/var/log/cloud-init-output.log`
   - System logs: `/var/log/messages`

3. **Check documentation**:
   - [CloudWatch Agent Troubleshooting](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/troubleshooting-CloudWatch-Agent.html)
   - [CloudWatch Metrics Troubleshooting](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/CloudWatch-metric-streams-troubleshoot.html)

---

## Summary of Fixes Applied

✅ **Dashboard Configuration Updated**:
- Removed hardcoded device names (`nvme0n1p1`, `xfs`)
- Now uses only `path: "/"` for disk metrics
- Works with any device type or filesystem

✅ **Alarm Configuration Updated**:
- Simplified disk alarm dimensions
- Removed device and fstype requirements
- More reliable alarm triggering

✅ **Metrics Being Tracked**:
- ✓ Memory utilization (mem_used_percent)
- ✓ Disk utilization (disk_used_percent)
- ✓ CPU utilization (CPUUtilization)
- ✓ Network traffic
- ✓ Disk I/O
- ✓ Status checks

**Next Steps**: Redeploy with `terraform apply` to apply the fixes!
