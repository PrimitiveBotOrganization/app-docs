# Incident Response Runbook
## ShopSmart E-commerce Platform

### Document Information
- **Version**: 3.2
- **Last Updated**: January 2024
- **Owner**: Platform Engineering Team
- **Classification**: Internal Use Only

---

## Table of Contents
1. [Incident Classification](#incident-classification)
2. [Lambda Function Failures](#lambda-function-failures)
3. [Database Performance Issues](#database-performance-issues)
4. [Application Load Balancer Issues](#application-load-balancer-issues)
5. [S3 Access Issues](#s3-access-issues)
6. [Auto Scaling Failures](#auto-scaling-failures)
7. [Cache Performance Issues](#cache-performance-issues)
8. [Network Connectivity Issues](#network-connectivity-issues)
9. [Security Incidents](#security-incidents)
10. [Post-Incident Procedures](#post-incident-procedures)

---

## Incident Classification

### Severity Levels

| Severity | Description | Response Time | Examples |
|----------|-------------|---------------|----------|
| **P0 - Critical** | Complete service outage | 15 minutes | Site down, payment processing failed |
| **P1 - High** | Major feature unavailable | 1 hour | Search not working, user login issues |
| **P2 - Medium** | Performance degradation | 4 hours | Slow page loads, intermittent errors |
| **P3 - Low** | Minor issues | 24 hours | Cosmetic bugs, non-critical features |

### Escalation Matrix

| Role | P0 | P1 | P2 | P3 |
|------|----|----|----|----|
| On-call Engineer | Immediate | Immediate | Normal hours | Normal hours |
| Team Lead | 30 min | 2 hours | Next day | Weekly review |
| Engineering Manager | 1 hour | 4 hours | Next day | Weekly review |
| VP Engineering | 2 hours | Next day | Weekly review | Monthly review |

---

## Lambda Function Failures

### Pattern: Runtime Errors and Timeouts

**Historical Context**: Based on 47 incidents over 18 months, Lambda failures account for 23% of P0/P1 incidents.

#### Common Symptoms
- CloudWatch logs showing "AccessDenied" or "UnauthorizedOperation" errors
- Lambda function completing but returning permission errors
- User reports of birthday notifications not being sent
- SES bounce notifications due to Lambda permission failures

#### Incident Examples from History

**Ticket #INC-2023-1247**: AmazonQBirthdayDemoLambda permission denied errors
- **Root Cause**: IAM role missing DynamoDB permissions after policy update
- **Resolution Time**: 18 minutes
- **Impact**: 100% of birthday notification requests failed

**Ticket #INC-2023-0892**: AmazonQBirthdayDemoLambda S3 access denied
- **Root Cause**: S3 bucket policy changed, Lambda role lost read access
- **Resolution Time**: 12 minutes
- **Impact**: Birthday email templates couldn't be retrieved

#### Standard Resolution Playbook

##### Step 1: Immediate Assessment (0-5 minutes)
```bash
# Check function status and recent invocations
aws lambda get-function --function-name AmazonQBirthdayDemoLambda
aws logs filter-log-events --log-group-name /aws/lambda/AmazonQBirthdayDemoLambda --start-time $(date -d '10 minutes ago' +%s)000 --filter-pattern "ERROR"
```

**Expected Outputs to Look For:**
- Error patterns: `AccessDenied`, `UnauthorizedOperation`, `Forbidden`
- Permission-related errors: `User: arn:aws:sts::123456789012:assumed-role/lambda-execution-role is not authorized`
- Resource access failures: `Access Denied` for DynamoDB, S3, or other AWS services

##### Step 2: IAM Role Analysis (5-10 minutes)
```bash
# Check Lambda execution role
ROLE_ARN=$(aws lambda get-function --function-name AmazonQBirthdayDemoLambda --query 'Configuration.Role' --output text)
ROLE_NAME=$(echo $ROLE_ARN | cut -d'/' -f2)

# Get attached policies
aws iam list-attached-role-policies --role-name $ROLE_NAME
aws iam list-role-policies --role-name $ROLE_NAME

# Check recent policy changes
aws iam get-role --role-name $ROLE_NAME --query 'Role.AssumeRolePolicyDocument'
```

**Success Pattern**: 78% of Lambda permission incidents resolved by IAM policy fixes

##### Step 3: Resource Access Testing (10-15 minutes)
```bash
# Test DynamoDB access (common birthday demo requirement)
aws dynamodb describe-table --table-name birthday-users --region us-east-1

# Test S3 bucket access (for email templates)
aws s3 ls s3://birthday-email-templates/ --region us-east-1

# Check if Lambda can assume required roles
aws sts get-caller-identity
```

**Historical Fix**: Ticket #INC-2023-1456 - DynamoDB table policy removed Lambda role access after security audit

##### Step 4: Permission Policy Fix (15-20 minutes)
```bash
# Create emergency policy for common birthday demo permissions
cat > emergency-lambda-policy.json << EOF
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "dynamodb:GetItem",
                "dynamodb:PutItem",
                "dynamodb:Query",
                "dynamodb:Scan"
            ],
            "Resource": "arn:aws:dynamodb:us-east-1:*:table/birthday-users"
        },
        {
            "Effect": "Allow",
            "Action": [
                "s3:GetObject"
            ],
            "Resource": "arn:aws:s3:::birthday-email-templates/*"
        },
        {
            "Effect": "Allow",
            "Action": [
                "ses:SendEmail",
                "ses:SendRawEmail"
            ],
            "Resource": "*"
        }
    ]
}
EOF

# Attach emergency policy
aws iam put-role-policy --role-name $ROLE_NAME --policy-name EmergencyBirthdayAccess --policy-document file://emergency-lambda-policy.json
```

##### Step 5: Validation and Monitoring (20-25 minutes)
```bash
# Test Lambda function manually
aws lambda invoke --function-name AmazonQBirthdayDemoLambda --payload '{"test": "true"}' response.json
cat response.json

# Monitor for continued errors
aws logs filter-log-events --log-group-name /aws/lambda/AmazonQBirthdayDemoLambda --start-time $(date +%s)000 --filter-pattern "ERROR"

# If still failing, rollback to previous version
PREVIOUS_VERSION=$(aws lambda list-versions-by-function --function-name AmazonQBirthdayDemoLambda --query 'Versions[-2].Version' --output text)
aws lambda update-alias --function-name AmazonQBirthdayDemoLambda --name LIVE --function-version $PREVIOUS_VERSION
```

**Success Metrics**: Average resolution time for permission issues: 18 minutes using this playbook

---

## Database Performance Issues

### Pattern: Connection Pool Exhaustion and Query Timeouts

**Historical Context**: 31 incidents, typically during traffic spikes or after deployments.

#### Incident Examples from History

**Ticket #INC-2023-2156**: RDS connection pool exhaustion during Black Friday
- **Root Cause**: Application not releasing connections properly
- **Resolution**: Implemented connection pooling with PgBouncer
- **Prevention**: Added connection monitoring alerts

**Ticket #INC-2023-1789**: Slow query causing cascade failures
- **Root Cause**: Missing index on frequently queried column
- **Resolution**: Added index, optimized query
- **Impact**: 40% of API calls timing out

#### Standard Resolution Playbook

##### Step 1: Connection Assessment (0-3 minutes)
```sql
-- Check active connections
SELECT count(*) as active_connections, state 
FROM pg_stat_activity 
WHERE state IS NOT NULL 
GROUP BY state;

-- Identify long-running queries
SELECT pid, now() - pg_stat_activity.query_start AS duration, query 
FROM pg_stat_activity 
WHERE (now() - pg_stat_activity.query_start) > interval '5 minutes';
```

##### Step 2: Quick Connection Relief (3-8 minutes)
```sql
-- Kill problematic long-running queries (use carefully)
SELECT pg_terminate_backend(pid) 
FROM pg_stat_activity 
WHERE (now() - pg_stat_activity.query_start) > interval '10 minutes'
AND state = 'active';
```

```bash
# Scale up RDS instance temporarily
aws rds modify-db-instance --db-instance-identifier shopsmart-prod-db --db-instance-class db.r5.2xlarge --apply-immediately
```

##### Step 3: Application-Level Fixes (8-15 minutes)
```bash
# Restart application servers to reset connection pools
aws autoscaling set-desired-capacity --auto-scaling-group-name shopsmart-prod-asg --desired-capacity 0
sleep 30
aws autoscaling set-desired-capacity --auto-scaling-group-name shopsmart-prod-asg --desired-capacity 3
```

**Historical Success**: Ticket #INC-2023-0445 - Connection pool reset resolved 89% of similar incidents

##### Step 4: Read Replica Failover (15-20 minutes)
```bash
# Promote read replica if master is overwhelmed
aws rds promote-read-replica --db-instance-identifier shopsmart-prod-db-replica
```

**Recovery Pattern**: Used in 12 incidents, average recovery time 18 minutes

---

## Application Load Balancer Issues

### Pattern: Target Health Failures and Traffic Distribution

**Historical Context**: 19 incidents, often related to deployment issues or instance failures.

#### Incident Examples from History

**Ticket #INC-2023-3421**: All targets unhealthy after deployment
- **Root Cause**: Health check endpoint changed without updating ALB config
- **Resolution**: Updated health check path
- **Prevention**: Added health check validation to deployment pipeline

#### Standard Resolution Playbook

##### Step 1: Target Health Assessment (0-2 minutes)
```bash
# Check target group health
aws elbv2 describe-target-health --target-group-arn arn:aws:elasticloadbalancing:us-east-1:123456789012:targetgroup/shopsmart-prod-tg/1234567890123456

# Get ALB status
aws elbv2 describe-load-balancers --names shopsmart-prod-alb
```

##### Step 2: Health Check Validation (2-5 minutes)
```bash
# Test health check endpoint directly
TARGET_IPS=$(aws elbv2 describe-target-health --target-group-arn arn:aws:elasticloadbalancing:us-east-1:123456789012:targetgroup/shopsmart-prod-tg/1234567890123456 --query 'TargetHealthDescriptions[].Target.Id' --output text)

for ip in $TARGET_IPS; do
    curl -I http://$ip:8080/health
done
```

##### Step 3: Quick Remediation (5-10 minutes)
```bash
# Deregister unhealthy targets
aws elbv2 deregister-targets --target-group-arn arn:aws:elasticloadbalancing:us-east-1:123456789012:targetgroup/shopsmart-prod-tg/1234567890123456 --targets Id=i-unhealthy-instance

# Force new instance launch
aws autoscaling set-desired-capacity --auto-scaling-group-name shopsmart-prod-asg --desired-capacity 4
```

**Success Rate**: 84% of ALB incidents resolved within 10 minutes using this approach

---

## S3 Access Issues

### Pattern: Permission Denied and Bucket Policy Conflicts

**Historical Context**: 15 incidents, usually after policy changes or IAM updates.

#### Incident Examples from History

**Ticket #INC-2023-0987**: Product images not loading
- **Root Cause**: Bucket policy accidentally removed public read access
- **Resolution**: Restored bucket policy with proper public access controls
- **Impact**: 100% of product pages affected

#### Standard Resolution Playbook

##### Step 1: Access Verification (0-3 minutes)
```bash
# Test bucket access
aws s3 ls s3://shopsmart-product-images/ --region us-east-1

# Check bucket policy
aws s3api get-bucket-policy --bucket shopsmart-product-images
```

##### Step 2: Permission Analysis (3-7 minutes)
```bash
# Verify IAM role permissions
aws iam get-role-policy --role-name ShopSmartEC2Role --policy-name S3AccessPolicy

# Check bucket ACL
aws s3api get-bucket-acl --bucket shopsmart-product-images
```

##### Step 3: Quick Policy Fix (7-12 minutes)
```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "PublicReadGetObject",
            "Effect": "Allow",
            "Principal": "*",
            "Action": "s3:GetObject",
            "Resource": "arn:aws:s3:::shopsmart-product-images/*",
            "Condition": {
                "StringEquals": {
                    "s3:ExistingObjectTag/data-classification": "public-dataset"
                }
            }
        }
    ]
}
```

```bash
# Apply emergency policy
aws s3api put-bucket-policy --bucket shopsmart-product-images --policy file://emergency-policy.json
```

---

## Auto Scaling Failures

### Pattern: Scaling Events Not Triggering or Instances Failing Health Checks

#### Incident Examples from History

**Ticket #INC-2023-2890**: Auto scaling not responding to traffic spike
- **Root Cause**: CloudWatch metrics delayed due to agent configuration
- **Resolution**: Fixed CloudWatch agent, adjusted scaling thresholds
- **Impact**: Response times increased 300% during peak traffic

#### Standard Resolution Playbook

##### Step 1: Scaling Activity Review (0-3 minutes)
```bash
# Check recent scaling activities
aws autoscaling describe-scaling-activities --auto-scaling-group-name shopsmart-prod-asg --max-items 10

# Verify current capacity vs desired
aws autoscaling describe-auto-scaling-groups --auto-scaling-group-names shopsmart-prod-asg
```

##### Step 2: Manual Scaling (3-6 minutes)
```bash
# Emergency scale-out
aws autoscaling set-desired-capacity --auto-scaling-group-name shopsmart-prod-asg --desired-capacity 8 --honor-cooldown false

# Check instance launch status
aws ec2 describe-instances --filters "Name=tag:aws:autoscaling:groupName,Values=shopsmart-prod-asg" --query 'Reservations[].Instances[?State.Name==`pending` || State.Name==`running`].[InstanceId,State.Name,LaunchTime]'
```

##### Step 3: Metric Validation (6-10 minutes)
```bash
# Check CloudWatch metrics
aws cloudwatch get-metric-statistics --namespace AWS/EC2 --metric-name CPUUtilization --dimensions Name=AutoScalingGroupName,Value=shopsmart-prod-asg --start-time $(date -d '30 minutes ago' -u +%Y-%m-%dT%H:%M:%S) --end-time $(date -u +%Y-%m-%dT%H:%M:%S) --period 300 --statistics Average
```

---

## Cache Performance Issues

### Pattern: ElastiCache Connection Failures and Memory Pressure

#### Incident Examples from History

**Ticket #INC-2023-1654**: Redis cluster failover causing application errors
- **Root Cause**: Application not handling Redis failover gracefully
- **Resolution**: Implemented connection retry logic, added circuit breaker
- **Impact**: 25% of requests experiencing cache misses

#### Standard Resolution Playbook

##### Step 1: Cache Cluster Status (0-2 minutes)
```bash
# Check ElastiCache cluster health
aws elasticache describe-cache-clusters --cache-cluster-id shopsmart-prod-redis --show-cache-node-info

# Verify replication group status
aws elasticache describe-replication-groups --replication-group-id shopsmart-prod-redis-rg
```

##### Step 2: Connection Testing (2-5 minutes)
```bash
# Test Redis connectivity from application servers
redis-cli -h shopsmart-prod-redis.abc123.cache.amazonaws.com -p 6379 ping

# Check memory usage
redis-cli -h shopsmart-prod-redis.abc123.cache.amazonaws.com -p 6379 info memory
```

##### Step 3: Cache Failover (5-10 minutes)
```bash
# Manual failover if needed
aws elasticache test-failover --replication-group-id shopsmart-prod-redis-rg --node-group-id 0001
```

---

## Network Connectivity Issues

### Pattern: VPC Configuration and Security Group Problems

#### Standard Resolution Playbook

##### Step 1: Network Path Analysis (0-5 minutes)
```bash
# Check security group rules
aws ec2 describe-security-groups --group-ids sg-web-servers sg-database sg-cache

# Verify route tables
aws ec2 describe-route-tables --filters "Name=vpc-id,Values=vpc-12345678"

# Check NACL rules
aws ec2 describe-network-acls --filters "Name=vpc-id,Values=vpc-12345678"
```

##### Step 2: Connectivity Testing (5-10 minutes)
```bash
# Test connectivity between tiers
# From web server to database
telnet database-endpoint.region.rds.amazonaws.com 5432

# From web server to cache
telnet cache-endpoint.region.cache.amazonaws.com 6379
```

---

## Security Incidents

### Pattern: Unauthorized Access and Suspicious Activity

#### Incident Examples from History

**Ticket #SEC-2023-0156**: Unusual API access patterns detected
- **Root Cause**: Compromised API key being used for data scraping
- **Resolution**: Revoked API key, implemented rate limiting
- **Impact**: Potential data exposure, increased infrastructure costs

#### Standard Response Playbook

##### Step 1: Immediate Containment (0-10 minutes)
```bash
# Block suspicious IP addresses
aws wafv2 update-ip-set --scope CLOUDFRONT --id suspicious-ips --addresses 192.0.2.1/32,203.0.113.0/24

# Disable compromised IAM users/roles
aws iam attach-user-policy --user-name suspicious-user --policy-arn arn:aws:iam::aws:policy/AWSDenyAll
```

##### Step 2: Evidence Collection (10-30 minutes)
```bash
# Collect CloudTrail logs
aws logs filter-log-events --log-group-name CloudTrail/SecurityEvents --start-time $(date -d '24 hours ago' +%s)000 --filter-pattern "{ $.sourceIPAddress = \"192.0.2.1\" }"

# Check access logs
aws s3 cp s3://security-logs/access-logs/ . --recursive --exclude "*" --include "*$(date +%Y-%m-%d)*"
```

---

## Post-Incident Procedures

### Immediate Actions (Within 1 hour of resolution)

1. **Update Incident Ticket**
   - Document root cause
   - List all actions taken
   - Record resolution time
   - Note any temporary fixes that need permanent solutions

2. **Stakeholder Communication**
   ```
   Subject: [RESOLVED] P1 Incident - Lambda Function Failures
   
   Incident: INC-2024-XXXX
   Status: RESOLVED
   Duration: 23 minutes
   Impact: 15% of user requests affected
   Root Cause: Database connection pool exhaustion
   Resolution: Increased Lambda memory allocation and implemented connection pooling
   
   Next Steps:
   - Permanent fix scheduled for next sprint
   - Monitoring enhancements to prevent recurrence
   ```

3. **Monitoring Validation**
   - Verify all metrics have returned to normal
   - Confirm alerts are functioning
   - Check for any secondary impacts

### Post-Incident Review (Within 48 hours)

#### Template for PIR Document

```markdown
# Post-Incident Review: INC-2024-XXXX

## Incident Summary
- **Date/Time**: 2024-01-15 14:30 UTC
- **Duration**: 23 minutes
- **Severity**: P1
- **Services Affected**: User Authentication, Product Search
- **Customer Impact**: 15% of requests failed

## Timeline
- 14:30 - Alert triggered: Lambda error rate spike
- 14:32 - On-call engineer paged
- 14:35 - Initial investigation started
- 14:42 - Root cause identified (connection pool exhaustion)
- 14:45 - Mitigation applied (increased memory allocation)
- 14:53 - Service fully restored

## Root Cause Analysis
**Primary Cause**: Database connection pool exhaustion in Lambda function
**Contributing Factors**:
- Recent traffic increase not accounted for in capacity planning
- Connection pool size not optimized for current load
- Lack of connection pool monitoring

## What Went Well
- Alert fired within 2 minutes of issue start
- On-call engineer responded quickly
- Standard runbook followed effectively
- Communication to stakeholders was timely

## What Could Be Improved
- Connection pool monitoring should have caught this earlier
- Capacity planning process needs enhancement
- Lambda memory allocation should be reviewed regularly

## Action Items
1. **[HIGH]** Implement connection pool monitoring (Owner: @platform-team, Due: 2024-01-22)
2. **[MEDIUM]** Review all Lambda memory allocations (Owner: @backend-team, Due: 2024-01-29)
3. **[LOW]** Update capacity planning process (Owner: @architecture-team, Due: 2024-02-05)
```

### Knowledge Base Updates

After each incident, update the runbook with:
- New resolution patterns discovered
- Updated commands or procedures
- Additional monitoring recommendations
- Prevention strategies

### Metrics Tracking

Track these metrics monthly:
- **MTTR (Mean Time To Resolution)**: Target < 30 minutes for P1 incidents
- **MTBF (Mean Time Between Failures)**: Track by incident type
- **Runbook Effectiveness**: % of incidents resolved using standard procedures
- **False Positive Rate**: % of alerts that weren't actual incidents

---

## Emergency Contacts

### On-Call Rotation
- **Primary**: platform-oncall@company.com
- **Secondary**: backend-oncall@company.com
- **Escalation**: engineering-leads@company.com

### External Vendors
- **AWS Support**: Enterprise Support Case
- **DataDog**: support@datadoghq.com
- **PagerDuty**: Emergency escalation available

### Communication Channels
- **Slack**: #incident-response
- **Status Page**: status.shopsmart.com
- **Customer Support**: support@shopsmart.com

---

**Document Maintenance**
- Review quarterly with all on-call engineers
- Update after each major incident
- Validate all commands and procedures monthly
- Archive outdated procedures annually
