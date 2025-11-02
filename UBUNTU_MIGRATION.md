# Ubuntu Migration Notes

## Changes Made

The infrastructure has been updated to use **Ubuntu 22.04 LTS** instead of Amazon Linux 2.

### Key Changes:

#### 1. AMI Selection (main.tf)
- **Old**: Amazon Linux 2 (`amzn2-ami-hvm-*-x86_64-gp2`)
- **New**: Ubuntu 22.04 LTS (`ubuntu-jammy-22.04-amd64-server-*`)
- **Owner**: Canonical (099720109477)

#### 2. Package Manager
- **Old**: `yum` (Red Hat/Amazon Linux)
- **New**: `apt-get` (Debian/Ubuntu)

#### 3. CloudWatch Agent Package
- **Old**: `.rpm` package for Amazon Linux
- **New**: `.deb` package for Ubuntu

#### 4. Default User
- **Old**: `ec2-user`
- **New**: `ubuntu`

### Updated Files:

1. **main.tf**
   - Changed AMI data source from `amazon_linux_2` to `ubuntu`
   - Updated filter to use Ubuntu 22.04 LTS images

2. **scripts/user_data.sh**
   - Changed `yum` to `apt-get`
   - Changed package format from `.rpm` to `.deb`
   - Updated CloudWatch Agent download URL

3. **scripts/test_cpu_stress.sh**
   - Updated stress tool installation command

4. **scripts/test_memory_stress.sh**
   - Updated stress tool installation command

### SSH Access:

When connecting to the instance, use the `ubuntu` user:

```bash
# Old (Amazon Linux 2)
ssh -i your-key.pem ec2-user@<instance-ip>

# New (Ubuntu)
ssh -i your-key.pem ubuntu@<instance-ip>
```

### SSM Session Manager:

SSM Session Manager works the same way:
```bash
aws ssm start-session --target <instance-id>
```

### CloudWatch Agent:

The CloudWatch Agent configuration remains the same. The agent works identically on Ubuntu as it does on Amazon Linux 2.

### Benefits of Ubuntu:

1. **Longer LTS Support**: Ubuntu 22.04 LTS is supported until April 2027
2. **Wider Package Availability**: More packages available in apt repositories
3. **Better Community Support**: Larger community and more documentation
4. **Stress Tool**: Available directly in standard repositories

### Deployment:

To deploy with Ubuntu:

```bash
export AWS_PROFILE=personal_new

# Build Lambda package
./scripts/build_lambda.sh

# Deploy infrastructure
./scripts/deploy.sh
```

The deployment process remains the same. The instance will automatically:
1. Install CloudWatch Agent
2. Configure metrics collection
3. Start publishing memory and disk metrics
4. Install stress tool for testing

### Verification:

After deployment, verify the same way:

```bash
export AWS_PROFILE=personal_new
./verify_agent_from_local.sh
```

### Testing:

Test commands remain the same:

```bash
# SSH to instance
ssh -i your-key.pem ubuntu@$(terraform output -raw instance_public_ip)

# Test CPU
sudo stress --cpu $(nproc) --timeout 300s

# Test Memory
sudo stress --vm 1 --vm-bytes 512M --timeout 300s
```

---

**All functionality remains the same - only the underlying OS has changed from Amazon Linux 2 to Ubuntu 22.04 LTS.**
