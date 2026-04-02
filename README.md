# IntiBank — Automated Incident Investigation with AWS DevOps Agent

CloudFormation templates that deploy IntiBank's transaction processing API and an automated bridge that triggers [AWS DevOps Agent](https://aws.amazon.com/devops-agent/) investigations from Amazon CloudWatch alarms.

## What's in each template

### `template.yaml` — Core Banking API

Deploys the transaction processing backend:

| Resource | Type | Purpose |
|---|---|---|
| `intibank-txn-processor` | AWS::Lambda::Function | Python 3.12 function that processes transactions. Simulates a core ledger outage (~60% failure rate) |
| `intibank-api` | AWS::ApiGatewayV2::Api | HTTP API Gateway that exposes the Lambda via `ANY /{proxy+}` |
| `intibank-txn-errors` | AWS::CloudWatch::Alarm | Fires when Lambda errors ≥ 3 in a 60-second period |
| `intibank-txn-high-latency` | AWS::CloudWatch::Alarm | Fires when average Lambda duration exceeds 5 seconds |
| `intibank-transactions-ops` | AWS::CloudWatch::Dashboard | Operational dashboard with error count, latency (avg + p99), invocations vs failures, and alarm status |
| `intibank-txn-processor-role` | AWS::IAM::Role | Lambda execution role with `AWSLambdaBasicExecutionRole` |

**Outputs:** API URL, function name, error alarm name.

### `bridge-template.yaml` — Incident Response Bridge

Connects CloudWatch alarms to AWS DevOps Agent:

| Resource | Type | Purpose |
|---|---|---|
| `intibank-incident-bridge` (topic) | AWS::SNS::Topic | Receives alarm notifications via CloudWatch alarm actions |
| `intibank-incident-bridge` (function) | AWS::Lambda::Function | Parses the SNS alarm message and calls the DevOps Agent API to create an `INVESTIGATION` backlog task |
| `intibank-incident-bridge-role` | AWS::IAM::Role | Lambda execution role with `aidevops:CreateBacklogTask` permission |

The bridge Lambda uses `botocore.auth.SigV4Auth` to sign requests against the `aidevops` service endpoint (`dp.aidevops.<region>.api.aws`), since the DevOps Agent SDK is not yet included in the Lambda runtime's bundled boto3.

**Parameters:** `AgentSpaceId` — the ID of your DevOps Agent Agent Space.

**Outputs:** SNS topic ARN (to use as alarm action).

## Flow

```
API Gateway → Lambda (transaction errors)
                   ↓
             CloudWatch Metrics
                   ↓
             CloudWatch Alarm ──alarm action──→ SNS Topic
                                                    ↓
                                              Bridge Lambda
                                                    ↓
                                          DevOps Agent Investigation
                                                    ↓
                                          Root Cause + Mitigation
```

## Prerequisites

- AWS CLI v2 with configured credentials
- AWS DevOps Agent enabled in your account
- An Agent Space with an associated AWS account (monitor type)

## Deploy

```bash
# 1. Core banking API
aws cloudformation deploy \
  --template-file template.yaml \
  --stack-name intibank-core-banking \
  --region us-east-1 \
  --capabilities CAPABILITY_NAMED_IAM

# 2. Incident bridge
aws cloudformation deploy \
  --template-file bridge-template.yaml \
  --stack-name intibank-incident-bridge \
  --region us-east-1 \
  --capabilities CAPABILITY_NAMED_IAM \
  --parameter-overrides AgentSpaceId="<YOUR_AGENT_SPACE_ID>"

# 3. Wire alarm to bridge
TOPIC_ARN=$(aws cloudformation describe-stacks \
  --stack-name intibank-incident-bridge \
  --query "Stacks[0].Outputs[?OutputKey=='AlarmTopicArn'].OutputValue" \
  --output text)

aws cloudwatch put-metric-alarm \
  --alarm-name intibank-txn-errors \
  --alarm-actions "$TOPIC_ARN" \
  --namespace AWS/Lambda --metric-name Errors \
  --dimensions Name=FunctionName,Value=intibank-txn-processor \
  --statistic Sum --period 60 --evaluation-periods 1 --threshold 3 \
  --comparison-operator GreaterThanOrEqualToThreshold \
  --treat-missing-data notBreaching \
  --alarm-description "Transaction processing failures exceeded threshold"
```

## Cleanup

```bash
aws cloudformation delete-stack --stack-name intibank-incident-bridge --region us-east-1
aws cloudformation delete-stack --stack-name intibank-core-banking --region us-east-1
```
