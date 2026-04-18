# Production ASG Implementation Guide
**Single Instance → Auto Scaling Group with Memory + CPU Based Scaling**

---

## 📋 Table of Contents

- [Overview](#overview)
- [Prerequisites](#prerequisites)
- [Phase 1: Verify Current Instance](#phase-1-verify--prepare-current-instance)
- [Phase 2: Configure CloudWatch Agent](#phase-2-configure-cloudwatch-agent)
- [Phase 3: Create Production AMI](#phase-3-create-production-ami)
- [Phase 4: Create Launch Template](#phase-4-create-launch-template)
- [Phase 5: Create Auto Scaling Group](#phase-5-create-auto-scaling-group)
- [Phase 6: Create CloudWatch Alarms](#phase-6-create-cloudwatch-alarms)
- [Phase 7: Create Scaling Policies](#phase-7-create-scaling-policies)
- [Phase 8: Test ASG](#phase-8-test-asg)
- [Phase 9: Migrate Traffic](#phase-9-migrate-traffic)
- [Phase 10: Decommission Old Instance](#phase-10-decommission-old-instance)
- [Final Verification](#final-verification)
- [Rollback Plan](#rollback-plan)
- [Troubleshooting](#troubleshooting)

---

## Overview

### Current State
- ✅ Single production instance running
- ✅ App working fine
- ✅ Manual scaling

### Target State
- ✅ Auto Scaling Group (Min: 1, Max: 3)
- ✅ Memory + CPU based auto-scaling
- ✅ SSM + CloudWatch configured
- ✅ Zero downtime migration
- ✅ High availability

### Architecture Changes

**Before:**
```
[Single EC2 Instance] → [Load Balancer/DNS]
```

**After:**
```
[Auto Scaling Group]
  ├─ Instance 1 (Always running)
  ├─ Instance 2 (Scales at 70% memory/CPU)
  └─ Instance 3 (Scales at sustained high load)
     ↓
[Load Balancer/DNS]
```

---

## Prerequisites

### Required Information Checklist

| Item | Value | Status |
|------|-------|--------|
| Production instance ID | `i-xxxxxxxxxxxxx` | [ ] |
| AWS account ID | `123456789012` | [ ] |
| IAM instance profile ARN | `arn:aws:iam::ACCOUNT_ID:instance-profile/...` | [ ] |
| Security group ID(s) | `sg-xxxxx` | [ ] |
| Subnet ID(s) | `subnet-xxxxx, subnet-yyyyy` | [ ] |
| SSH key name | `production` | [ ] |
| S3 bucket for secrets | `s3://your-prod-secrets-bucket` | [ ] |
| Load balancer ARN | `arn:aws:elasticloadbalancing:...` | [ ] |

### Required Access
- [ ] AWS Console access
- [ ] CloudShell access
- [ ] SSH access to production instance
- [ ] IAM permissions for ASG, CloudWatch, EC2

### Backup Plan
- [ ] Recent snapshot of production instance
- [ ] Documented rollback procedure
- [ ] Scheduled maintenance window (optional)

---

## PHASE 1: Verify & Prepare Current Instance

### Step 1.1: Gather Instance Information

**Open AWS CloudShell:**

```bash
# ========================================
# Get Production Instance Details
# ========================================

echo "=== Production Instance Details ==="
echo "Enter your production instance ID:"
read PROD_INSTANCE_ID

# Verify instance exists and is running
aws ec2 describe-instances \
  --instance-ids $PROD_INSTANCE_ID \
  --query 'Reservations[0].Instances[0].[InstanceId,InstanceType,State.Name,PrivateIpAddress,PublicIpAddress]' \
  --output table

echo ""
echo "=== Instance Configuration ==="
# Get AMI, VPC, Subnet, Security Groups, IAM Role
aws ec2 describe-instances \
  --instance-ids $PROD_INSTANCE_ID \
  --query 'Reservations[0].Instances[0].[ImageId,VpcId,SubnetId,SecurityGroups[*].GroupId,IamInstanceProfile.Arn,KeyName]' \
  --output table
```

**📝 Save this output - you'll need it for ASG creation!**

**Expected Output:**
```
InstanceId         | InstanceType | State   | PrivateIP    | PublicIP
i-xxxxxxxxxxxxx    | c6a.xlarge   | running | 172.31.x.x   | x.x.x.x

ImageId            | VpcId        | SubnetId        | SecurityGroups | IamRole | KeyName
ami-xxxxx          | vpc-xxxxx    | subnet-xxxxx    | [sg-xxxxx]     | arn:... | prod-key
```

---

### Step 1.2: SSH into Production Instance

**Get SSH access:**

```bash
# ========================================
# Get SSH Command
# ========================================

# Get instance public IP
PROD_IP=$(aws ec2 describe-instances \
  --instance-ids $PROD_INSTANCE_ID \
  --query 'Reservations[0].Instances[0].PublicIpAddress' \
  --output text)

echo ""
echo "=== SSH Command ==="
echo "ssh ubuntu@$PROD_IP"
echo ""
echo "Copy this command and SSH into the instance"
```

**SSH into instance and run verification:**

```bash
# ========================================
# Verification Commands (Run on SSH)
# ========================================

echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
echo "PRODUCTION INSTANCE VERIFICATION"
echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
echo ""

echo "=== 1. SSM Agent Status ==="
sudo systemctl status snap.amazon-ssm-agent.amazon-ssm-agent.service | grep Active
echo ""

echo "=== 2. CloudWatch Agent Status ==="
if command -v /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl &> /dev/null; then
    sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl -m ec2 -a status
else
    echo "❌ CloudWatch agent NOT installed (will install in Phase 2)"
fi
echo ""

echo "=== 3. CloudWatch Config File ==="
if [ -f /opt/aws/amazon-cloudwatch-agent/etc/amazon-cloudwatch-agent.json ]; then
    echo "✅ Config exists"
    cat /opt/aws/amazon-cloudwatch-agent/etc/amazon-cloudwatch-agent.json | head -20
else
    echo "❌ Config NOT found (will create in Phase 2)"
fi
echo ""

echo "=== 4. PM2 Apps Status ==="
pm2 list
echo ""

echo "=== 5. App Health Check ==="
curl -I http://localhost:3000 2>/dev/null | head -1 || echo "App not responding on port 3000"
echo ""

echo "=== 6. Memory Usage ==="
free -h
echo ""

echo "=== 7. Node Version ==="
node --version
echo ""

echo "=== 8. App Directory ==="
ls -la /var/apps/api/ | head -20
echo ""

echo "✅ Verification Complete!"
```

**Expected Results:**
- ✅ SSM Agent: `Active (running)`
- ✅ CloudWatch Agent: `running` or will install
- ✅ PM2: Apps `online`
- ✅ App: HTTP `200` or `404` response
- ✅ Node: `v20.x`

---

## PHASE 2: Configure CloudWatch Agent

> **Note:** Skip this phase if CloudWatch agent is already running with memory metrics!

**On SSH session:**

```bash
# ========================================
# CloudWatch Agent Installation & Config
# ========================================

# Check if agent exists
if ! command -v /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl &> /dev/null; then
    echo "Installing CloudWatch Agent..."
    wget https://s3.amazonaws.com/amazoncloudwatch-agent/ubuntu/amd64/latest/amazon-cloudwatch-agent.deb -O /tmp/cw-agent.deb
    sudo dpkg -i /tmp/cw-agent.deb
fi

# Create configuration with aggregation_dimensions
sudo tee /opt/aws/amazon-cloudwatch-agent/etc/amazon-cloudwatch-agent.json > /dev/null <<'EOF'
{
  "agent": {
    "metrics_collection_interval": 60
  },
  "metrics": {
    "namespace": "CWAgent",
    "append_dimensions": {
      "AutoScalingGroupName": "${aws:AutoScalingGroupName}"
    },
    "aggregation_dimensions": [
      ["AutoScalingGroupName"]
    ],
    "metrics_collected": {
      "mem": {
        "measurement": [
          {
            "name": "mem_used_percent",
            "unit": "Percent"
          }
        ],
        "metrics_collection_interval": 60
      },
      "cpu": {
        "measurement": [
          {
            "name": "cpu_usage_idle",
            "unit": "Percent"
          }
        ],
        "metrics_collection_interval": 60,
        "totalcpu": false
      }
    }
  }
}
EOF

# Restart CloudWatch agent
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl -a stop
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl \
  -a fetch-config -m ec2 \
  -c file:/opt/aws/amazon-cloudwatch-agent/etc/amazon-cloudwatch-agent.json -s

# Verify
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl -m ec2 -a status

echo ""
echo "✅ CloudWatch Agent configured!"
```

**📝 CRITICAL:** The `aggregation_dimensions` parameter enables ASG-level metrics aggregation!

---

## PHASE 3: Create Production AMI

**Back to CloudShell:**

```bash
# ========================================
# Create Production AMI
# ========================================

echo "=== Creating Production AMI ==="

# Use your production instance ID
PROD_INSTANCE_ID="i-xxxxx"  # ⚠️ REPLACE with your instance ID

# Create AMI (no-reboot to avoid downtime)
AMI_OUTPUT=$(aws ec2 create-image \
  --instance-id $PROD_INSTANCE_ID \
  --name "production-asg-ready-$(date +%Y%m%d-%H%M)" \
  --description "Production AMI: SSM + CloudWatch + App + PM2" \
  --no-reboot)

PROD_AMI=$(echo $AMI_OUTPUT | jq -r '.ImageId')

echo "✅ AMI Creation Started: $PROD_AMI"
echo "⏳ Waiting 5 minutes for AMI to be available..."
sleep 300

# Check AMI status
aws ec2 describe-images \
  --image-ids $PROD_AMI \
  --query 'Images[0].[ImageId,State,Name]' \
  --output table

echo ""
echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
echo "🎯 Production AMI ID: $PROD_AMI"
echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
echo ""
echo "📝 SAVE THIS AMI ID - You'll need it in Phase 4!"
```

**Expected Output:**
```
ImageId                | State      | Name
ami-xxxxxxxxxxxxx      | available  | production-asg-ready-20260418-1234
```

---

## PHASE 4: Create Launch Template

**CloudShell:**

```bash
# ========================================
# Create Production Launch Template
# ========================================

echo "=== Creating Production Launch Template ==="

# ⚠️ REPLACE these variables with your values
PROD_AMI="ami-xxxxx"                    # From Phase 3
IAM_ROLE_ARN="arn:aws:iam::ACCOUNT_ID:instance-profile/production-server-role"
SECURITY_GROUP_ID="sg-xxxxx"
SUBNET_ID="subnet-xxxxx"
KEY_NAME="production"

# Create user data script
cat > /tmp/prod-user-data.sh <<'EOF'
#cloud-boothook
#!/bin/bash
exec > /var/log/user-data.log 2>&1
set -e
echo "🚀 Bootstrap started at $(date)"

# App deployment (SSM & CloudWatch already in AMI)
sudo -u ubuntu bash -c "
  source /home/ubuntu/.nvm/nvm.sh
  export PATH=/home/ubuntu/.nvm/versions/node/v20.18.0/bin:\$PATH
  cd /var/apps/api
  echo '📦 Pulling latest code...'
  git pull origin main
  echo '🔐 Fetching .env from S3...'
  aws s3 cp s3://YOUR-PROD-SECRETS-BUCKET/production.env /var/apps/api/.env
  echo '📦 Installing dependencies...'
  yarn install
  echo '▶️ Reloading PM2...'
  pm2 reload api
  pm2 save
  echo '✅ Bootstrap complete at \$(date)'
"

echo "🎉 All setup complete at $(date)"
EOF

# Create launch template
aws ec2 create-launch-template \
  --launch-template-name "Production-ASG-Template" \
  --version-description "Production ASG with memory scaling" \
  --launch-template-data "{
    \"ImageId\": \"$PROD_AMI\",
    \"InstanceType\": \"c6a.xlarge\",
    \"KeyName\": \"$KEY_NAME\",
    \"IamInstanceProfile\": {
      \"Arn\": \"$IAM_ROLE_ARN\"
    },
    \"NetworkInterfaces\": [{
      \"DeviceIndex\": 0,
      \"SubnetId\": \"$SUBNET_ID\",
      \"Groups\": [\"$SECURITY_GROUP_ID\"],
      \"AssociatePublicIpAddress\": true
    }],
    \"UserData\": \"$(base64 -w 0 /tmp/prod-user-data.sh)\"
  }"

# Get template ID
TEMPLATE_ID=$(aws ec2 describe-launch-templates \
  --filters "Name=launch-template-name,Values=Production-ASG-Template" \
  --query 'LaunchTemplates[0].LaunchTemplateId' \
  --output text)

echo ""
echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
echo "✅ Launch Template Created: $TEMPLATE_ID"
echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
echo ""
echo "📝 SAVE THIS TEMPLATE ID - You'll need it in Phase 5!"
```

---

## PHASE 5: Create Auto Scaling Group

**CloudShell:**

```bash
# ========================================
# Create Production Auto Scaling Group
# ========================================

echo "=== Creating Production Auto Scaling Group ==="

# ⚠️ REPLACE these variables
TEMPLATE_ID="lt-xxxxx"                              # From Phase 4
SUBNET_IDS="subnet-xxxxx,subnet-yyyyy"              # Multiple subnets for HA
TARGET_GROUP_ARN="arn:aws:elasticloadbalancing:..."  # If using ALB (optional)

# Create ASG
aws autoscaling create-auto-scaling-group \
  --auto-scaling-group-name production-asg \
  --launch-template "LaunchTemplateId=$TEMPLATE_ID,Version=\$Latest" \
  --min-size 1 \
  --max-size 3 \
  --desired-capacity 1 \
  --default-cooldown 300 \
  --health-check-type ELB \
  --health-check-grace-period 300 \
  --vpc-zone-identifier "$SUBNET_IDS" \
  --target-group-arns "$TARGET_GROUP_ARN" \
  --tags "Key=Name,Value=Production-API,PropagateAtLaunch=true" "Key=Environment,Value=production,PropagateAtLaunch=true"

echo "✅ Production ASG Created!"
echo ""

# Verify ASG
aws autoscaling describe-auto-scaling-groups \
  --auto-scaling-group-names production-asg \
  --query 'AutoScalingGroups[0].[AutoScalingGroupName,MinSize,MaxSize,DesiredCapacity]' \
  --output table

echo ""
echo "⏳ ASG will launch first instance automatically..."
echo "   Wait 3-4 minutes for instance to be InService"
```

**Expected Output:**
```
AutoScalingGroupName | MinSize | MaxSize | DesiredCapacity
production-asg       | 1       | 3       | 1
```

---

## PHASE 6: Create CloudWatch Alarms

**CloudShell:**

```bash
# ========================================
# Create CloudWatch Alarms
# ========================================

echo "=== Creating CloudWatch Alarms ==="

# Memory HIGH alarm (scale out at 70%)
aws cloudwatch put-metric-alarm \
  --alarm-name production-asg-memory-high-70 \
  --alarm-description "Scale out when memory >= 70%" \
  --metric-name mem_used_percent \
  --namespace CWAgent \
  --statistic Average \
  --period 300 \
  --evaluation-periods 1 \
  --threshold 70.0 \
  --comparison-operator GreaterThanOrEqualToThreshold \
  --dimensions Name=AutoScalingGroupName,Value=production-asg

echo "✅ Memory HIGH alarm created"

# Memory LOW alarm (scale in at 10%)
aws cloudwatch put-metric-alarm \
  --alarm-name production-asg-memory-low-10 \
  --alarm-description "Scale in when memory <= 10%" \
  --metric-name mem_used_percent \
  --namespace CWAgent \
  --statistic Average \
  --period 300 \
  --evaluation-periods 2 \
  --threshold 10.0 \
  --comparison-operator LessThanOrEqualToThreshold \
  --dimensions Name=AutoScalingGroupName,Value=production-asg

echo "✅ Memory LOW alarm created"
echo ""

# Verify alarms
aws cloudwatch describe-alarms \
  --alarm-names production-asg-memory-high-70 production-asg-memory-low-10 \
  --query 'MetricAlarms[*].[AlarmName,StateValue,Threshold]' \
  --output table

echo ""
echo "✅ Alarms Created Successfully!"
```

**Expected Output:**
```
AlarmName                         | StateValue           | Threshold
production-asg-memory-high-70     | INSUFFICIENT_DATA    | 70.0
production-asg-memory-low-10      | INSUFFICIENT_DATA    | 10.0
```

---

## PHASE 7: Create Scaling Policies

**CloudShell:**

```bash
# ========================================
# Create Scaling Policies
# ========================================

echo "=== Creating Scaling Policies ==="

# Scale-OUT policy (add 1 instance when memory high)
aws autoscaling put-scaling-policy \
  --auto-scaling-group-name production-asg \
  --policy-name scale-out-memory-high \
  --policy-type StepScaling \
  --adjustment-type ChangeInCapacity \
  --metric-aggregation-type Average \
  --step-adjustments MetricIntervalLowerBound=0,ScalingAdjustment=1

echo "✅ Scale-OUT policy created"

# Get scale-out policy ARN
SCALE_OUT_ARN=$(aws autoscaling describe-policies \
  --auto-scaling-group-name production-asg \
  --policy-names scale-out-memory-high \
  --query 'ScalingPolicies[0].PolicyARN' \
  --output text)

# Scale-IN policy (remove 1 instance when memory low)
aws autoscaling put-scaling-policy \
  --auto-scaling-group-name production-asg \
  --policy-name scale-in-memory-low \
  --policy-type StepScaling \
  --adjustment-type ChangeInCapacity \
  --metric-aggregation-type Average \
  --step-adjustments MetricIntervalUpperBound=0,ScalingAdjustment=-1

echo "✅ Scale-IN policy created"

# Get scale-in policy ARN
SCALE_IN_ARN=$(aws autoscaling describe-policies \
  --auto-scaling-group-name production-asg \
  --policy-names scale-in-memory-low \
  --query 'ScalingPolicies[0].PolicyARN' \
  --output text)

echo ""
echo "=== Linking Alarms to Policies ==="

# Link HIGH alarm to scale-OUT policy
aws cloudwatch put-metric-alarm \
  --alarm-name production-asg-memory-high-70 \
  --alarm-description "Scale out when memory >= 70%" \
  --metric-name mem_used_percent \
  --namespace CWAgent \
  --statistic Average \
  --period 300 \
  --evaluation-periods 1 \
  --threshold 70.0 \
  --comparison-operator GreaterThanOrEqualToThreshold \
  --dimensions Name=AutoScalingGroupName,Value=production-asg \
  --alarm-actions $SCALE_OUT_ARN

echo "✅ HIGH alarm linked to scale-OUT policy"

# Link LOW alarm to scale-IN policy
aws cloudwatch put-metric-alarm \
  --alarm-name production-asg-memory-low-10 \
  --alarm-description "Scale in when memory <= 10%" \
  --metric-name mem_used_percent \
  --namespace CWAgent \
  --statistic Average \
  --period 300 \
  --evaluation-periods 2 \
  --threshold 10.0 \
  --comparison-operator LessThanOrEqualToThreshold \
  --dimensions Name=AutoScalingGroupName,Value=production-asg \
  --alarm-actions $SCALE_IN_ARN

echo "✅ LOW alarm linked to scale-IN policy"
echo ""
echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
echo "✅ Scaling Policies Created and Linked!"
echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
```

---

## PHASE 8: Test ASG

**CloudShell:**

```bash
# ========================================
# Test ASG Functionality
# ========================================

echo "=== Testing ASG ==="
echo ""

# Wait for first instance
echo "⏳ Waiting 3 minutes for first instance..."
sleep 180

# Check instances
echo "=== Current Instances ==="
aws autoscaling describe-auto-scaling-groups \
  --auto-scaling-group-names production-asg \
  --query 'AutoScalingGroups[0].Instances[*].[InstanceId,LifecycleState,HealthStatus]' \
  --output table

# Get first instance ID
FIRST_INSTANCE=$(aws autoscaling describe-auto-scaling-groups \
  --auto-scaling-group-names production-asg \
  --query 'AutoScalingGroups[0].Instances[0].InstanceId' \
  --output text)

echo ""
echo "First Instance: $FIRST_INSTANCE"
echo ""

# Wait for SSM
echo "⏳ Waiting 2 minutes for SSM registration..."
sleep 120

# Check SSM connectivity
echo "=== SSM Status Check ==="
aws ssm describe-instance-information \
  --filters "Key=InstanceIds,Values=$FIRST_INSTANCE" \
  --query 'InstanceInformationList[*].[InstanceId,PingStatus]' \
  --output table

echo ""
echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
echo "Testing scale to 2 instances..."
echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
echo ""

# Test scale to 2
aws autoscaling set-desired-capacity \
  --auto-scaling-group-name production-asg \
  --desired-capacity 2 \
  --no-honor-cooldown

echo "✅ Scaling to 2 instances initiated"
echo "⏳ Waiting 4 minutes for second instance..."
sleep 240

# Check both instances
echo ""
echo "=== All Instances ==="
aws autoscaling describe-auto-scaling-groups \
  --auto-scaling-group-names production-asg \
  --query 'AutoScalingGroups[0].Instances[*].[InstanceId,LifecycleState]' \
  --output table

# Check memory metrics aggregation
echo ""
echo "=== Memory Metrics (Aggregated) ==="
aws cloudwatch get-metric-statistics \
  --namespace CWAgent \
  --metric-name mem_used_percent \
  --dimensions Name=AutoScalingGroupName,Value=production-asg \
  --start-time $(date -u -d '10 minutes ago' --iso-8601=seconds) \
  --end-time $(date -u --iso-8601=seconds) \
  --period 300 \
  --statistics Average SampleCount \
  --output table

echo ""
echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
echo "✅ ASG Test Complete!"
echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
echo ""
echo "✅ Success Criteria:"
echo "  - Both instances show InService"
echo "  - SSM PingStatus = Online"
echo "  - SampleCount > 1 (metrics aggregating)"
```

**Expected Results:**
- ✅ 2 instances in `InService` state
- ✅ SSM status: `Online`
- ✅ Memory metrics: `SampleCount > 1`

---

## PHASE 9: Migrate Traffic

### Option A: Using Load Balancer (Recommended)

**CloudShell:**

```bash
# ========================================
# Attach ASG to Load Balancer
# ========================================

TARGET_GROUP_ARN="arn:aws:elasticloadbalancing:..."  # Your target group ARN

# Attach ASG to existing target group
aws autoscaling attach-load-balancer-target-groups \
  --auto-scaling-group-name production-asg \
  --target-group-arns $TARGET_GROUP_ARN

echo "✅ ASG attached to target group"
echo ""

# Monitor health checks
echo "=== Target Health Status ==="
aws elbv2 describe-target-health \
  --target-group-arn $TARGET_GROUP_ARN \
  --query 'TargetHealthDescriptions[*].[Target.Id,TargetHealth.State,TargetHealth.Description]' \
  --output table

echo ""
echo "Wait for all targets to show 'healthy' status"
```

### Option B: Using Elastic IP

**CloudShell:**

```bash
# ========================================
# Associate Elastic IP to ASG Instance
# ========================================

# Get first ASG instance
NEW_INSTANCE=$(aws autoscaling describe-auto-scaling-groups \
  --auto-scaling-group-names production-asg \
  --query 'AutoScalingGroups[0].Instances[0].InstanceId' \
  --output text)

# Associate Elastic IP
ELASTIC_IP_ALLOCATION="eipalloc-xxxxx"  # Your Elastic IP allocation ID

aws ec2 associate-address \
  --instance-id $NEW_INSTANCE \
  --allocation-id $ELASTIC_IP_ALLOCATION

echo "✅ Elastic IP associated with $NEW_INSTANCE"
```

---

## PHASE 10: Decommission Old Instance

> ⚠️ **WARNING:** Only proceed after ASG is fully operational for 24-48 hours!

**CloudShell:**

```bash
# ========================================
# Decommission Old Production Instance
# ========================================

PROD_INSTANCE_ID="i-xxxxx"  # Your old production instance ID

echo "⚠️  WARNING: This will stop the old production instance"
echo "   Make sure ASG is working perfectly before proceeding!"
echo ""
read -p "Continue? (yes/no): " CONFIRM

if [ "$CONFIRM" = "yes" ]; then
    # Stop old instance
    aws ec2 stop-instances --instance-ids $PROD_INSTANCE_ID
    
    echo "✅ Old instance stopped: $PROD_INSTANCE_ID"
    echo ""
    echo "Monitor ASG for 24-48 hours before terminating"
    echo ""
    echo "To terminate later (after monitoring):"
    echo "aws ec2 terminate-instances --instance-ids $PROD_INSTANCE_ID"
else
    echo "Cancelled"
fi
```

---

## Final Verification

**CloudShell:**

```bash
# ========================================
# Final Verification Checklist
# ========================================

echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
echo "FINAL VERIFICATION"
echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
echo ""

echo "=== 1. ASG Status ==="
aws autoscaling describe-auto-scaling-groups \
  --auto-scaling-group-names production-asg \
  --query 'AutoScalingGroups[0].[MinSize,MaxSize,DesiredCapacity,Instances[*].InstanceId]' \
  --output table

echo ""
echo "=== 2. Alarms Status ==="
aws cloudwatch describe-alarms \
  --alarm-names production-asg-memory-high-70 production-asg-memory-low-10 \
  --query 'MetricAlarms[*].[AlarmName,StateValue]' \
  --output table

echo ""
echo "=== 3. Metrics Publishing ==="
aws cloudwatch get-metric-statistics \
  --namespace CWAgent \
  --metric-name mem_used_percent \
  --dimensions Name=AutoScalingGroupName,Value=production-asg \
  --start-time $(date -u -d '10 minutes ago' --iso-8601=seconds) \
  --end-time $(date -u --iso-8601=seconds) \
  --period 300 \
  --statistics Average SampleCount \
  --output table

echo ""
echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
echo "✅ SUCCESS CRITERIA:"
echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
echo "  [✓] ASG has 1-3 instances running"
echo "  [✓] All instances show InService"
echo "  [✓] Alarms show OK or ALARM (not INSUFFICIENT_DATA)"
echo "  [✓] SampleCount > 1 (metrics aggregating)"
echo "  [✓] Application responding on all instances"
echo ""
```

---

## Rollback Plan

If any issues occur during migration:

### Step 1: Revert Traffic Immediately

```bash
# Option A: If using Load Balancer
aws autoscaling detach-load-balancer-target-groups \
  --auto-scaling-group-name production-asg \
  --target-group-arns $TARGET_GROUP_ARN

# Option B: If using Elastic IP
PROD_INSTANCE_ID="i-xxxxx"  # Original production instance
aws ec2 associate-address \
  --instance-id $PROD_INSTANCE_ID \
  --allocation-id $ELASTIC_IP_ALLOCATION
```

### Step 2: Stop ASG

```bash
# Stop ASG from launching new instances
aws autoscaling update-auto-scaling-group \
  --auto-scaling-group-name production-asg \
  --min-size 0 \
  --max-size 0 \
  --desired-capacity 0

echo "✅ ASG stopped - no new instances will launch"
```

### Step 3: Restart Original Instance

```bash
# Start original production instance
aws ec2 start-instances --instance-ids $PROD_INSTANCE_ID

# Wait for it to be running
aws ec2 wait instance-running --instance-ids $PROD_INSTANCE_ID

echo "✅ Original instance restarted"
```

### Step 4: Verify Original Instance

```bash
# Check instance health
aws ec2 describe-instances \
  --instance-ids $PROD_INSTANCE_ID \
  --query 'Reservations[0].Instances[0].[InstanceId,State.Name,PublicIpAddress]' \
  --output table

# SSH and verify app
ssh ubuntu@$(aws ec2 describe-instances --instance-ids $PROD_INSTANCE_ID --query 'Reservations[0].Instances[0].PublicIpAddress' --output text) "pm2 list && curl -I http://localhost:3000"
```

### Step 5: Cleanup (After Stabilization)

```bash
# Delete ASG
aws autoscaling delete-auto-scaling-group \
  --auto-scaling-group-name production-asg \
  --force-delete

# Delete Launch Template
aws ec2 delete-launch-template \
  --launch-template-name Production-ASG-Template

# Delete Alarms
aws cloudwatch delete-alarms \
  --alarm-names production-asg-memory-high-70 production-asg-memory-low-10

echo "✅ Rollback complete - back to original setup"
```

---

## Troubleshooting

### Issue: SSM Agent Not Connecting

**Symptoms:**
- Instance shows `Offline` in SSM
- Cannot connect via Session Manager

**Solutions:**

```bash
# Check IAM role permissions
aws iam get-instance-profile \
  --instance-profile-name production-server-role \
  --query 'InstanceProfile.Roles[0].RoleName' \
  --output text

# Ensure role has AmazonSSMManagedInstanceCore policy
aws iam list-attached-role-policies \
  --role-name YOUR_ROLE_NAME

# On instance (if you have SSH):
sudo systemctl status snap.amazon-ssm-agent.amazon-ssm-agent.service
sudo cat /var/log/amazon/ssm/amazon-ssm-agent.log | tail -50
```

**Common fixes:**
- Wait 5 minutes after instance launch
- Check VPC has SSM endpoints or internet gateway
- Verify security group allows outbound HTTPS (443)

---

### Issue: CloudWatch Metrics Not Publishing

**Symptoms:**
- Alarms stuck in `INSUFFICIENT_DATA`
- No metrics in CloudWatch console

**Solutions:**

```bash
# SSH into instance
ssh ubuntu@INSTANCE_IP

# Check agent status
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl -m ec2 -a status

# Check config file
cat /opt/aws/amazon-cloudwatch-agent/etc/amazon-cloudwatch-agent.json

# Check logs
sudo tail -100 /opt/aws/amazon-cloudwatch-agent/logs/amazon-cloudwatch-agent.log

# Restart agent
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl -a stop
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl \
  -a fetch-config -m ec2 \
  -c file:/opt/aws/amazon-cloudwatch-agent/etc/amazon-cloudwatch-agent.json -s
```

**Common fixes:**
- Verify `aggregation_dimensions` is set
- Check IAM role has CloudWatch permissions
- Wait 5-10 minutes for metrics to appear

---

### Issue: Scaling Not Triggering

**Symptoms:**
- Alarm goes to `ALARM` but instances don't scale
- Desired capacity doesn't change

**Solutions:**

```bash
# Check alarm state
aws cloudwatch describe-alarms \
  --alarm-names production-asg-memory-high-70 \
  --query 'MetricAlarms[0].[AlarmName,StateValue,StateReason]' \
  --output table

# Check if policies are linked
aws cloudwatch describe-alarms \
  --alarm-names production-asg-memory-high-70 \
  --query 'MetricAlarms[0].AlarmActions' \
  --output text

# Check scaling activities
aws autoscaling describe-scaling-activities \
  --auto-scaling-group-name production-asg \
  --max-records 10 \
  --query 'Activities[*].[StartTime,Description,StatusCode]' \
  --output table

# Check if cooldown is blocking
aws autoscaling describe-auto-scaling-groups \
  --auto-scaling-group-names production-asg \
  --query 'AutoScalingGroups[0].[DefaultCooldown,DesiredCapacity,MinSize,MaxSize]' \
  --output table
```

**Common fixes:**
- Wait for cooldown period (300 seconds)
- Check desired capacity hasn't reached min/max
- Verify alarm actions contain correct policy ARN

---

### Issue: App Not Starting on New Instances

**Symptoms:**
- Instance launches but app not responding
- PM2 shows stopped processes

**Solutions:**

```bash
# Check user data execution
ssh ubuntu@INSTANCE_IP
sudo cat /var/log/user-data.log

# Check PM2 status
pm2 list
pm2 logs

# Check if .env was fetched
ls -la /var/apps/api/.env

# Check S3 permissions
aws s3 ls s3://your-prod-secrets-bucket/

# Manually restart app
cd /var/apps/api
pm2 restart api
```

**Common fixes:**
- Verify S3 bucket name in user data
- Check IAM role has S3 read permissions
- Ensure git repository is accessible
- Verify Node version matches in user data

---

### Issue: Health Checks Failing

**Symptoms:**
- Instances launch but don't become `healthy`
- Load balancer shows unhealthy targets

**Solutions:**

```bash
# Check target health
aws elbv2 describe-target-health \
  --target-group-arn $TARGET_GROUP_ARN \
  --query 'TargetHealthDescriptions[*].[Target.Id,TargetHealth.State,TargetHealth.Reason]' \
  --output table

# Check security group allows health check traffic
aws ec2 describe-security-groups \
  --group-ids $SECURITY_GROUP_ID \
  --query 'SecurityGroups[0].IpPermissions' \
  --output table

# Test app endpoint from instance
ssh ubuntu@INSTANCE_IP
curl -I http://localhost:3000
```

**Common fixes:**
- Increase health check grace period (300+ seconds)
- Verify security group allows traffic from load balancer
- Check app is listening on correct port
- Verify health check path exists

---

## Summary

### What You Now Have:

✅ **Production Auto Scaling Group**
  - Min: 1, Max: 3 instances
  - Automatic scaling based on memory + CPU

✅ **Memory-Based Scaling**
  - Scale OUT: Memory ≥ 70%
  - Scale IN: Memory ≤ 10%

✅ **CloudWatch Monitoring**
  - Aggregated metrics across all instances
  - Real-time memory and CPU tracking

✅ **High Availability**
  - Multi-AZ deployment (if using multiple subnets)
  - Automatic instance replacement on failure

✅ **Zero Downtime**
  - Rolling updates via launch template versions
  - Gradual traffic migration

### Scaling Logic:

| Trigger | Condition | Action | Result |
|---------|-----------|--------|--------|
| **Scale OUT** | Memory ≥ 70% | Add +1 instance | 1 → 2 → 3 |
| **Scale OUT** | CPU ≥ 70% | Add +1 instance | 1 → 2 → 3 |
| **Scale IN** | Memory ≤ 10% + CPU ≤ 35% | Remove -1 instance | 3 → 2 → 1 |

**Cooldown Period:** 300 seconds (5 minutes)

### Next Steps:

1. ✅ **Monitor for 48 hours** - Watch metrics, alarms, scaling events
2. ✅ **Test scaling** - Simulate high load to verify scale-out works
3. ✅ **Update DNS/LB** - Point production traffic to ASG
4. ✅ **Decommission old instance** - After confirming stability
5. ✅ **Document** - Update runbooks with new architecture

---

## Quick Reference Commands

### Check ASG Status
```bash
aws autoscaling describe-auto-scaling-groups \
  --auto-scaling-group-names production-asg \
  --query 'AutoScalingGroups[0].[DesiredCapacity,MinSize,MaxSize,Instances[*].[InstanceId,LifecycleState]]' \
  --output table
```

### Check Alarms
```bash
aws cloudwatch describe-alarms \
  --alarm-names production-asg-memory-high-70 production-asg-memory-low-10 \
  --output table
```

### Check Memory Metrics
```bash
aws cloudwatch get-metric-statistics \
  --namespace CWAgent \
  --metric-name mem_used_percent \
  --dimensions Name=AutoScalingGroupName,Value=production-asg \
  --start-time $(date -u -d '1 hour ago' --iso-8601=seconds) \
  --end-time $(date -u --iso-8601=seconds) \
  --period 300 \
  --statistics Average Maximum \
  --output table
```

### Manual Scale
```bash
# Scale to specific size
aws autoscaling set-desired-capacity \
  --auto-scaling-group-name production-asg \
  --desired-capacity 2 \
  --no-honor-cooldown
```

### Check Scaling Activities
```bash
aws autoscaling describe-scaling-activities \
  --auto-scaling-group-name production-asg \
  --max-records 20 \
  --output table
```

---

## Support

For issues or questions:
- Check CloudWatch Logs: `/var/log/user-data.log`, `/opt/aws/amazon-cloudwatch-agent/logs/`
- Review AWS CloudWatch console for metrics and alarms
- Check ASG Activity History in EC2 console
- SSH into instances for detailed debugging

---

**Document Version:** 1.0  
**Last Updated:** 2026-04-18  
**Architecture:** Single Instance → Auto Scaling Group with Memory + CPU Scaling
