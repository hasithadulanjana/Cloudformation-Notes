# AWS CloudFormation — SME Exam Notes

---

## 1. Core Concepts

### What is CloudFormation?
- Infrastructure as Code (IaC) service that provisions and configures AWS resources from templates
- **Free service** — you only pay for the resources it creates
- Declarative: describe *what* you want, CloudFormation figures out *how*

### Templates
- Written in **YAML or JSON** (YAML preferred for readability)
- File extensions: `.yaml`, `.json`, `.template`, `.txt`
- Uploaded directly or via **S3** (CloudFormation auto-creates S3 buckets per region for local uploads)
- Maximum template size: **51,200 bytes** (console/CLI); **460,800 bytes** via S3 URL

### Stacks
- A deployed instance of a template — a collection of AWS resources managed as a single unit
- One template → many stacks
- Stack operations: **Create**, **Update**, **Delete**
- If creation fails → automatic **rollback** (resources deleted)

---

## 2. Template Anatomy

```yaml
AWSTemplateFormatVersion: "2010-09-09"   # Optional, always this value
Description: "My stack"                  # Optional
Metadata: {}                             # Optional
Parameters: {}                           # Optional — user inputs
Mappings: {}                             # Optional — lookup tables
Conditions: {}                           # Optional — conditional logic
Transform: []                            # Optional — macros (e.g. SAM)
Resources: {}                            # ⚠️ REQUIRED — the only mandatory section
Outputs: {}                              # Optional — exported values
```

> **Exam tip:** `Resources` is the **only required section** in a CloudFormation template.

---

## 3. Template Sections in Detail

### Parameters
- Make templates dynamic/reusable — prompt for input at stack creation/update
- Max **60 parameters** per template
- Types: `String`, `Number`, `List<Number>`, `CommaDelimitedList`, AWS-specific types (e.g. `AWS::EC2::KeyPair::KeyName`, `AWS::EC2::VPC::Id`)
- Properties: `Type`, `Default`, `AllowedValues`, `AllowedPattern`, `MinLength`, `MaxLength`, `NoEcho` (for secrets)

```yaml
Parameters:
  InstanceType:
    Type: String
    Default: t2.micro
    AllowedValues: [t2.micro, m5.large]
    Description: EC2 instance type
```

### Mappings
- Static lookup tables — fixed values known at template authoring time
- Use `Fn::FindInMap` to look up values
- Common use: region-to-AMI mapping, environment-to-size mapping

```yaml
Mappings:
  RegionMap:
    us-east-1:
      HVM64: ami-0ff8a91507f77f867
    us-west-1:
      HVM64: ami-0bdb828fd58c52235

# Usage:
# !FindInMap [RegionMap, !Ref "AWS::Region", HVM64]
```

> **Exam tip:** Use Mappings for values you know in advance. Use Parameters for user-supplied values.

### Conditions
- Control whether resources or properties are created
- Evaluated based on parameter values or pseudo-parameters
- Condition functions: `Fn::If`, `Fn::And`, `Fn::Or`, `Fn::Not`, `Fn::Equals`

```yaml
Conditions:
  IsProd: !Equals [!Ref EnvType, production]

Resources:
  ProdBucket:
    Type: AWS::S3::Bucket
    Condition: IsProd
```

### Resources
- Declare AWS resources to create
- Format: `Type: AWS::ServiceName::ResourceType`
- Key attributes: `Type`, `Properties`, `DependsOn`, `DeletionPolicy`, `UpdateReplacePolicy`, `Metadata`, `Condition`

```yaml
Resources:
  MyBucket:
    Type: AWS::S3::Bucket
    DeletionPolicy: Retain
    Properties:
      BucketName: my-bucket-2025
```

### Outputs
- Return values after stack creation (e.g. URLs, ARNs, IPs)
- Max **60 outputs** per template
- Can be **exported** for cross-stack references using `Export`
- Referenced in other stacks using `Fn::ImportValue`

```yaml
Outputs:
  BucketArn:
    Value: !GetAtt MyBucket.Arn
    Export:
      Name: !Sub "${AWS::StackName}-BucketArn"
```

---

## 4. Intrinsic Functions

| Function | Short Form | Purpose |
|---|---|---|
| `Ref` | `!Ref` | Returns value of parameter or resource ID |
| `Fn::GetAtt` | `!GetAtt` | Returns attribute of a resource |
| `Fn::Sub` | `!Sub` | String substitution with variables |
| `Fn::Join` | `!Join` | Joins values with a delimiter |
| `Fn::Split` | `!Split` | Splits a string into a list |
| `Fn::Select` | `!Select` | Selects item from a list by index |
| `Fn::FindInMap` | `!FindInMap` | Looks up value in Mappings |
| `Fn::If` | `!If` | Returns one of two values based on condition |
| `Fn::Equals` | `!Equals` | Checks if two values are equal |
| `Fn::And` | `!And` | Logical AND of conditions |
| `Fn::Or` | `!Or` | Logical OR of conditions |
| `Fn::Not` | `!Not` | Logical NOT of a condition |
| `Fn::Base64` | `!Base64` | Encodes string to Base64 (e.g. UserData) |
| `Fn::Cidr` | `!Cidr` | Generates CIDR blocks |
| `Fn::ImportValue` | `!ImportValue` | Imports output exported by another stack |
| `Fn::Transform` | — | Applies a macro transformation |

> **Exam tip:** `!Ref` on a resource returns its **physical ID**. `!Ref` on a parameter returns its **value**.

---

## 5. Pseudo Parameters

Built-in values available without declaring them:

| Pseudo Parameter | Returns |
|---|---|
| `AWS::AccountId` | Current AWS account ID |
| `AWS::Region` | Current region |
| `AWS::StackId` | Full ARN of the stack |
| `AWS::StackName` | Name of the stack |
| `AWS::Partition` | `aws`, `aws-cn`, `aws-us-gov` |
| `AWS::URLSuffix` | Domain suffix (e.g. `amazonaws.com`) |
| `AWS::NoValue` | Removes a property (used with Conditions) |
| `AWS::NotificationARNs` | List of SNS ARNs for stack notifications |

---

## 6. Resource Attributes

### DeletionPolicy
Controls what happens to a resource when its stack is deleted:

| Policy | Behaviour |
|---|---|
| `Delete` | Default — resource is deleted |
| `Retain` | Resource is kept (orphaned) |
| `Snapshot` | Creates a snapshot before deletion (RDS, EC2 volumes, ElastiCache, Redshift) |

### UpdateReplacePolicy
Controls what happens to the *old* resource when a resource must be **replaced** during an update:
- `Delete` (default), `Retain`, `Snapshot`

### DependsOn
- Forces explicit ordering when CloudFormation can't infer dependencies
- Resource will only be created after the listed resource

### CreationPolicy
- Used with EC2 instances and Auto Scaling groups
- Stack waits for a **success signal** before marking resource as created
- Works with `cfn-signal` helper script

```yaml
CreationPolicy:
  ResourceSignal:
    Count: 1
    Timeout: PT15M  # ISO 8601 duration
```

### UpdatePolicy
- Controls how CloudFormation handles updates to Auto Scaling Groups, Lambda aliases, and ElastiCache
- `AutoScalingRollingUpdate`, `AutoScalingReplacingUpdate`, `AutoScalingScheduledAction`

---

## 7. Change Sets

- **Preview** infrastructure changes before applying them — no resources modified until executed
- Two steps: **Create** (generates diff) → **Execute** (applies changes)
- Shows: resources to add, modify, or delete; replacement flag
- Replacement types: `True` (resource will be replaced), `False` (in-place update), `Conditional`

> **Exam tip:** Change sets are the recommended approach for production stack updates. They integrate naturally into CI/CD pipelines.

---

## 8. Stack Policies

- JSON policy document attached to a stack to **protect resources from unintended updates**
- Once set, all resources are denied update by default unless explicitly allowed
- Prevents accidental replacement or deletion of critical resources

```json
{
  "Statement": [
    {
      "Effect": "Deny",
      "Action": "Update:Replace",
      "Principal": "*",
      "Resource": "LogicalResourceId/ProductionDatabase"
    }
  ]
}
```

---

## 9. Rollback Behaviour

| Scenario | Default Behaviour |
|---|---|
| Stack creation fails | Rolls back and deletes all created resources |
| Stack update fails | Rolls back to last known good state |
| Rollback itself fails | Stack enters `UPDATE_ROLLBACK_FAILED` state |

- `UPDATE_ROLLBACK_FAILED` stacks can be fixed with `ContinueUpdateRollback` (skip problem resources or fix them manually first)
- `--disable-rollback` flag keeps resources on failure for debugging

---

## 10. Drift Detection

- Identifies resources modified **outside** of CloudFormation (manual console/CLI/SDK changes)
- Compares actual resource properties vs. template-defined expected state
- Drift statuses: `DRIFTED`, `IN_SYNC`, `NOT_CHECKED`, `DELETED`
- Can run on entire stack or individual resources
- Resources that don't support drift detection are marked `NOT_CHECKED`

**Valid stack statuses for drift detection:**
`CREATE_COMPLETE`, `UPDATE_COMPLETE`, `UPDATE_ROLLBACK_COMPLETE`, `UPDATE_ROLLBACK_FAILED`

> **Exam tip:** Changes made *through CloudFormation* (even directly to a member stack of a StackSet) are **not** considered drift. Only out-of-band changes count as drift.

**Drift-aware change sets (2025):** New feature that highlights drift alongside proposed changes, enabling systematic revert of drift during stack updates.

---

## 11. Nested Stacks

- A stack that creates another stack as a resource (`AWS::CloudFormation::Stack`)
- Enables **modular, reusable** template design
- Parent stack passes parameters to child stacks; child outputs can be referenced in parent
- Stored as S3 URLs in parent template
- All nested stacks count toward CloudFormation quotas

```yaml
Resources:
  NetworkStack:
    Type: AWS::CloudFormation::Stack
    Properties:
      TemplateURL: https://s3.amazonaws.com/mybucket/network.yaml
      Parameters:
        VpcCidr: "10.0.0.0/16"
```

> **Exam tip:** When deleting a parent stack, nested stacks are deleted automatically (in reverse dependency order).

---

## 12. Cross-Stack References

- Share outputs between **independent** stacks (alternative to nested stacks)
- Exporting stack uses `Export` in Outputs; importing stack uses `Fn::ImportValue`
- **Export names must be unique per region per account**
- You **cannot delete** an exporting stack while another stack is importing its values
- Circular dependencies between exports and imports are blocked by CloudFormation validation

> **Exam tip:** Nested stacks are better for tightly coupled resources. Cross-stack references are better for loosely coupled shared infrastructure (e.g., a shared VPC stack referenced by many app stacks).

---

## 13. StackSets

- Deploy stacks across **multiple accounts and regions** from a single operation
- Managed from an **administrator account**; deployed to **target accounts**

### Permission Models
| Model | How it works |
|---|---|
| **Self-managed** | Manually create IAM roles (`AWSCloudFormationStackSetAdministrationRole` and `AWSCloudFormationStackSetExecutionRole`) in each account |
| **Service-managed** | Uses AWS Organizations — no manual IAM setup needed; supports auto-deployment to new accounts |

### Key Concepts
- **Stack instance:** Reference to a stack in a target account/region (exists even if stack creation failed)
- **Deployment options:** `MaxConcurrentCount/Percentage`, `FailureToleranceCount/Percentage`
- **Deployment ordering (2025):** Specify up to 10 dependencies per stack instance using `DependsOn` in `AutoDeployment` config
- Drift detection works at StackSet level (runs on each member stack instance)

> **Exam tip:** Changes made directly to a member stack (not via the StackSet) are **not** considered StackSet drift.

---

## 14. CloudFormation Registry

- Manage **extensions**: resource types, modules, and hooks
- Supports public (AWS and third-party) and private extensions
- Advantages over custom resources: supports CRUDL operations, drift detection, schema validation

### Resource Types
- Public: `AWS::*` types (first-party)
- Private: custom types registered via CloudFormation CLI
- Third-party: available in CloudFormation Public Registry (e.g. MongoDB, Datadog)

### Modules
- Reusable template fragments — encapsulate common patterns
- Registered in Registry, referenced like a resource type

---

## 15. Custom Resources

- Extend CloudFormation to provision **non-AWS resources** or run **custom logic**
- Backed by **Lambda** or **SNS**
- CloudFormation sends a request (Create/Update/Delete) with a `ResponseURL` (pre-signed S3 URL)
- Your function must respond with `SUCCESS` or `FAILED` to that URL

```yaml
Resources:
  MyCustomResource:
    Type: Custom::MyThing
    Properties:
      ServiceToken: !GetAtt MyLambdaFunction.Arn
      MyParam: "value"
```

> **Exam tip:** If a custom resource Lambda times out or fails to respond to the pre-signed S3 URL, the stack will wait until the `Timeout` expires, then fail.

---

## 16. CloudFormation Hooks

- Run **proactive validations** before resource changes are provisioned
- Intercept `CREATE`, `UPDATE`, `DELETE` operations
- Handlers: `preCreate`, `preUpdate`, `preDelete`
- Return `SUCCESS` or `FAILED` (with optional failure modes: `FAIL`, `WARN`)
- **Managed proactive controls (2025):** AWS-curated controls from Control Tower catalog — no custom code needed; supports `warn` mode for testing before enforcing

---

## 17. Helper Scripts (EC2)

Installed on EC2 AMIs, used to configure instances during stack creation:

| Script | Purpose |
|---|---|
| `cfn-init` | Reads `AWS::CloudFormation::Init` metadata; installs packages, creates files, starts services |
| `cfn-signal` | Sends success/failure signal back to CloudFormation (used with `CreationPolicy`) |
| `cfn-get-metadata` | Retrieves metadata blocks for inspection |
| `cfn-hup` | Daemon that detects metadata changes and re-runs `cfn-init` (enables in-place config updates) |

```yaml
Metadata:
  AWS::CloudFormation::Init:
    config:
      packages:
        yum:
          httpd: []
      services:
        sysvinit:
          httpd:
            enabled: true
            ensureRunning: true
```

> **Exam tip:** `cfn-init` does **not** require AWS credentials. `cfn-signal` must be called to prevent stack timeout when using `CreationPolicy`.

---

## 18. WaitConditions

- Pause stack creation until an external process signals completion
- `AWS::CloudFormation::WaitCondition` + `AWS::CloudFormation::WaitConditionHandle`
- Replaced by `CreationPolicy` for EC2/ASG use cases (prefer `CreationPolicy`)
- Still useful when the signal comes from outside AWS (e.g. on-prem systems)

---

## 19. Stack Status Reference

| Status | Meaning |
|---|---|
| `CREATE_IN_PROGRESS` | Stack creation underway |
| `CREATE_COMPLETE` | Stack created successfully |
| `CREATE_FAILED` | Stack creation failed |
| `UPDATE_IN_PROGRESS` | Update underway |
| `UPDATE_COMPLETE` | Update succeeded |
| `UPDATE_ROLLBACK_FAILED` | Rollback failed — requires manual intervention |
| `DELETE_IN_PROGRESS` | Deletion underway |
| `DELETE_FAILED` | Deletion failed (often due to non-empty S3 buckets or retained dependencies) |
| `ROLLBACK_IN_PROGRESS` | Rolling back after failed create |
| `ROLLBACK_COMPLETE` | Rolled back — stack exists but has no resources |

---

## 20. Service Limits (Key Quotas)

| Resource | Limit |
|---|---|
| Stacks per account per region | 2,000 |
| Parameters per template | 60 |
| Outputs per template | 60 |
| Mappings per template | 200 |
| Resources per template | 500 |
| Template size (inline) | 51,200 bytes |
| Template size (S3) | 460,800 bytes |

---

## 21. IAM & Security

- CloudFormation calls AWS APIs **on your behalf** — you need permissions for those resource types
- Use **IAM service roles** (`--role-arn`) to grant CloudFormation a specific role (least-privilege)
- `CAPABILITY_IAM` / `CAPABILITY_NAMED_IAM` — required acknowledgement when template creates IAM resources
- `CAPABILITY_AUTO_EXPAND` — required when template uses macros or Transform (e.g. SAM)

---

## 22. AWS SAM (Serverless Application Model)

- Extension of CloudFormation for serverless resources
- Uses the `Transform: AWS::Serverless-2016-10-31` declaration
- SAM resource types (`AWS::Serverless::Function`, `AWS::Serverless::Api`, etc.) are transformed into standard CloudFormation resources at deploy time

---

## 23. CDK (Cloud Development Kit)

- Programmatic IaC using Python, TypeScript, Java, etc.
- Synthesises down to CloudFormation templates
- CDK apps deploy as CloudFormation stacks — all CloudFormation features apply
- `cdk synth` generates template; `cdk deploy` deploys it

---

## 24. Infrastructure Composer (Visual Designer)

- Drag-and-drop visual tool for creating and editing CloudFormation templates
- Architecture planning, template modernisation, stakeholder communication
- Integrated into the CloudFormation console

---

## 25. IaC Generator

- Generate CloudFormation templates from **existing, manually-created AWS resources**
- Useful for bringing unmanaged resources under CloudFormation control
- Starting point for template creation — review and refine the generated template

---

## 26. Best Practices

### Template Design
- Use Parameters for environment-specific values; never hard-code account IDs or region strings
- Use `!Sub` instead of `!Join` for readability in string construction
- Use Mappings for known, static values (e.g. AMI IDs per region)
- Use AWS-specific parameter types (e.g. `AWS::EC2::VPC::Id`) for built-in validation
- Validate templates with `aws cloudformation validate-template` and `cfn-lint`

### Stack Management
- Always use **Change Sets** before updating production stacks
- Apply **Stack Policies** to protect critical resources
- Set appropriate **DeletionPolicy** (`Retain` or `Snapshot`) on stateful resources
- Check account limits before deploying large stacks

### Multi-Account/Region
- Use StackSets with **service-managed permissions** for AWS Organizations deployments
- Use separate stacks per environment; share resources via cross-stack references
- Export names must be globally unique within account/region — use `!Sub` with stack name prefix

### Modular Design
- Use Nested Stacks for tightly coupled resources that always deploy together
- Use Cross-Stack References for shared, independently-managed infrastructure
- Store reusable child templates in versioned S3 paths

---

## 27. CI/CD Integration

- `aws cloudformation deploy` combines create-change-set + execute-change-set in one command
- Integrate with **AWS CodePipeline**, GitHub Actions, Jenkins
- Use `--no-execute-changeset` to create a change set for review without applying it
- Use `cfn-lint` in CI pipelines for pre-deployment linting
- SAM CLI provides local testing with `sam local invoke` / `sam local start-api`

---

## 28. Quick-Reference: Common Exam Scenarios

| Scenario | Answer |
|---|---|
| Deploy same template to 50 accounts | StackSets with service-managed permissions |
| Preview changes before applying to prod | Change Set |
| Protect RDS instance from deletion | `DeletionPolicy: Retain` or `Snapshot` |
| Wait for app to be ready on EC2 | `CreationPolicy` + `cfn-signal` |
| Share VPC ID between stacks | Outputs + Export + `Fn::ImportValue` |
| Detect manual config changes | Drift Detection |
| Custom logic during provisioning | Custom Resource (Lambda-backed) |
| Enforce compliance on all resources | CloudFormation Hooks |
| Template uses SAM resources | Add `CAPABILITY_AUTO_EXPAND` |
| Template creates IAM roles | Add `CAPABILITY_NAMED_IAM` |
| Stack stuck in UPDATE_ROLLBACK_FAILED | `ContinueUpdateRollback` |
| Generate template from existing resources | IaC Generator |

---

*Sources: AWS CloudFormation User Guide, AWS Best Practices Guide, AWS 2025 CloudFormation Year in Review*
