# Chapter 35: CDK (Cloud Development Kit)

---

## Table of Contents

- [Overview](#overview)
- [Part 1: CDK Fundamentals](#part-1-cdk-fundamentals)
- [Part 2: Getting Started (Full Walkthrough)](#part-2-getting-started-full-walkthrough)
- [Part 3: Constructs (L1, L2, L3)](#part-3-constructs-l1-l2-l3)
- [Part 4: Stacks, Apps & Environments](#part-4-stacks-apps--environments)
- [Part 5: Synthesis, Diff & Deploy](#part-5-synthesis-diff--deploy)
- [Part 6: Common CDK Patterns](#part-6-common-cdk-patterns)
- [Part 7: Testing CDK Code](#part-7-testing-cdk-code)
- [Part 8: Best Practices](#part-8-best-practices)
- [Quick Reference](#quick-reference)
- [What's Next?](#whats-next)

---

## Overview

### What is CDK? Why Use It Instead of CloudFormation?

CloudFormation templates are powerful but verbose — a simple VPC with subnets and an EC2 instance can be 200+ lines of YAML. If you're a developer, writing YAML all day doesn't feel natural.

**AWS CDK (Cloud Development Kit)** lets you define infrastructure using **real programming languages** — TypeScript, Python, Java, C#, or Go. Behind the scenes, CDK converts your code into a CloudFormation template and deploys it. You get the best of both worlds: the power of CloudFormation with the expressiveness of a programming language.

**Think of it this way:**
- **CloudFormation** = Writing instructions in plain text (YAML): "Create a VPC. Create a subnet. Create a security group..."
- **CDK** = Writing instructions in code: `new Vpc(this, 'MyVpc')` — one line that creates a VPC with all best-practice defaults

**When to use CDK vs CloudFormation:**
- You're a developer comfortable with TypeScript/Python → **CDK**
- Your team already has a large CloudFormation library → **Keep CloudFormation**
- You want high-level abstractions ("give me a production-ready VPC") → **CDK**
- You prefer a third-party tool with multi-cloud support → **Terraform**

AWS CDK lets you define cloud infrastructure using familiar programming languages (TypeScript, Python, Java, C#, Go). It synthesizes into CloudFormation templates and deploys via CloudFormation.

```
What you'll learn:
├── CDK Fundamentals (vs CloudFormation vs Terraform)
├── Project setup & bootstrapping
├── Constructs (L1, L2, L3 — abstraction levels)
├── Stacks, Apps, and Environments
├── Synthesis, diff, and deployment
├── Common infrastructure patterns
├── Testing CDK code
└── Best practices
```

---

## Part 1: CDK Fundamentals

```
┌─────────────────────────────────────────────────────────────────────┐
│           HOW CDK WORKS                                               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ ┌──────────────┐   ┌───────────┐   ┌──────────────┐   ┌─────────┐│
│ │ CDK App      │──►│ cdk synth │──►│ CloudFormation│──►│ AWS     ││
│ │ (TypeScript, │   │           │   │ Template      │   │Resources││
│ │  Python,     │   │ Synthesis │   │ (YAML/JSON)   │   │         ││
│ │  Java, etc.) │   └───────────┘   └──────────────┘   └─────────┘│
│ └──────────────┘                                                   │
│                                                                       │
│ CDK vs CloudFormation vs Terraform:                                 │
│ ┌──────────────────┬──────────────┬──────────────┬───────────────┐│
│ │                  │ CDK          │CloudFormation│ Terraform     ││
│ ├──────────────────┼──────────────┼──────────────┼───────────────┤│
│ │ Language         │ TypeScript,  │ YAML/JSON    │ HCL           ││
│ │                  │ Python, etc. │              │               ││
│ │ Abstraction      │ High (L2/L3) │ Low          │ Medium        ││
│ │ Loops/conditions │ Native code  │ Limited      │ HCL syntax    ││
│ │ Testing          │ Unit tests   │ cfn-lint     │ Terratest     ││
│ │ Multi-cloud      │ AWS only     │ AWS only     │ Yes           ││
│ │ State mgmt       │ CloudFormation│CloudFormation│ State file   ││
│ │ Generates        │ CF template  │ —            │ —             ││
│ └──────────────────┴──────────────┴──────────────┴───────────────┘│
│                                                                       │
│ Key advantages of CDK:                                              │
│ ├── Use real programming languages (IDE support, types)          │
│ ├── High-level constructs (L2) set smart defaults               │
│ ├── L3 patterns = entire architectures in a few lines           │
│ ├── Loops, conditionals, functions — native code                │
│ ├── Unit testable infrastructure                                 │
│ └── Generates CloudFormation (reliable, battle-tested backend)  │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 2: Getting Started (Full Walkthrough)

```bash
# Prerequisites
node --version  # Node.js 18+ required
npm install -g aws-cdk  # Install CDK CLI globally

# Verify
cdk --version

# Create a new CDK project
mkdir my-cdk-app && cd my-cdk-app

# Initialize project (TypeScript)
cdk init app --language typescript
# Other languages: python, java, csharp, go

# Project structure created:
# my-cdk-app/
# ├── bin/
# │   └── my-cdk-app.ts          ← App entry point
# ├── lib/
# │   └── my-cdk-app-stack.ts    ← Stack definition
# ├── test/
# │   └── my-cdk-app.test.ts     ← Tests
# ├── cdk.json                    ← CDK config
# ├── package.json
# └── tsconfig.json

# Bootstrap (one-time per account+region)
cdk bootstrap aws://123456789012/us-east-1
# → Creates CDKToolkit stack with:
#   - S3 bucket (for assets/templates)
#   - ECR repository (for Docker assets)
#   - IAM roles (for CDK deployment)
```

```
┌─────────────────────────────────────────────────────────────────────┐
│           CDK BOOTSTRAPPING                                           │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Bootstrap creates a CloudFormation stack "CDKToolkit" with:        │
│ ├── S3 bucket: Stores synthesized templates + file assets        │
│ ├── ECR repo: Stores Docker image assets                         │
│ ├── IAM roles:                                                    │
│ │   ├── File publishing role                                    │
│ │   ├── Image publishing role                                   │
│ │   ├── CloudFormation execution role                           │
│ │   ├── Deploy action role                                      │
│ │   └── Lookup role                                              │
│ └── SSM parameter: Bootstrap version tracker                    │
│                                                                       │
│ ⚡ Must bootstrap each account+region pair before deploying       │
│ ⚠️ Bootstrap once per account+region (not per project)            │
│                                                                       │
│ Cross-account bootstrap:                                            │
│ cdk bootstrap aws://PROD_ACCOUNT/us-east-1 \                     │
│   --trust DEV_ACCOUNT \                                            │
│   --cloudformation-execution-policies arn:aws:iam::aws:policy/... │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 3: Constructs (L1, L2, L3)

```
┌─────────────────────────────────────────────────────────────────────┐
│           CONSTRUCT LEVELS                                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ L1 (CFN Resources) — Lowest level:                                  │
│ ├── 1:1 mapping to CloudFormation resources                      │
│ ├── Prefix: Cfn* (e.g., CfnBucket, CfnInstance)                │
│ ├── No smart defaults — you set everything                      │
│ └── Use when L2 doesn't support a feature                       │
│                                                                       │
│ new s3.CfnBucket(this, 'Bucket', {                                │
│   bucketName: 'my-bucket',                                        │
│   versioningConfiguration: { status: 'Enabled' }                  │
│ });                                                                 │
│                                                                       │
│ ─────────────────────────────────────────────────────────────────  │
│                                                                       │
│ L2 (Curated Constructs) — ⚡ Most commonly used:                    │
│ ├── Smart defaults (encryption, logging auto-configured)         │
│ ├── Convenient methods (grant(), addToPolicy(), etc.)           │
│ ├── Type-safe properties                                          │
│ └── 80% of what you need                                         │
│                                                                       │
│ const bucket = new s3.Bucket(this, 'Bucket', {                    │
│   versioned: true,                                                 │
│   encryption: s3.BucketEncryption.S3_MANAGED,                    │
│   removalPolicy: cdk.RemovalPolicy.RETAIN,                       │
│ });                                                                 │
│ bucket.grantRead(myLambda);  // Auto-creates IAM policy          │
│                                                                       │
│ ─────────────────────────────────────────────────────────────────  │
│                                                                       │
│ L3 (Patterns) — Highest level:                                      │
│ ├── Multi-resource architectures in one construct               │
│ ├── Opinionated, batteries-included                              │
│ └── Examples: LoadBalancedFargateService, LambdaRestApi         │
│                                                                       │
│ new ecs_patterns.ApplicationLoadBalancedFargateService(           │
│   this, 'WebService', {                                             │
│     cluster,                                                        │
│     taskImageOptions: {                                              │
│       image: ecs.ContainerImage.fromRegistry('nginx'),            │
│     },                                                               │
│     desiredCount: 3,                                                │
│     publicLoadBalancer: true,                                       │
│   }                                                                  │
│ );                                                                   │
│ // Creates: ALB + Target Group + ECS Service + Task Def          │
│ //   + Security Groups + IAM Roles + CloudWatch Logs             │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 4: Stacks, Apps & Environments

```typescript
// bin/my-cdk-app.ts — App entry point
import * as cdk from 'aws-cdk-lib';
import { VpcStack } from '../lib/vpc-stack';
import { AppStack } from '../lib/app-stack';
import { DatabaseStack } from '../lib/database-stack';

const app = new cdk.App();

// Environment configuration
const envUSA = { account: '123456789012', region: 'us-east-1' };
const envEU  = { account: '123456789012', region: 'eu-west-1' };

// Shared VPC stack
const vpcStack = new VpcStack(app, 'VpcStack', { env: envUSA });

// Application stack (depends on VPC)
const appStack = new AppStack(app, 'AppStack', {
  env: envUSA,
  vpc: vpcStack.vpc,       // Pass resources between stacks
});
appStack.addDependency(vpcStack);

// Database stack
const dbStack = new DatabaseStack(app, 'DatabaseStack', {
  env: envUSA,
  vpc: vpcStack.vpc,
});
dbStack.addDependency(vpcStack);
```

```typescript
// lib/vpc-stack.ts — Stack definition
import * as cdk from 'aws-cdk-lib';
import * as ec2 from 'aws-cdk-lib/aws-ec2';
import { Construct } from 'constructs';

export class VpcStack extends cdk.Stack {
  public readonly vpc: ec2.Vpc;  // Export for other stacks

  constructor(scope: Construct, id: string, props?: cdk.StackProps) {
    super(scope, id, props);

    this.vpc = new ec2.Vpc(this, 'MainVpc', {
      maxAzs: 3,
      natGateways: 1,
      subnetConfiguration: [
        { name: 'Public', subnetType: ec2.SubnetType.PUBLIC, cidrMask: 24 },
        { name: 'Private', subnetType: ec2.SubnetType.PRIVATE_WITH_EGRESS, cidrMask: 24 },
        { name: 'Isolated', subnetType: ec2.SubnetType.PRIVATE_ISOLATED, cidrMask: 24 },
      ],
    });
  }
}
```

---

## Part 5: Synthesis, Diff & Deploy

```bash
# Synthesize (generate CloudFormation template)
cdk synth
# → Output: cdk.out/ directory with CF templates

# View specific stack template
cdk synth VpcStack

# Diff (compare deployed vs local)
cdk diff
# → Shows what will change (add/remove/modify)
# → ⚡ ALWAYS run diff before deploy

# Deploy single stack
cdk deploy AppStack

# Deploy all stacks
cdk deploy --all

# Deploy with approval prompts for IAM/security changes
cdk deploy --require-approval broadening
# → never: No prompts
# → any-change: Prompt for any security change
# → broadening: Prompt only for new permissions (⚡ default)

# Deploy with context values
cdk deploy -c environment=production

# Destroy stack
cdk destroy AppStack
# ⚠️ Resources with RemovalPolicy.RETAIN won't be deleted

# List stacks
cdk list
```

---

## Part 6: Common CDK Patterns

```typescript
// Pattern 1: Lambda + API Gateway
import * as lambda from 'aws-cdk-lib/aws-lambda';
import * as apigw from 'aws-cdk-lib/aws-apigateway';

const fn = new lambda.Function(this, 'Handler', {
  runtime: lambda.Runtime.NODEJS_18_X,
  handler: 'index.handler',
  code: lambda.Code.fromAsset('lambda'),
  environment: { TABLE_NAME: table.tableName },
  timeout: cdk.Duration.seconds(30),
  memorySize: 256,
});

const api = new apigw.RestApi(this, 'Api', {
  restApiName: 'My API',
});
api.root.addResource('items').addMethod('GET', new apigw.LambdaIntegration(fn));

// Pattern 2: ECS Fargate Service
import * as ecs from 'aws-cdk-lib/aws-ecs';
import * as ecs_patterns from 'aws-cdk-lib/aws-ecs-patterns';

new ecs_patterns.ApplicationLoadBalancedFargateService(this, 'Service', {
  cluster: new ecs.Cluster(this, 'Cluster', { vpc }),
  desiredCount: 3,
  taskImageOptions: {
    image: ecs.ContainerImage.fromAsset('./app'),
    containerPort: 8080,
    environment: { NODE_ENV: 'production' },
  },
  publicLoadBalancer: true,
});

// Pattern 3: DynamoDB + Lambda with auto IAM
const table = new dynamodb.Table(this, 'Table', {
  partitionKey: { name: 'pk', type: dynamodb.AttributeType.STRING },
  billingMode: dynamodb.BillingMode.PAY_PER_REQUEST,
  removalPolicy: cdk.RemovalPolicy.RETAIN,
});
table.grantReadWriteData(fn);  // ⚡ Auto-creates least-privilege IAM
```

---

## Part 7: Testing CDK Code

```typescript
// test/app-stack.test.ts
import * as cdk from 'aws-cdk-lib';
import { Template, Match } from 'aws-cdk-lib/assertions';
import { AppStack } from '../lib/app-stack';

describe('AppStack', () => {
  const app = new cdk.App();
  const stack = new AppStack(app, 'TestStack');
  const template = Template.fromStack(stack);

  test('creates a DynamoDB table', () => {
    template.hasResourceProperties('AWS::DynamoDB::Table', {
      BillingMode: 'PAY_PER_REQUEST',
      KeySchema: Match.arrayWith([
        Match.objectLike({ AttributeName: 'pk', KeyType: 'HASH' }),
      ]),
    });
  });

  test('creates exactly 2 Lambda functions', () => {
    template.resourceCountIs('AWS::Lambda::Function', 2);
  });

  test('Lambda has correct environment variables', () => {
    template.hasResourceProperties('AWS::Lambda::Function', {
      Environment: {
        Variables: Match.objectLike({
          NODE_ENV: 'production',
        }),
      },
    });
  });

  test('S3 bucket has encryption enabled', () => {
    template.hasResourceProperties('AWS::S3::Bucket', {
      BucketEncryption: Match.objectLike({
        ServerSideEncryptionConfiguration: Match.anyValue(),
      }),
    });
  });
});
```

---

## Part 8: Best Practices

```
┌─────────────────────────────────────────────────────────────────────┐
│           CDK BEST PRACTICES                                          │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Project structure:                                                   │
│ ├── One app, multiple stacks (VPC, App, DB, Pipeline)            │
│ ├── Separate stacks for resources with different lifecycles      │
│ ├── Use interfaces (props) to pass between stacks               │
│ └── Keep constructs small and focused                             │
│                                                                       │
│ Development:                                                         │
│ ├── Use L2 constructs (smart defaults)                           │
│ ├── Always set RemovalPolicy for stateful resources             │
│ ├── Use cdk diff before every deploy                             │
│ ├── Write unit tests (assertions library)                       │
│ ├── Use cdk.context.json for cached lookups                     │
│ └── Use cdk.json for project configuration                      │
│                                                                       │
│ Security:                                                            │
│ ├── Let CDK generate IAM policies (grant* methods)              │
│ ├── Don't hardcode secrets (use Secrets Manager)                │
│ ├── Use --require-approval broadening                           │
│ └── Review synthesized CF template before deploy                │
│                                                                       │
│ Production:                                                          │
│ ├── Pin CDK version in package.json                              │
│ ├── Use CDK Pipelines for CI/CD                                  │
│ ├── Tag all resources via Stack tags                             │
│ ├── Enable termination protection on stacks                     │
│ └── Use cdk.context.json to avoid non-deterministic lookups     │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Quick Reference

```
CDK Quick Reference:
├── What: Define AWS infra in TypeScript/Python/Java/C#/Go
├── Generates: CloudFormation templates (cdk synth)
├── Constructs: L1 (CFN), L2 (curated ⚡), L3 (patterns)
├── Commands: init, synth, diff, deploy, destroy
├── Bootstrap: cdk bootstrap (once per account+region)
├── Testing: aws-cdk-lib/assertions (unit tests)
├── grant*: Auto-generates least-privilege IAM
├── RemovalPolicy: RETAIN for databases, DESTROY for dev
├── CDK Pipelines: Self-mutating CI/CD pipeline
└── ⚡ Always run cdk diff before deploy
```

---

## What's Next?

In **Chapter 36: CloudWatch**, we'll begin the Monitoring & Logging section — covering metrics, alarms, dashboards, logs, and CloudWatch Synthetics.
