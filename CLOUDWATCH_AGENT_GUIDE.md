# CloudWatch Agent Quick Reference Guide

## Ensuring CloudWatch Agent Runs and Collects Memory & Disk Data

### ✅ What's Already Configured

Your infrastructure automatically:
1. **Installs CloudWatch Agent** via user_data script
2. **Configures metrics collection** for:
   - Memory utilization (mem_used_percent)
   - Disk utilization (disk_used_percent)
   - CPU, Network, Disk I/O
3. **Starts the agent** automatically on instance launch
4. **Sends metrics to CloudWatch** every 60 seconds

### 📋 Quick Verification (From Your Local Machine)

After deploying, wait **5-10 minutes**, then run:

```bash
# Get instance ID
INSTANCE_ID=$(terraform output -raw instance_id)

# Check if metrics are being published
aws cloudwatch list-metrics \
    --namespace CWAgent \
    --dimensions Name=InstanceId,Value=$INSTANCE_ID

# Expected output: List of metrics including mem_used_percent and disk_used_percent
```

If you see metrics listed, **CloudWatch Agent is working!** ✅

### 🔍 Detailed Verification (On EC2 Instance)

#### Option 1: Use the Automated Verification Script

```bash
# SSH to instance
aws ssm start-session --target $(terraform output -raw instance_id)

# Run verification script (already on instance)
sudo bash /tmp/verify_cloudwatch_agent.sh
```

Or copy the script to the instance:
```bash
# From your local machine
scp scripts/verify_cloudwatch_agent.sh ec2-user@$(terraform output -raw instance_public_ip):~
ssh ec2-user@$(terraform output -raw instance_public_ip)
chmod +x verify_cloudwatch_agent.sh
sudo ./verify_cloudwatch_agent.sh
```

#### Option 2: Manual Verification Commands

```bash
# 1. Check if agent is installed
ls -la /opt/aws/amazon-cloudwatch-agent/bin/

# 2. Check agent status
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl \
    -a query -m ec2 -c default -s

# Expected output: {"status":"running",...}

# 3. Check configuration
cat /opt/aws/amazon-cloudwatch-agent/etc/config.json

# 4. Check agent logs
sudo tail -f /opt/aws/amazon-cloudwatch-agent/logs/amazon-cloudwatch-agent.log

# 5. Check user-data execution log
sudo cat /var/log/user-data.log

# 6. Verify memory metrics are being collected
free -h

# 7. Verify disk metrics are being collected
df -h
```

### 🔧 Common Issues and Fixes

#### Issue 1: Agent Not Running

**Check status:**
```bash
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl \
    -a query -m ec2 -c default -s
```

**If status is "stopped", start it:**
```bash
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl \
    -a fetch-config -m ec2 -s \
    -c file:/opt/aws/amazon-cloudwatch-agent/etc/config.json
```

**Verify it's running:**
```bash
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl \
    -a query -m ec2 -c default -s
```

#### Issue 2: Agent Running But No Metrics

**Check IAM role:**
```bash
# On EC2 instance
curl http://169.254.169.254/latest/meta-data/iam/security-credentials/

# Should return role name, not 404
```

**Check logs for errors:**
```bash
sudo tail -100 /opt/aws/amazon-cloudwatch-agent/logs/amazon-cloudwatch-agent.log | grep -i error
```

**Restart agent:**
```bash
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl -a stop -m ec2
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl \
    -a fetch-config -m ec2 -s \
    -c file:/opt/aws/amazon-cloudwatch-agent/etc/config.json
```

#### Issue 3: User-Data Script Failed

**Check user-data logs:**
```bash
sudo cat /var/log/user-data.log
# or
sudo cat /var/log/cloud-init-output.log
```

**Manually install if needed:**
```bash
# Download agent
wget https://s3.amazonaws.com/amazoncloudwatch-agent/amazon_linux/amd64/latest/amazon-cloudwatch-agent.rpm

# Install
sudo rpm -U ./amazon-cloudwatch-agent.rpm

# Copy config from your local machine or create it
# Then start agent
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl \
    -a fetch-config -m ec2 -s \
    -c file:/opt/aws/amazon-cloudwatch-agent/etc/config.json
```

### 📊 Verify Metrics in CloudWatch Console

1. **Open CloudWatch Console**:
   ```bash
   open "https://console.aws.amazon.com/cloudwatch/home?region=$(terraform output -raw aws_region)#metricsV2:"
   ```

2. **Navigate to Metrics → CWAgent**

3. **Look for**:
   - `mem_used_percent` - Memory utilization
   - `disk_used_percent` - Disk utilization
   - `diskio_read_bytes` - Disk read operations
   - `diskio_write_bytes` - Disk write operations

4. **Select metrics** and click "Graphed metrics" tab to view

### 🎯 What Metrics Are Collected

#### Memory Metrics (Namespace: CWAgent)
- `mem_used_percent` - **Memory utilization percentage** ⭐
- `mem_used` - Memory used in bytes
- `mem_available` - Available memory
- `mem_total` - Total memory

#### Disk Metrics (Namespace: CWAgent)
- `disk_used_percent` - **Disk utilization percentage** ⭐
- `disk_used` - Disk space used
- `disk_free` - Free disk space
- `diskio_read_bytes` - Bytes read from disk
- `diskio_write_bytes` - Bytes written to disk
- `diskio_reads` - Number of read operations
- `diskio_writes` - Number of write operations

#### CPU Metrics (Namespace: CWAgent)
- `cpu_usage_idle` - CPU idle percentage
- `cpu_usage_iowait` - CPU waiting for I/O

#### Network Metrics (Namespace: CWAgent)
- `tcp_established` - Established TCP connections
- `tcp_time_wait` - TCP connections in TIME_WAIT state

#### Standard EC2 Metrics (Namespace: AWS/EC2)
- `CPUUtilization` - CPU usage percentage
- `NetworkIn` - Network bytes in
- `NetworkOut` - Network bytes out
- `StatusCheckFailed` - Status check failures

### 🔄 Restart Agent (If Needed)

```bash
# Stop agent
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl \
    -a stop -m ec2 -c default

# Start agent
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl \
    -a fetch-config -m ec2 -s \
    -c file:/opt/aws/amazon-cloudwatch-agent/etc/config.json

# Verify status
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl \
    -a query -m ec2 -c default -s
```

### 📝 Configuration File Location

The CloudWatch Agent configuration is stored at:
```
/opt/aws/amazon-cloudwatch-agent/etc/config.json
```

To view it:
```bash
sudo cat /opt/aws/amazon-cloudwatch-agent/etc/config.json
```

Key sections:
- `"metrics"` → Defines what metrics to collect
- `"logs"` → Defines what logs to collect
- `"metrics_collection_interval"` → How often to collect (60 seconds)

### 🚀 Testing Memory and Disk Metrics

#### Test Memory Collection

```bash
# Check current memory
free -h

# Stress memory to see metrics change
sudo stress --vm 1 --vm-bytes 256M --timeout 60s

# Wait 1-2 minutes, then check CloudWatch
```

#### Test Disk Collection

```bash
# Check current disk usage
df -h /

# Create a test file to increase disk usage
sudo dd if=/dev/zero of=/tmp/testfile bs=1M count=100

# Check disk usage again
df -h /

# Clean up
sudo rm /tmp/testfile

# Wait 1-2 minutes, then check CloudWatch
```

### 📈 View Metrics from CLI

```bash
# Get instance ID
INSTANCE_ID=$(terraform output -raw instance_id)

# View memory metrics (last hour)
aws cloudwatch get-metric-statistics \
    --namespace CWAgent \
    --metric-name mem_used_percent \
    --dimensions Name=InstanceId,Value=$INSTANCE_ID \
    --start-time $(date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%S) \
    --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
    --period 300 \
    --statistics Average \
    --query 'Datapoints[*].[Timestamp,Average]' \
    --output table

# View disk metrics (last hour)
aws cloudwatch get-metric-statistics \
    --namespace CWAgent \
    --metric-name disk_used_percent \
    --dimensions Name=InstanceId,Value=$INSTANCE_ID Name=path,Value=/ \
    --start-time $(date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%S) \
    --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
    --period 300 \
    --statistics Average \
    --query 'Datapoints[*].[Timestamp,Average]' \
    --output table
```

### ✅ Success Checklist

After deployment, verify:

- [ ] Wait 10 minutes after instance launch
- [ ] Agent status shows "running"
- [ ] No errors in agent logs
- [ ] IAM role attached with CloudWatchAgentServerPolicy
- [ ] CWAgent namespace exists in CloudWatch
- [ ] mem_used_percent metric appears
- [ ] disk_used_percent metric appears
- [ ] Dashboard shows memory and disk widgets with data
- [ ] Alarms are in OK state (not INSUFFICIENT_DATA)

### 🆘 Still Having Issues?

1. **Check TROUBLESHOOTING.md** for detailed troubleshooting steps
2. **Run verification script**: `sudo bash verify_cloudwatch_agent.sh`
3. **Check agent logs**: `sudo tail -100 /opt/aws/amazon-cloudwatch-agent/logs/amazon-cloudwatch-agent.log`
4. **Verify IAM permissions**: Ensure CloudWatchAgentServerPolicy is attached
5. **Redeploy instance**: `terraform taint aws_instance.monitored && terraform apply`

### 📚 Additional Resources

- [CloudWatch Agent Documentation](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/Install-CloudWatch-Agent.html)
- [CloudWatch Agent Configuration Reference](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/CloudWatch-Agent-Configuration-File-Details.html)
- [Troubleshooting CloudWatch Agent](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/troubleshooting-CloudWatch-Agent.html)

---

## Quick Command Reference

```bash
# Check status
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl -a query -m ec2 -c default -s

# Start agent
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl -a fetch-config -m ec2 -s -c file:/opt/aws/amazon-cloudwatch-agent/etc/config.json

# Stop agent
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl -a stop -m ec2

# View logs
sudo tail -f /opt/aws/amazon-cloudwatch-agent/logs/amazon-cloudwatch-agent.log

# List metrics
aws cloudwatch list-metrics --namespace CWAgent --dimensions Name=InstanceId,Value=<instance-id>
```

---

**The CloudWatch Agent is configured to automatically start on instance launch and collect memory and disk metrics every 60 seconds!** ✅
