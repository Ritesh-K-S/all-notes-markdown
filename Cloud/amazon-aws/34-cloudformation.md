# Chapter 34: CloudFormation

---

## Table of Contents

- [Overview](#overview)
- [Part 1: CloudFormation Fundamentals](#part-1-cloudformation-fundamentals)
- [Part 2: Creating a Stack (Full Portal Walkthrough)](#part-2-creating-a-stack-full-portal-walkthrough)
- [Part 3: Template Anatomy Deep Dive](#part-3-template-anatomy-deep-dive)
- [Part 4: Intrinsic Functions & Pseudo Parameters](#part-4-intrinsic-functions--pseudo-parameters)
- [Part 5: Change Sets, Drift Detection & Stack Policies](#part-5-change-sets-drift-detection--stack-policies)
- [Part 6: Nested Stacks & StackSets](#part-6-nested-stacks--stacksets)
- [Part 7: Advanced Features](#part-7-advanced-features)
- [Part 8: CLI Examples & Best Practices](#part-8-cli-examples--best-practices)
- [Quick Reference](#quick-reference)
- [What's Next?](#whats-next)

---

## Overview

### What is Infrastructure as Code (IaC)? Why Do We Need It?

Imagine you need to set up a production environment: a VPC, subnets, security groups, an EC2 instance, an RDS database, a load balancer, and DNS records. You could click through the AWS Console for 2 hours... and then do it all over again for staging, dev, and disaster recovery.

**Infrastructure as Code (IaC)** means you write a **blueprint** (template) that describes all your infrastructure, and a tool creates it for you. It's like the difference between building furniture by hand vs. using an IKEA instruction manual — the manual is repeatable, shareable, and version-controlled.

**CloudFormation is AWS's native IaC tool.** You write a YAML/JSON template describing what you want, and CloudFormation figures out the correct order to create everything, handles dependencies, and can tear it all down with one command.

**Why IaC matters:**
- 🔁 **Repeatable**: Same template creates identical environments every time
- 📝 **Version-controlled**: Track infrastructure changes in Git like code
- ⚡ **Fast**: Create 50 resources in minutes instead of hours of clicking
- 🗑️ **Easy cleanup**: Delete the stack and ALL resources are removed

AWS CloudFormation is the native Infrastructure as Code (IaC) service. You define your infrastructure in JSON or YAML templates, and CloudFormation provisions and manages the resources for you.

```
What you'll learn:
├── CloudFormation Fundamentals (template → stack → resources)
├── Creating stacks (full portal walkthrough)
├── Template anatomy (all sections explained)
├── Intrinsic functions (!Ref, !Sub, !GetAtt, etc.)
├── Change sets, drift detection, stack policies
├── Nested stacks & StackSets
├── Advanced features (custom resources, macros, modules)
└── CLI examples & best practices
```

---

## Part 1: CloudFormation Fundamentals

```
┌─────────────────────────────────────────────────────────────────────┐
│           HOW CLOUDFORMATION WORKS                                    │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ ┌──────────────┐    ┌───────────────┐    ┌────────────────────┐   │
│ │   Template   │───►│ CloudFormation │───►│   AWS Resources    │   │
│ │  (YAML/JSON) │    │   Service      │    │ ├── VPC            │   │
│ │              │    │   ┌───────┐    │    │ ├── EC2 instances  │   │
│ │ Resources:   │    │   │ Stack │    │    │ ├── RDS databases  │   │
│ │   VPC:       │    │   └───────┘    │    │ ├── S3 buckets     │   │
│ │   EC2:       │    │   Creates/     │    │ └── IAM roles      │   │
│ │   RDS:       │    │   Updates/     │    │                     │   │
│ │              │    │   Deletes      │    │                     │   │
│ └──────────────┘    └───────────────┘    └────────────────────┘   │
│                                                                       │
│ Key concepts:                                                        │
│ ├── Template: Blueprint (YAML/JSON) defining resources            │
│ ├── Stack: Running instance of a template                         │
│ ├── Stack Events: Log of all operations                           │
│ ├── Change Set: Preview changes before applying                   │
│ ├── Drift: Difference between actual and template state          │
│ └── Stack lifecycle: CREATE → UPDATE → DELETE                     │
│                                                                       │
│ Benefits:                                                            │
│ ├── Declarative: Describe WHAT, not HOW                          │
│ ├── Version controlled: Templates in Git                         │
│ ├── Repeatable: Same template → same infrastructure             │
│ ├── Dependency management: Automatic resource ordering           │
│ ├── Rollback: Automatic rollback on failure                      │
│ └── Drift detection: Find manual changes                         │
│                                                                       │
│ Pricing: Free! (only pay for resources created)                    │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 2: Creating a Stack (Full Portal Walkthrough)

```
Console → CloudFormation → Stacks → Create stack

┌─────────────────────────────────────────────────────────────────┐
│           STEP 1: CREATE STACK                                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ Prerequisite - Prepare template:                               │
│ ● Template is ready                                            │
│ ○ Use a sample template                                       │
│ ○ Create template in Designer (visual editor)                 │
│                                                                   │
│ Specify template:                                              │
│ ● Upload a template file  [Choose file]                       │
│ ○ Amazon S3 URL                                               │
│ → S3: [https://s3.amazonaws.com/mybucket/template.yaml]      │
│                                                                   │
│ → ⚡ For CI/CD: Upload to S3 first, reference URL            │
│ → Max template size: 51,200 bytes (upload) or 460,800 (S3)  │
│                                                                   │
│                    [Next]                                        │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│           STEP 2: SPECIFY STACK DETAILS                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ Stack name: [prod-web-infrastructure]                          │
│ → Must be unique in region+account                            │
│ → Used for resource naming and identification                │
│                                                                   │
│ Parameters (defined in template):                              │
│ Environment: [production ▼]                                   │
│ InstanceType: [t3.medium ▼]                                   │
│ KeyPair: [prod-keypair ▼]                                     │
│ VpcCIDR: [10.0.0.0/16]                                       │
│ DatabasePassword: [****] (NoEcho parameter)                   │
│ → Parameters make templates reusable (dev/staging/prod)     │
│                                                                   │
│                    [Next]                                        │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│           STEP 3: CONFIGURE STACK OPTIONS                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ Tags:                                                           │
│ Key: [environment]  Value: [production]                       │
│ Key: [team]  Value: [platform]                                │
│ → Tags applied to stack AND all resources                    │
│                                                                   │
│ Permissions:                                                    │
│ IAM role: [arn:aws:iam::123456:role/CloudFormationRole ▼]    │
│ → Role CloudFormation assumes to create resources            │
│ → ⚡ Best practice: Dedicated CF role with least privilege    │
│                                                                   │
│ Stack failure options:                                          │
│ ● Roll back all stack resources                               │
│ ○ Preserve successfully provisioned resources                │
│ → Rollback: Delete everything on failure (clean state)       │
│ → Preserve: Keep what worked (debugging)                     │
│                                                                   │
│ Advanced options:                                              │
│ Stack policy: [JSON policy URL/body]                          │
│ → Protect specific resources from updates                    │
│ → e.g., prevent RDS database from being replaced            │
│                                                                   │
│ Rollback configuration:                                        │
│ CloudWatch alarm: [prod-errors-alarm ▼]                       │
│ Monitoring time: [5] minutes                                  │
│ → Rolls back if alarm triggers during/after creation         │
│                                                                   │
│ Notification options:                                          │
│ SNS topic: [arn:aws:sns:...:cf-notifications ▼]              │
│ → Notifies on stack events (create, update, delete)          │
│                                                                   │
│ Timeout: [60] minutes (optional)                              │
│ Termination protection: ☑ Enable                              │
│ → ⚡ Always enable for production stacks                      │
│ → Prevents accidental stack deletion                         │
│                                                                   │
│                    [Next]                                        │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│           STEP 4: REVIEW & CREATE                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ Review all settings...                                         │
│                                                                   │
│ Capabilities:                                                   │
│ ☑ I acknowledge that AWS CloudFormation might create IAM      │
│   resources with custom names                                  │
│ ☑ I acknowledge that AWS CloudFormation might require the     │
│   following capability: CAPABILITY_AUTO_EXPAND                 │
│ → Required when template creates IAM resources or uses macros│
│                                                                   │
│ Transform:                                                     │
│ AWS::Serverless-2016-10-31 → SAM transform detected          │
│                                                                   │
│                    [Submit]                                      │
└─────────────────────────────────────────────────────────────────┘
```

---

## Part 3: Template Anatomy Deep Dive

```yaml
AWSTemplateFormatVersion: "2010-09-09"
Description: "Production web infrastructure"

# ── Metadata ──
# UI customization, parameter grouping
Metadata:
  AWS::CloudFormation::Interface:
    ParameterGroups:
      - Label: { default: "Network Configuration" }
        Parameters: [VpcCIDR, PublicSubnetCIDR]
      - Label: { default: "Instance Configuration" }
        Parameters: [InstanceType, KeyPair]

# ── Parameters ──
# User-provided values (makes template reusable)
Parameters:
  Environment:
    Type: String
    Default: production
    AllowedValues: [development, staging, production]
    Description: "Deployment environment"

  InstanceType:
    Type: String
    Default: t3.medium
    AllowedValues: [t3.micro, t3.small, t3.medium, t3.large]
    Description: "EC2 instance type"

  VpcCIDR:
    Type: String
    Default: "10.0.0.0/16"
    AllowedPattern: '(\d{1,3})\.(\d{1,3})\.(\d{1,3})\.(\d{1,3})/(\d{1,2})'
    ConstraintDescription: "Must be valid CIDR block"

  DatabasePassword:
    Type: String
    NoEcho: true                    # Hidden in console
    MinLength: 8
    MaxLength: 41

  LatestAmiId:
    Type: AWS::SSM::Parameter::Value<AWS::EC2::Image::Id>
    Default: /aws/service/ami-amazon-linux-latest/al2023-ami-kernel-default-x86_64

# ── Mappings ──
# Static lookup tables
Mappings:
  RegionConfig:
    us-east-1:
      AMI: ami-0123456789abcdef0
      AZ1: us-east-1a
      AZ2: us-east-1b
    eu-west-1:
      AMI: ami-0fedcba9876543210
      AZ1: eu-west-1a
      AZ2: eu-west-1b

# ── Conditions ──
# Conditional resource creation
Conditions:
  IsProduction: !Equals [!Ref Environment, production]
  CreateReadReplica: !Equals [!Ref Environment, production]

# ── Resources (REQUIRED — only mandatory section) ──
Resources:
  VPC:
    Type: AWS::EC2::VPC
    Properties:
      CidrBlock: !Ref VpcCIDR
      EnableDnsHostnames: true
      Tags:
        - Key: Name
          Value: !Sub "${Environment}-vpc"

  WebServer:
    Type: AWS::EC2::Instance
    DependsOn: RDSDatabase        # Explicit dependency
    Properties:
      ImageId: !Ref LatestAmiId
      InstanceType: !Ref InstanceType
      SubnetId: !Ref PublicSubnet
      SecurityGroupIds:
        - !GetAtt WebSecurityGroup.GroupId
      Tags:
        - Key: Name
          Value: !Sub "${Environment}-web"

  RDSDatabase:
    Type: AWS::RDS::DBInstance
    DeletionPolicy: Snapshot      # Take snapshot before deleting
    UpdateReplacePolicy: Snapshot
    Properties:
      Engine: mysql
      DBInstanceClass: !If [IsProduction, db.r6g.large, db.t3.medium]
      MasterUsername: admin
      MasterUserPassword: !Ref DatabasePassword
      MultiAZ: !If [IsProduction, true, false]

  ReadReplica:
    Type: AWS::RDS::DBInstance
    Condition: CreateReadReplica  # Only created if condition is true
    Properties:
      SourceDBInstanceIdentifier: !Ref RDSDatabase
      DBInstanceClass: db.r6g.large

# ── Outputs ──
# Values to export or display
Outputs:
  VpcId:
    Description: "VPC ID"
    Value: !Ref VPC
    Export:
      Name: !Sub "${Environment}-VpcId"   # Cross-stack reference

  WebServerPublicIP:
    Description: "Web server public IP"
    Value: !GetAtt WebServer.PublicIp

  DatabaseEndpoint:
    Description: "RDS endpoint"
    Value: !GetAtt RDSDatabase.Endpoint.Address
```

```
┌─────────────────────────────────────────────────────────────────────┐
│           TEMPLATE SECTIONS                                           │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Section          │ Required │ Purpose                               │
│ ─────────────────┼──────────┼──────────────────────────────────── │
│ AWSTemplateFormat│ No       │ Template version (always 2010-09-09)│
│ Description      │ No       │ What this template does              │
│ Metadata         │ No       │ UI hints, parameter groups           │
│ Parameters       │ No       │ User inputs (reusable templates)    │
│ Mappings         │ No       │ Static lookup tables                 │
│ Conditions       │ No       │ Conditional resource creation       │
│ Resources        │ YES ⚡   │ AWS resources to create             │
│ Outputs          │ No       │ Values to display/export             │
│ Transform        │ No       │ Macros (SAM, Include)               │
│ Rules            │ No       │ Parameter validation rules           │
│                                                                       │
│ Resource attributes:                                                │
│ ├── DependsOn: Explicit ordering                                │
│ ├── DeletionPolicy: Retain, Snapshot, or Delete                │
│ ├── UpdateReplacePolicy: Retain, Snapshot, or Delete           │
│ ├── Condition: Only create if condition is true                 │
│ ├── CreationPolicy: Wait for signal (cfn-signal)               │
│ └── UpdatePolicy: Rolling update config (ASG)                   │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 4: Intrinsic Functions & Pseudo Parameters

```
┌─────────────────────────────────────────────────────────────────────┐
│           INTRINSIC FUNCTIONS                                         │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ !Ref                                                                 │
│ → Reference parameter value or resource physical ID               │
│ → !Ref VpcCIDR → "10.0.0.0/16"                                  │
│ → !Ref MyVPC → "vpc-abc123" (physical ID)                        │
│                                                                       │
│ !GetAtt                                                              │
│ → Get attribute of a resource                                     │
│ → !GetAtt MyInstance.PublicIp → "54.123.45.67"                   │
│ → !GetAtt MyDB.Endpoint.Address → "mydb.xyz.rds.amazonaws.com" │
│                                                                       │
│ !Sub                                                                 │
│ → String substitution (variable interpolation)                    │
│ → !Sub "${Environment}-web-server"                               │
│ → !Sub "arn:aws:s3:::${BucketName}/*"                           │
│                                                                       │
│ !Join                                                                │
│ → Concatenate strings with delimiter                              │
│ → !Join ["-", [!Ref Environment, web, server]]                  │
│                                                                       │
│ !Select                                                              │
│ → Select item from list by index                                 │
│ → !Select [0, [us-east-1a, us-east-1b]] → "us-east-1a"         │
│                                                                       │
│ !Split                                                               │
│ → Split string into list                                          │
│ → !Split [",", "a,b,c"] → ["a","b","c"]                        │
│                                                                       │
│ !FindInMap                                                           │
│ → Lookup value in Mappings section                                │
│ → !FindInMap [RegionConfig, !Ref "AWS::Region", AMI]            │
│                                                                       │
│ !If                                                                  │
│ → Conditional value                                               │
│ → !If [IsProduction, db.r6g.large, db.t3.medium]                │
│                                                                       │
│ !Equals, !Not, !And, !Or                                            │
│ → Condition functions                                             │
│ → !Equals [!Ref Environment, production]                         │
│                                                                       │
│ !ImportValue                                                         │
│ → Import output from another stack                                │
│ → !ImportValue prod-VpcId                                        │
│                                                                       │
│ !GetAZs                                                              │
│ → Get availability zones for region                               │
│ → !GetAZs "" → ["us-east-1a","us-east-1b","us-east-1c"]        │
│                                                                       │
│ !Cidr                                                                │
│ → Generate CIDR blocks                                            │
│ → !Cidr [!Ref VpcCIDR, 4, 8] → 4 /24 subnets                  │
│                                                                       │
│ Pseudo Parameters:                                                   │
│ ├── AWS::AccountId → "123456789012"                              │
│ ├── AWS::Region → "us-east-1"                                   │
│ ├── AWS::StackName → "prod-web-infrastructure"                  │
│ ├── AWS::StackId → Full stack ARN                                │
│ ├── AWS::URLSuffix → "amazonaws.com"                             │
│ └── AWS::NoValue → Remove property conditionally                │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 5: Change Sets, Drift Detection & Stack Policies

```
┌─────────────────────────────────────────────────────────────────────┐
│           CHANGE SETS                                                  │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Console → CloudFormation → [Stack] → Stack actions → Create       │
│ change set for current stack                                        │
│                                                                       │
│ Upload modified template → Review changes BEFORE applying         │
│                                                                       │
│ Change types shown:                                                  │
│ ├── Add: New resource will be created                             │
│ ├── Modify: Resource will be updated in-place                    │
│ ├── Remove: Resource will be deleted                              │
│ └── Replacement: Resource will be REPLACED (delete + recreate)   │
│     → ⚠️ Replacement means downtime! (e.g., changing RDS engine)│
│                                                                       │
│ ⚡ ALWAYS use change sets for production stack updates              │
│                                                                       │
│ ── DRIFT DETECTION ──                                               │
│ Console → [Stack] → Stack actions → Detect drift                  │
│                                                                       │
│ Detects manual changes made outside CloudFormation:                │
│ ├── IN_SYNC: Resource matches template definition                │
│ ├── MODIFIED: Resource changed (shows diff)                      │
│ ├── DELETED: Resource was deleted manually                       │
│ └── NOT_CHECKED: Resource type doesn't support drift             │
│                                                                       │
│ → Shows exact property differences                                │
│ → ⚡ Run drift detection regularly in production                  │
│ → Fix: Update template to match actual state, or reimport       │
│                                                                       │
│ ── STACK POLICIES ──                                                │
│ Prevent accidental updates to critical resources:                  │
│                                                                       │
│ {                                                                    │
│   "Statement": [                                                    │
│     {                                                                │
│       "Effect": "Deny",                                             │
│       "Action": "Update:Replace",                                  │
│       "Principal": "*",                                             │
│       "Resource": "LogicalResourceId/ProductionDatabase"           │
│     },                                                               │
│     {                                                                │
│       "Effect": "Allow",                                            │
│       "Action": "Update:*",                                        │
│       "Principal": "*",                                             │
│       "Resource": "*"                                                │
│     }                                                                │
│   ]                                                                  │
│ }                                                                    │
│ → Prevents RDS from being replaced during stack update            │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 6: Nested Stacks & StackSets

```
┌─────────────────────────────────────────────────────────────────────┐
│           NESTED STACKS                                               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Break large templates into smaller, reusable components:           │
│                                                                       │
│ ┌──────────────────┐                                                │
│ │ Root Stack       │                                                │
│ │ ├── VPC Stack    │ (nested - separate template)                 │
│ │ ├── App Stack    │ (nested - separate template)                 │
│ │ └── DB Stack     │ (nested - separate template)                 │
│ └──────────────────┘                                                │
│                                                                       │
│ Root template:                                                       │
│ Resources:                                                           │
│   VPCStack:                                                         │
│     Type: AWS::CloudFormation::Stack                                │
│     Properties:                                                      │
│       TemplateURL: https://s3.amazonaws.com/mybucket/vpc.yaml      │
│       Parameters:                                                    │
│         Environment: !Ref Environment                               │
│         VpcCIDR: !Ref VpcCIDR                                       │
│                                                                       │
│   AppStack:                                                          │
│     Type: AWS::CloudFormation::Stack                                │
│     Properties:                                                      │
│       TemplateURL: https://s3.amazonaws.com/mybucket/app.yaml      │
│       Parameters:                                                    │
│         VpcId: !GetAtt VPCStack.Outputs.VpcId                      │
│         SubnetIds: !GetAtt VPCStack.Outputs.SubnetIds              │
│                                                                       │
│ ⚡ Best practices:                                                    │
│ ├── Each nested stack = one concern (VPC, DB, App)               │
│ ├── Pass values via Parameters and Outputs                       │
│ └── Reuse across environments (dev, staging, prod)               │
│                                                                       │
│ ── STACKSETS ──                                                     │
│ Deploy stacks across multiple accounts and/or regions:             │
│                                                                       │
│ Console → CloudFormation → StackSets → Create StackSet             │
│                                                                       │
│ Target accounts: [123456789, 987654321]                             │
│ Target regions: [us-east-1, eu-west-1, ap-southeast-1]             │
│                                                                       │
│ Deployment options:                                                  │
│ ├── Self-managed: Specify accounts manually                      │
│ ├── Service-managed: Deploy to AWS Organization OUs             │
│ │   → Auto-deploys to new accounts added to OU                 │
│ ├── Max concurrent: 10 accounts (or percentage)                 │
│ └── Failure tolerance: 2 accounts (or percentage)               │
│                                                                       │
│ Use cases:                                                           │
│ ├── Deploy GuardDuty across all accounts                        │
│ ├── Create IAM roles in all accounts                             │
│ ├── Deploy VPC in all regions                                    │
│ └── Enforce compliance (Config rules everywhere)                │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 7: Advanced Features

```
┌─────────────────────────────────────────────────────────────────────┐
│           ADVANCED FEATURES                                           │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Custom Resources:                                                    │
│ → Execute custom logic (Lambda) during stack operations           │
│ → For things CloudFormation doesn't natively support             │
│ Resources:                                                           │
│   CustomResourceExample:                                            │
│     Type: Custom::MyResource                                        │
│     Properties:                                                      │
│       ServiceToken: !GetAtt MyLambda.Arn                           │
│       CustomProperty: "value"                                       │
│ → Lambda receives Create/Update/Delete events                    │
│ → Must send SUCCESS/FAILED response to CF                        │
│                                                                       │
│ cfn-init & UserData:                                                │
│ → Bootstrap EC2 instances during stack creation                   │
│ → cfn-init: Install packages, create files, start services      │
│ → cfn-signal: Signal CF that instance is ready                   │
│ → CreationPolicy: Wait for N signals before proceeding          │
│                                                                       │
│ Resource Import:                                                     │
│ → Import existing AWS resources into a stack                     │
│ → Console → [Stack] → Stack actions → Import resources          │
│ → Add resource to template with DeletionPolicy: Retain          │
│ → Provide physical resource ID                                    │
│                                                                       │
│ Macros & Transforms:                                                │
│ ├── AWS::Include: Include template snippets from S3             │
│ ├── AWS::Serverless: SAM transform (Lambda, API Gateway)       │
│ └── Custom macros: Lambda-based template processing             │
│                                                                       │
│ Modules:                                                             │
│ → Reusable resource packages published to CF Registry           │
│ → Type: My::Organization::VPC::MODULE                           │
│ → Like Terraform modules but for CloudFormation                 │
│                                                                       │
│ Stack lifecycle:                                                     │
│ ├── DeletionPolicy: Retain — Keep resource when stack deleted   │
│ ├── DeletionPolicy: Snapshot — Take snapshot then delete        │
│ ├── UpdateReplacePolicy: Same options for updates               │
│ └── ⚡ Always Retain or Snapshot for databases and S3            │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 8: CLI Examples & Best Practices

```bash
# Create stack
aws cloudformation create-stack \
  --stack-name prod-web \
  --template-body file://template.yaml \
  --parameters ParameterKey=Environment,ParameterValue=production \
  --capabilities CAPABILITY_IAM CAPABILITY_NAMED_IAM \
  --tags Key=environment,Value=production \
  --enable-termination-protection

# Create change set (preview update)
aws cloudformation create-change-set \
  --stack-name prod-web \
  --change-set-name update-instance-type \
  --template-body file://template-v2.yaml \
  --parameters ParameterKey=InstanceType,ParameterValue=t3.large

# Execute change set
aws cloudformation execute-change-set \
  --stack-name prod-web \
  --change-set-name update-instance-type

# Validate template
aws cloudformation validate-template \
  --template-body file://template.yaml

# Detect drift
aws cloudformation detect-stack-drift --stack-name prod-web

# Delete stack
aws cloudformation delete-stack --stack-name dev-web

# Wait for stack completion
aws cloudformation wait stack-create-complete --stack-name prod-web

# List stack outputs
aws cloudformation describe-stacks \
  --stack-name prod-web \
  --query 'Stacks[0].Outputs'
```

```
⚡ Best Practices:
├── Always use change sets for production
├── Enable termination protection on production stacks
├── Use DeletionPolicy: Retain for databases
├── Use parameters for environment-specific values
├── Store templates in S3 (version with Git)
├── Use nested stacks for complex infrastructure
├── Run drift detection regularly
├── Never hardcode secrets (use SSM/Secrets Manager)
├── Use cfn-lint to validate templates before deploying
└── Tag everything through stack tags
```

---

## Troubleshooting: Common CloudFormation Issues

### "Stack stuck in UPDATE_ROLLBACK_FAILED"

This is the most dreaded CloudFormation state. The update failed AND the rollback also failed.

```
Solutions:
1. Console → CloudFormation → Stack → Stack actions →
   "Continue update rollback"
   Skip the resources that caused the failure

2. If that fails: Manually fix the resource outside CF,
   then retry "Continue update rollback"

3. Last resort: Delete the stack and recreate
   (Use DeletionPolicy: Retain on critical resources first!)
```

### "Template format error" or "Invalid template"

```
Common causes:
1. YAML indentation errors (use 2 spaces, never tabs)
2. Missing !Ref or !Sub syntax
3. Unsupported resource type (check spelling)
4. Validate before deploying:
   aws cloudformation validate-template --template-body file://template.yaml
```

### Common Mistakes

| Mistake | Impact | Fix |
|---------|--------|-----|
| No DeletionPolicy on databases | Data deleted if stack deleted | Add `DeletionPolicy: Snapshot` on RDS |
| Hardcoding values | Template not reusable | Use Parameters and Mappings |
| Not using ChangeSets | Unexpected destructive changes | Always preview with `create-change-set` |
| YAML tabs instead of spaces | Template validation fails | Use 2 spaces, configure editor settings |
| Circular dependencies | Stack creation fails | Use `DependsOn` or restructure resources |

---

## Quick Reference

```
CloudFormation Quick Reference:
├── What: AWS native Infrastructure as Code
├── Templates: YAML or JSON
├── Only required section: Resources
├── Key functions: !Ref, !Sub, !GetAtt, !If, !Join
├── Change sets: Preview before update (⚡ always use)
├── Drift detection: Find manual changes
├── Nested stacks: Reusable template components
├── StackSets: Multi-account + multi-region deploy
├── Stack policies: Protect critical resources from update
├── Custom resources: Lambda for unsupported operations
├── DeletionPolicy: Retain/Snapshot for databases
├── Pricing: Free (pay only for resources)
└── ⚡ vs Terraform: CF = AWS-only, Terraform = multi-cloud
```

---

## What's Next?

In **Chapter 35: CDK (Cloud Development Kit)**, we'll cover defining AWS infrastructure using familiar programming languages — constructs, stacks, synthesis, and bootstrapping.
