# AWS Resource Tagging Policy

## Overview

This document defines the mandatory and recommended tagging standards for all AWS resources within our organization. Proper tagging enables cost allocation, resource management, security compliance, and operational efficiency.

## Mandatory Tags

All AWS resources **MUST** include the following tags:

### Core Business Tags

| Tag Key | Description | Valid Values | Example |
|---------|-------------|--------------|---------|
| `Environment` | Deployment environment | `production`, `staging`, `development`, `test` | `production` |
| `Application` | Application or service name | Alphanumeric, hyphens allowed | `user-authentication` |
| `Owner` | Team or individual responsible | Email address or team name | `platform-team@company.com` |
| `CostCenter` | Cost allocation identifier | Department code | `ENG-001` |
| `Project` | Project or initiative name | Alphanumeric, hyphens allowed | `mobile-app-v2` |

### Operational Tags

| Tag Key | Description | Valid Values | Example |
|---------|-------------|--------------|---------|
| `Backup` | Backup requirement level | `required`, `optional`, `none` | `required` |
| `Monitoring` | Monitoring requirement | `critical`, `standard`, `minimal` | `critical` |
| `Compliance` | Compliance requirements | `pci`, `hipaa`, `sox`, `gdpr`, `none` | `pci,gdpr` |

## Environment-Specific Requirements

### Production Environment
Production resources (`Environment: production`) **MUST** include additional tags:

| Tag Key | Description | Valid Values | Example |
|---------|-------------|--------------|---------|
| `DataClassification` | Data sensitivity level | `public`, `internal`, `confidential`, `restricted` | `confidential` |
| `BusinessCriticality` | Business impact level | `critical`, `high`, `medium`, `low` | `critical` |
| `MaintenanceWindow` | Preferred maintenance time | `weekday-night`, `weekend`, `24x7-available` | `weekend` |

### Development/Test Environments
Non-production resources **SHOULD** include:

| Tag Key | Description | Valid Values | Example |
|---------|-------------|--------------|---------|
| `AutoShutdown` | Automatic shutdown schedule | `daily`, `weekends`, `never` | `daily` |
| `ExpirationDate` | Resource cleanup date | YYYY-MM-DD format | `2024-12-31` |

## Service-Specific Tagging Requirements

### S3 Buckets

#### Public Buckets
S3 buckets with public access **MUST** include:

```yaml
Tags:
  DataClassification: "public-dataset"
  PublicAccess: "approved"
  AccessRestriction: "read-only"
  ApprovedBy: "data-governance-team@company.com"
```

#### Production Buckets
All production S3 buckets **MUST** include:

```yaml
Tags:
  Versioning: "enabled"
  Logging: "enabled"
  Encryption: "enabled"
  CrossRegionReplication: "enabled"  # For critical data
```

### EC2 Instances

EC2 instances **MUST** include:

| Tag Key | Description | Valid Values | Example |
|---------|-------------|--------------|---------|
| `InstancePurpose` | Primary function | `web-server`, `database`, `cache`, `worker` | `web-server` |
| `PatchGroup` | Patching schedule group | `critical`, `standard`, `delayed` | `critical` |
| `AutoScaling` | Auto scaling group member | `true`, `false` | `true` |

### RDS Instances

RDS instances **MUST** include:

| Tag Key | Description | Valid Values | Example |
|---------|-------------|--------------|---------|
| `DatabaseEngine` | Database type | `mysql`, `postgresql`, `oracle`, `sqlserver` | `postgresql` |
| `BackupRetention` | Backup retention days | Number (1-35) | `30` |
| `MultiAZ` | Multi-AZ deployment | `true`, `false` | `true` |

### Lambda Functions

Lambda functions **MUST** include:

| Tag Key | Description | Valid Values | Example |
|---------|-------------|--------------|---------|
| `Runtime` | Lambda runtime version | Runtime identifier | `nodejs18.x` |
| `TriggerType` | Primary trigger source | `api-gateway`, `s3`, `dynamodb`, `scheduled` | `api-gateway` |
| `MemorySize` | Allocated memory | Memory in MB | `512` |

## Compliance and Security Tags

### PCI Compliance
Resources handling payment data **MUST** include:

```yaml
Tags:
  Compliance: "pci"
  DataType: "payment-card-data"
  PCIScope: "in-scope"
  LogDestination: "cloudwatch"  # PCI environments use CloudWatch
```

### GDPR Compliance
Resources handling EU personal data **MUST** include:

```yaml
Tags:
  Compliance: "gdpr"
  DataType: "personal-data"
  DataResidency: "eu-west-1"
  RetentionPeriod: "7-years"
```

### Identity and Access
All production resources **MUST** include:

```yaml
Tags:
  IdentityProvider: "okta"
  AccessMethod: "identity-center"
  SSOEnabled: "true"
```

## Tag Formatting Standards

### Naming Conventions
- **Tag Keys**: Use PascalCase (e.g., `CostCenter`, `DataClassification`)
- **Tag Values**: Use lowercase with hyphens (e.g., `user-authentication`, `web-server`)
- **No Spaces**: Use hyphens instead of spaces
- **Consistent Abbreviations**: Use standard abbreviations (e.g., `prod` not `production`)

### Character Limits
- **Tag Keys**: Maximum 128 characters
- **Tag Values**: Maximum 256 characters
- **Total Tags**: Maximum 50 tags per resource

### Reserved Prefixes
The following prefixes are reserved for system use:
- `aws:` - AWS system tags
- `cloudformation:` - CloudFormation stack tags
- `elasticbeanstalk:` - Elastic Beanstalk tags

## Automation and Enforcement

### Required Automation
- **Tag Inheritance**: Child resources inherit parent tags where applicable
- **Default Tags**: Apply organization defaults via CloudFormation/Terraform
- **Validation**: Implement tag validation in CI/CD pipelines

### Enforcement Mechanisms
- **Service Control Policies (SCPs)**: Prevent resource creation without required tags
- **Config Rules**: Monitor tag compliance and generate reports
- **Cost Allocation**: Enable detailed cost tracking through tags

### Example SCP for Tag Enforcement

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Deny",
      "Action": [
        "ec2:RunInstances",
        "rds:CreateDBInstance",
        "s3:CreateBucket"
      ],
      "Resource": "*",
      "Condition": {
        "Null": {
          "aws:RequestedRegion": "false"
        },
        "ForAllValues:StringNotEquals": {
          "aws:TagKeys": [
            "Environment",
            "Application", 
            "Owner",
            "CostCenter"
          ]
        }
      }
    }
  ]
}
```

## Monitoring and Reporting

### Tag Compliance Dashboard
Monitor tag compliance through:
- **AWS Config**: Tag compliance rules and reports
- **Cost Explorer**: Cost allocation by tags
- **Resource Groups**: Organize resources by tag combinations

### Regular Audits
- **Monthly**: Tag compliance reports by team
- **Quarterly**: Cost allocation accuracy review
- **Annually**: Tag policy effectiveness assessment

## Examples

### Complete Production Web Server
```yaml
Tags:
  # Mandatory Core Tags
  Environment: "production"
  Application: "e-commerce-frontend"
  Owner: "frontend-team@company.com"
  CostCenter: "ENG-001"
  Project: "website-redesign"
  
  # Mandatory Operational Tags
  Backup: "required"
  Monitoring: "critical"
  Compliance: "pci,gdpr"
  
  # Production-Specific Tags
  DataClassification: "confidential"
  BusinessCriticality: "critical"
  MaintenanceWindow: "weekend"
  
  # EC2-Specific Tags
  InstancePurpose: "web-server"
  PatchGroup: "critical"
  AutoScaling: "true"
  
  # Identity Tags
  IdentityProvider: "okta"
  AccessMethod: "identity-center"
  SSOEnabled: "true"
```

### Development Database
```yaml
Tags:
  # Mandatory Core Tags
  Environment: "development"
  Application: "user-service"
  Owner: "backend-team@company.com"
  CostCenter: "ENG-002"
  Project: "api-modernization"
  
  # Mandatory Operational Tags
  Backup: "optional"
  Monitoring: "minimal"
  Compliance: "none"
  
  # Development-Specific Tags
  AutoShutdown: "daily"
  ExpirationDate: "2024-06-30"
  
  # RDS-Specific Tags
  DatabaseEngine: "postgresql"
  BackupRetention: "7"
  MultiAZ: "false"
```

## Policy Violations and Remediation

### Common Violations
1. **Missing Required Tags**: Resources without mandatory tags
2. **Incorrect Tag Values**: Values not matching approved list
3. **Inconsistent Formatting**: Wrong case or special characters
4. **Outdated Tags**: Tags not updated after resource changes

### Remediation Process
1. **Automated Detection**: Config rules identify violations
2. **Notification**: Alert resource owners via SNS/email
3. **Grace Period**: 7 days to remediate violations
4. **Enforcement**: Automatic resource shutdown/restriction after grace period

## Contact and Support

- **Policy Questions**: cloud-governance@company.com
- **Technical Support**: platform-team@company.com
- **Emergency Exceptions**: Submit ticket to IT Service Desk

---

**Document Version**: 2.1  
**Last Updated**: January 2024  
**Next Review**: July 2024  
**Approved By**: Cloud Governance Committee
