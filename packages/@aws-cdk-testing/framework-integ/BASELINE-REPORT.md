# aws-cdk integ snapshots: real-AWS baseline and AWS Emulator comparison

2026-11-21. Real AWS is the source of truth. Results are on
[iankhou/aws-cdk `resolved-snapshots`](https://github.com/iankhou/aws-cdk/tree/resolved-snapshots) (last commit
`635b0bb649`), checked with [iankhou/cdk-integ-assertions `fix/comparisons`](https://github.com/iankhou/cdk-integ-assertions/tree/fix/comparisons)
(`b9a247a`).

## What was run

- **Snapshots:** all 1,520 committed `*.js.snapshot` directories under `packages/@aws-cdk-testing/framework-integ/test`,
  from upstream aws/aws-cdk `main` at `d7508813b9`.
- **Real AWS:** each one was deployed as committed (no synthesis) and its template resolved with the toolkit-lib
  `evaluate` action. The deployed resources were then checked with the cdk-integ-assertions handlers, and the test was
  destroyed.
  - It ran in 4 regions (us-east-1, us-east-2, us-west-2, eu-west-1) of a test account, 2 tests at a time per region.
  - Tests that share stack names ran in separate rounds, so they never deployed at the same time.
  - A few tests were rerun in a second test account (test-2) or in us-west-2 because of quota and capacity limits.
- **AWS Emulator:** version 1.0.512, running natively. It ran the 49 tests rerun with the new checker, from a copy of
  their snapshots, in us-east-1, the emulator's only region.

## Real AWS: results

| Result | Tests |
|---|---|
| Deployed and checked | 1,311 |
| Deploy failed | 208 |
| No manifest upstream (`aws-rds/integ.serverless-cluster-scaling`) | 1 |

| pass | fail | missing | unresolved | gap | no handler | error |
|---|---|---|---|---|---|---|
| 64,321 | 20 | 188 | 2,559 | 27 | 3,728 | 1 |

**Deploy failures (208).** None is caused by this run. Each is about what the test needs, or about how it is
committed:

| Reason | Tests |
|---|---|
| Dummy environment or other-account ids committed in the snapshot (`test-region`, `123456789012`, ...) | 36 |
| Needs a hosted zone, domain or certificate | 28 |
| Deprecated or unsupported runtime, version or feature | 23 |
| Custom resource or assertion stack failed | 18 |
| Asset directory not in upstream git (uploaded as an empty zip) | 13 |
| Asset file in the snapshot not in upstream git | 11 |
| Asset `NoSuchKey` | 9 |
| Account or capacity limits (GPU node groups, t4g capacity, load balancer capacity reservation, AWS Config delivery channel) | 8 |
| No CodeBuild access in the account | 7 |
| External integrations (Slack, Pinpoint, WorkMail, OIDC) | 6 |
| Other, one-off (invalid principals, X-Ray settings, route conflicts, fixed names, ...) | 49 |

**Checker fixes.** The first pass had 92 fails. 72 of them were the checker being stricter than AWS, and
`fix/comparisons` removes those 72:
- list order (CloudFront methods and origins, alarm actions, scheduled actions, binary media types);
- defaults AWS adds (Events targets, Step Functions logging and encryption, S3 lifecycle filters, SNS delivery policy,
  Lambda destinations and file systems);
- the same value in another form:
  - key names: ECS `efsVolumeConfiguration`/`properties`/`awsvpcConfiguration`, Kafka `KAFKA_BOOTSTRAP_SERVERS`;
  - API Gateway method-setting path keys;
  - ARNs against ids: ECS cluster KMS keys, ECS service task definitions, KMS account principals;
  - the OAI principal in an S3 bucket policy.

Before the rerun, every recorded check was replayed offline through the new comparisons. All 194 earlier passes
still passed, and an altered expected value was still caught for each.

**The 20 remaining fails:**

| Fails | What | Cause |
|---|---|---|
| 6 | `AWS::ApiGateway::Account` `CloudWatchRoleArn` | One setting per region per account; tests running at the same time overwrite it |
| 3 | `AWS::DynamoDB::GlobalTable` `Replicas` | Checked from the wrong region (see the region gap below) |
| 3 | `AWS::IAM::Role` `ManagedPolicyArns` | Real: an extra managed policy is attached (`replace-depends-on-test`, `cdk-ecs-fluentd`, `integ-ecs-al2023-ami`) |
| 2 | `AWS::ECS::ClusterCapacityProviderAssociations` `CapacityProviders` | Real: an extra managed-instances capacity provider |
| 2 | `AWS::Events::Rule` `State` | Real: `ENABLED` expected, `DISABLED` deployed (`firehose-event-target`, `aws-ecs-integ-fargate`) |
| 2 | `AWS::ApiGateway::Method` `MethodResponses` | Real: the 200 response has `responseParameters` instead of `responseModels` |
| 1 | `AWS::DynamoDB::Table` `DeletionProtectionEnabled` | Real: `true` expected, `false` deployed (`deletion-protection-stack`) |
| 1 | `AWS::IAM::Role` `AssumeRolePolicyDocument` | Real: an extra trust statement, added after deploy (`emr-containers-start-job-run`) |

**Gaps in how the checks are run (in `baseline.js`):**
- **Region gap:** 183 of the 188 missing checks come from 32 stacks in 22 tests that are pinned to a region other than
  the worker's. `baseline.js` deploys them correctly but checks them in the worker's region.
- **Dependency stacks:** stacks pulled in as dependencies but not listed in `integ.json` (35 snapshots) are deployed
  but not destroyed.
- **The other 5 missing checks:** buckets the 2 `auto-delete-objects` tests remove on purpose, and an AppSync metric
  filter.

**Destroy problems seen on real AWS** (all cleaned up): ECS-on-EC2 stacks hang on delete. The container instances
lose their agent connection before the service drains, and capacity-provider scale-in protection then blocks the
auto-scaling group delete. Fargate profiles, EMR-managed security groups, ALB-controller load balancers and Synthetics
network interfaces also outlive their stacks.

## AWS Emulator: all 1,520 snapshots

AWS Emulator 1.0.512, running natively. All snapshots were deployed as committed and checked with the same checker
(`fix/comparisons`), from a copy of the snapshots without the real-AWS results.
- **Region:** us-east-1 only, because the emulator doesn't keep regions or accounts apart.
- **Batches:** tests ran in the same collision-free rounds, in batches of 25.
- **Fresh state for each batch:** the emulator restarted with an empty state and CDK was bootstrapped again, because
  the emulator keeps at most 64 MiB of objects per S3 bucket. In a first attempt without restarts, CDK's asset bucket
  filled after about 560 tests, and every later deploy failed.
- **Solo reruns:** 46 tests that hit the cap only because of other tests in their batch were rerun with a fresh
  emulator each.

### Deploy outcomes against real AWS

| Real AWS / Emulator | Tests |
|---|---|
| Deployed on both | 633 |
| Deployed on AWS, failed on the emulator | 678 |
| Failed on both | 183 |
| Failed on AWS, deployed on the emulator | 25 |
| No manifest upstream | 1 |

Of the 1,311 tests that deploy on AWS, **48% also deploy on the emulator**.

**Why 678 tests that deploy on AWS fail on the emulator:**

| Tests | Emulator behavior |
|---|---|
| 212 | Custom resources and CDK assertion stacks fail: Lambda function runs aren't ready (`DeployAssert`, `Custom::VpcRestrictDefaultSG`, cross-region readers, log-retention, bucket notifications, ...) |
| 81 | Request validation stricter than AWS: for example `Value null at ...` in CloudFront origins where AWS applies defaults, API Gateway access-log format rules, `expected string, got boolean` |
| 69 | CloudFormation doesn't know the resource type (for example `AWS::MediaPackageV2::*`, `AWS::S3Files::*`) |
| 53 | Behavior the emulator reports it doesn't support (for example `RestApi` `BodyS3Location`, `Lambda::Permission` `InvokedViaFunctionUrl`) |
| 42 | RDS rejects current engine versions (`aurora-mysql 8.0.mysql_aurora.3.12.0`, `aurora-postgresql 18.6`, `mysql 8.4.10`) |
| 40 | Docker image assets can't be pushed: the ECR registry host (`<account>.dkr.ecr.<region>.amazonaws.com`) isn't the emulator's endpoint (`no basic auth credentials`) |
| 31 | CloudFormation template errors on attributes AWS supports (`resource ... does not support` a `Fn::GetAtt`) |
| 17 | A resource the stack just created isn't found (IAM groups, load balancers) |
| 6 | Operation not implemented (`NotImplemented`) |
| 2 | The test's own assets are over the 64 MiB per-bucket cap |
| 125 | Other, one-off |

**25 tests deploy only on the emulator.** Each was rejected by AWS for a reason the emulator doesn't check:
- deprecated runtimes (Synthetics, Node.js 14, Glue versions);
- an invalid Elasticsearch TLS policy;
- invalid Bedrock model ids;
- invalid principals;
- certificates that don't exist;
- missing CodeBuild source tokens;
- unknown third-party CodePipeline action types.

So a green deploy on the emulator doesn't mean the stack is valid on AWS.

**A hang that never ends.** In `aws-ec2/integ.vpc-egress-only-igw-feature-flag`, the emulator's CloudFormation
rejected a subnet with `Template error: resource ... does not support` an attribute. It then left the stack in
`ROLLBACK_IN_PROGRESS` for 22 hours, until the deploy was stopped.

### Checks on the 633 tests that deploy on both

430 of the 633 (68%) have every check identical to real AWS. For the 23,813 checks that pass on AWS:

| On the emulator | Checks | Main causes |
|---|---|---|
| pass | 22,518 (94.6%) | |
| error | 405 | ECS `DescribeServices` returns timestamps as integer strings; the JavaScript SDK throws `Epoch timestamps must be expressed as floating point numbers` (all ECS service checks in 53 tests) |
| missing | 136 | CloudFormation reports `CREATE_COMPLETE` for resources that don't exist: `S3::BucketPolicy` (`NoSuchBucket`, 48 tests). Physical ids in another shape: `ApiGateway::Method` (no resource id, 27 tests) and `SNS::TopicPolicy` (not a topic ARN, 8 tests) |
| not checked | 489 | Property checks skipped after one of the `missing` resources above |
| unresolved | 235 | The `evaluate` action can't resolve values from the emulator's stacks: `Fn::GetAtt`/`Ref` in IAM policies (66 tests), ECS network configuration, Lambda permission source ARNs, API Gateway parent ids |
| fail | 30 | The deployed value differs from AWS (below) |

**Fails on the emulator where AWS passes:**
- S3 bucket `LoggingConfiguration` isn't stored; `GetBucketLogging` returns nothing (3 tests).
- `GetBucketLifecycleConfiguration` returns `Filter: {}` where AWS returns `Filter: { Prefix: "" }` (2 tests).
- Step Functions `DefinitionString` comes back changed (5 tests).
- `AWS::URLSuffix` resolves to `127.0.0.1`, so IAM service principals become `ec2.127.0.0.1` instead of
  `ec2.amazonaws.com` (3 roles).
- KMS stores a bare account-id principal as given; AWS stores the account's root ARN.
- `IAM::ManagedPolicy` `Description` is `""` instead of absent.
- One each in CloudWatch dashboard bodies, S3 bucket encryption and an IAM policy document.
- The `ApiGateway::Account` `CloudWatchRoleArn` differences (5 one way, 3 the other) are the same per-account
  setting being overwritten by concurrent tests, as on AWS.

### Takeaway

The emulator runs about half of these tests, and where it does, 95% of the checks that pass on AWS also pass. The
gaps that block the most tests, in order:
1. Lambda-backed custom resources (212 tests);
2. request validation that's stricter than AWS (81);
3. missing CloudFormation resource types (69) and unsupported features (53);
4. RDS engine versions (42);
5. Docker asset pushes (40);
6. CloudFormation attribute support (31).

On the checks:
- ECS `DescribeServices` timestamps break the JavaScript SDK.
- S3 bucket policies are reported created but don't exist.
- `AWS::URLSuffix` is `127.0.0.1`.

The emulator also accepts templates AWS rejects (25 tests), so it can't replace a real deploy. Each item above is a
concrete report for the emulator team (CTI AWS / AWSInABox / Feedback). Per-test emulator results are in
`~/emulator-run-full`, and the comparison is in `~/baseline-tools/compare-emulator.js`.

## Cleanup and accounts

- **test-0:** no stacks or Inferentia instances left from the run. The Inferentia quota is now 32 vCPUs in us-east-1
  and us-west-2.
- **test-2:** only its 4 `CDKToolkit` stacks, kept as asked. Its support case `179492781600617` is still open.
- **test-3:** cleared of its app stacks; only `CDKToolkit` is left.
- **Emulator:** running in tmux session `emulator` (state is in memory). The full-run results are in `~/emulator-run-full`.
