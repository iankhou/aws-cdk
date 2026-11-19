# aws-cdk integ snapshots: real-AWS baseline and AWS Emulator comparison

2026-11-18. Real AWS is the source of truth. Results are on
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

## AWS Emulator: results on the same 49 tests

On real AWS, all 49 deploy and are checked, and 48 have no fails. On the emulator:

| | Real AWS | Emulator |
|---|---|---|
| Deployed and checked | 49 | 26 |
| Deploy failed | 0 | 23 |
| Checks on all 49: pass / fail / missing / error | 2,964 / 1 / 0 / 0 | n/a |
| Checks on the 26 the emulator deployed: pass / fail / missing / error | 1,522 / 0 / 0 / 0 | 1,456 / 4 / 5 / 24 |
| Unresolved checks on those 26 | 39 | 61 |

Of the 26 tests the emulator deployed, 15 match real AWS exactly. The other 11 differ as follows.

**Why 23 tests fail to deploy on the emulator** (all deploy on real AWS):

| Tests | Emulator behavior |
|---|---|
| 7 | CloudFront `CreateDistributionWithTags` rejects origins: `Value null at ...s3OriginConfig.originAccessIdentity` / `customOriginConfig.hTTPPort`, where AWS applies defaults |
| 6 | Custom resources fail: CDK assertion stacks (`DeployAssert` SDK calls to Lambda, Step Functions and HTTP), `Custom::CloudwatchLogResourcePolicy` and `Custom::VpcRestrictDefaultSG`. Lambda function runs aren't ready in the emulator |
| 4 | Docker image assets can't be pushed: `docker push` to the ECR registry host (`<account>.dkr.ecr.<region>.amazonaws.com`) fails with `no basic auth credentials`, since that host isn't the emulator's endpoint |
| 2 | CloudFormation doesn't know the resource types `AWS::MediaPackageV2::ChannelGroup` and `AWS::S3Files::*` |
| 1 | API Gateway rejects the path part `$appliances:all`, which AWS accepts |
| 1 | API Gateway doesn't read a `RestApi` definition from `BodyS3Location` |
| 1 | `AWS::Lambda::Permission` `InvokedViaFunctionUrl` isn't applied |
| 1 | Application Auto Scaling scheduled action: `invalid timestamp: failed to parse int` |

**Where the emulator differs on the 26 tests it deployed:**
- **CloudFormation reports success for resources it didn't create.**
  - 4 `AWS::S3::BucketPolicy` are `missing` (`NoSuchBucket`).
  - 1 `ClusterCapacityProviderAssociations` is `missing` (`ClusterNotFoundException`).
- **The ECS `DescribeServices` response can't be read by the JavaScript SDK.** Timestamps come back as integer strings,
  so the SDK throws `Epoch timestamps must be expressed as floating point numbers`. That's 24 `error` checks across 4
  tests.
- **S3 `GetBucketLifecycleConfiguration` returns `Filter: {}`** where AWS returns `Filter: { Prefix: "" }` (3 fails).
- **KMS keeps a bare account-id principal as given.** AWS stores it as the account's root ARN (1 fail, `aws-kms/integ.key`).
- **More `unresolved` checks:** 61 against 39 on AWS. The `evaluate` action couldn't resolve some `Fn::GetAtt`
  values from the emulator's stacks.
- **Same as AWS:** for the 15 tests that matched exactly, every check matches real AWS. That covers CloudWatch
  alarms, Kafka event sources, DynamoDB autoscaling, Step Functions logging, SNS URL subscriptions and several
  CloudFront distributions.

**Takeaway.** For these 49 tests, the emulator deployed about half. Where it deployed, most checks match AWS. The
biggest gaps are:
1. CloudFront distribution validation (7 tests);
2. custom resources and Lambda (6);
3. Docker image assets (4);
4. resources reported as created but never created (5 checks);
5. ECS `DescribeServices` timestamps.

These are concrete, reproducible bug reports for the emulator team: CTI AWS / AWSInABox / Feedback.

## Cleanup and accounts

- **test-0:** no stacks or Inferentia instances left from the run. The Inferentia quota is now 32 vCPUs in us-east-1
  and us-west-2.
- **test-2:** only its 4 `CDKToolkit` stacks, kept as asked. Its support case `179492781600617` is still open.
- **test-3:** cleared of its app stacks; only `CDKToolkit` is left.
- **Emulator:** running in tmux session `emulator` (state is in memory; stopping it deletes everything). The results
  are in `~/emulator-run` and `~/emulator-snapshots`.
