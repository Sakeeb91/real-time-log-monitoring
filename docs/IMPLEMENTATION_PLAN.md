# Implementation Plan: Real-Time Log Monitoring

## Expert Role: Cloud/DevOps Engineer (AWS Serverless Specialist)

**Why this role is optimal:**
This project is fundamentally about building serverless infrastructure on AWS. It requires deep understanding of:
- AWS service integrations and IAM permissions
- Event-driven architecture patterns
- Infrastructure-as-Code best practices
- Cost optimization within Free Tier limits
- Observability and monitoring principles

A Cloud/DevOps Engineer with serverless expertise will navigate AWS service configurations, handle the complex permission chains between services, and ensure the system is both functional and cost-effective.

---

## System Architecture

### Component Diagram

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        REAL-TIME LOG MONITORING                         │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  ┌──────────────┐                                                       │
│  │ Application  │                                                       │
│  │   (Source)   │                                                       │
│  └──────┬───────┘                                                       │
│         │ Logs                                                          │
│         ▼                                                               │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │                    CLOUDWATCH LOGS                                │  │
│  │  ┌─────────────┐                                                  │  │
│  │  │  Log Group  │ /aws/lambda/test-log-emitter                    │  │
│  │  │             │                                                  │  │
│  │  │ Log Streams │ 2024/01/01/[$LATEST]xxxxx                       │  │
│  │  └──────┬──────┘                                                  │  │
│  └─────────┼────────────────────────────────────────────────────────┘  │
│            │                                                            │
│     ┌──────┴──────┐                                                     │
│     │             │                                                     │
│     ▼             ▼                                                     │
│  ┌─────────────────────┐    ┌────────────────────────────────────────┐ │
│  │   ALERTING PATH     │    │         PROCESSING PATH                │ │
│  │                     │    │                                        │ │
│  │  ┌──────────────┐   │    │  ┌──────────────────┐                  │ │
│  │  │Metric Filter │   │    │  │Subscription Filter│                 │ │
│  │  │ (ERROR count)│   │    │  │ (Pattern: ERROR)  │                 │ │
│  │  └──────┬───────┘   │    │  └────────┬─────────┘                  │ │
│  │         │           │    │           │                            │ │
│  │         ▼           │    │           ▼                            │ │
│  │  ┌──────────────┐   │    │  ┌──────────────────┐                  │ │
│  │  │CloudWatch    │   │    │  │    Lambda        │                  │ │
│  │  │Alarm         │   │    │  │ LogAlertProcessor│                  │ │
│  │  │(>=1 in 1min) │   │    │  └────────┬─────────┘                  │ │
│  │  └──────┬───────┘   │    │           │                            │ │
│  │         │           │    │     ┌─────┴─────┐                      │ │
│  │         ▼           │    │     ▼           ▼                      │ │
│  │  ┌──────────────┐   │    │  ┌──────┐  ┌──────────┐                │ │
│  │  │     SNS      │   │    │  │ DDB  │  │ SNS      │                │ │
│  │  │   (Topic)    │   │    │  │      │  │(Optional)│                │ │
│  │  └──────┬───────┘   │    │  └──────┘  └──────────┘                │ │
│  │         │           │    │                                        │ │
│  │         ▼           │    └────────────────────────────────────────┘ │
│  │  ┌──────────────┐   │                                               │
│  │  │    Email     │   │                                               │
│  │  │ Notification │   │                                               │
│  │  └──────────────┘   │                                               │
│  └─────────────────────┘                                               │
│                                                                         │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │                    VISUALIZATION                                  │  │
│  │                                                                   │  │
│  │  ┌──────────────────────────────────────────────────────────┐    │  │
│  │  │              CloudWatch Dashboard                         │    │  │
│  │  │  ┌────────────────┐  ┌────────────────┐                  │    │  │
│  │  │  │ Error Count    │  │ Logs Insights  │                  │    │  │
│  │  │  │ Metric Widget  │  │ Query Widget   │                  │    │  │
│  │  │  └────────────────┘  └────────────────┘                  │    │  │
│  │  └──────────────────────────────────────────────────────────┘    │  │
│  └──────────────────────────────────────────────────────────────────┘  │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### Data Flow

1. **Log Ingestion**: Application writes logs to CloudWatch Logs
2. **Pattern Detection**: Metric Filter counts ERROR occurrences
3. **Alarm Trigger**: CloudWatch Alarm fires when threshold exceeded
4. **Notification**: SNS sends email to subscribed addresses
5. **Processing**: Lambda processes log events, enriches data
6. **Storage**: DynamoDB stores incident records
7. **Visualization**: Dashboard displays real-time metrics

---

## Technology Choices

| Component | Choice | Rationale | Tradeoffs | Fallback |
|-----------|--------|-----------|-----------|----------|
| Infrastructure | CloudFormation | Native AWS, no extra tools, YAML-based | Verbose syntax, AWS-only | AWS SAM or Terraform |
| Runtime | Python 3.11 | Junior-friendly, excellent AWS SDK | Slower than compiled | Node.js |
| Log Storage | CloudWatch Logs | Native integration, Free Tier | Limited query capability | OpenSearch |
| Database | DynamoDB | Serverless, Free Tier, simple schema | NoSQL learning curve | Aurora Serverless |
| Alerting | SNS | Native integration, email support | Limited formatting | SES |
| Functions | Lambda | Pay-per-use, Free Tier | Cold starts | Fargate |

### Patterns Applied

| Pattern | Where Used | Benefit |
|---------|------------|---------|
| Fan-out | Log Group to multiple subscribers | Parallel processing paths |
| Event Sourcing | DynamoDB incidents | Full audit trail |
| Dead Letter Queue | Lambda DLQ | Error recovery |
| Idempotency | Lambda handler | Safe retries |

---

## Phased Implementation Plan

### Phase 1: Foundation - Log Ingestion and Basic Alerting

**Objective**: Establish working log pipeline with email alerts

**Scope**:
- Create CloudWatch Log Group
- Create Metric Filter for ERROR pattern
- Create CloudWatch Alarm
- Create SNS Topic with email subscription
- Create test Lambda function to emit logs

**Files to Create**:
```
infrastructure/
├── cloudformation/
│   └── phase1-alerting.yaml
└── scripts/
    └── deploy-phase1.sh
lambda/
└── test_emitter/
    └── handler.py
```

**Deliverables**:
- [ ] CloudFormation stack deploys successfully
- [ ] Test Lambda emits ERROR logs
- [ ] Alarm triggers on ERROR threshold
- [ ] Email notification received

**Verification Method**:
```bash
# Deploy stack
aws cloudformation deploy --template-file phase1-alerting.yaml --stack-name log-monitor-phase1 --capabilities CAPABILITY_IAM

# Invoke test emitter
aws lambda invoke --function-name TestLogEmitter response.json

# Check alarm state
aws cloudwatch describe-alarms --alarm-names ErrorAlarm
```

**Technical Challenges**:
- IAM permissions for CloudWatch Logs to invoke Lambda
- SNS subscription confirmation timeout
- Metric Filter pattern syntax

**Definition of Done**:
- Stack creates without errors
- Alarm reaches ALARM state within 2 minutes of ERROR log
- Email arrives at configured address

**Time Estimate**: 45 minutes
**Contingency Cut**: Skip SNS formatting customization

---

### Phase 2: Processing - Lambda Log Processor and DynamoDB

**Objective**: Process log events and persist incidents

**Scope**:
- Create DynamoDB table for incidents
- Create Lambda processor function
- Create CloudWatch Logs subscription filter
- Configure IAM roles and policies

**Files to Create**:
```
infrastructure/
└── cloudformation/
    └── phase2-processing.yaml
lambda/
└── log_processor/
    ├── handler.py
    └── requirements.txt
```

**Deliverables**:
- [ ] DynamoDB table created with proper schema
- [ ] Lambda processes log events
- [ ] Incidents written to DynamoDB
- [ ] Subscription filter connects logs to Lambda

**Code Skeleton - handler.py**:
```python
import json
import base64
import gzip
import boto3
import os
from datetime import datetime

dynamodb = boto3.resource('dynamodb')
table = dynamodb.Table(os.environ['TABLE_NAME'])

def lambda_handler(event, context):
    """
    Process CloudWatch Logs events and store ERROR incidents in DynamoDB.

    Event structure:
    {
        "awslogs": {
            "data": "<base64-encoded-gzipped-json>"
        }
    }
    """
    # Decode and decompress log data
    compressed_payload = base64.b64decode(event['awslogs']['data'])
    uncompressed_payload = gzip.decompress(compressed_payload)
    log_data = json.loads(uncompressed_payload)

    log_group = log_data['logGroup']
    log_stream = log_data['logStream']

    incidents = []
    for log_event in log_data['logEvents']:
        message = log_event['message']
        timestamp = log_event['timestamp']

        # Only process ERROR logs
        if 'ERROR' in message:
            incident = {
                'incident_id': f"{log_stream}-{log_event['id']}",
                'timestamp': datetime.fromtimestamp(timestamp/1000).isoformat(),
                'log_group': log_group,
                'log_stream': log_stream,
                'message': message,
                'severity': 'ERROR',
                'created_at': datetime.utcnow().isoformat()
            }
            incidents.append(incident)
            table.put_item(Item=incident)

    return {
        'statusCode': 200,
        'body': json.dumps({
            'processed': len(log_data['logEvents']),
            'incidents_created': len(incidents)
        })
    }
```

**Verification Method**:
```bash
# Deploy stack
aws cloudformation deploy --template-file phase2-processing.yaml --stack-name log-monitor-phase2 --capabilities CAPABILITY_IAM

# Emit test errors
aws lambda invoke --function-name TestLogEmitter response.json

# Check DynamoDB
aws dynamodb scan --table-name LogIncidents
```

**Technical Challenges**:
- Base64 + gzip decoding of CloudWatch Logs events
- DynamoDB key design for efficient queries
- Lambda permission to write to DynamoDB
- Subscription filter permission chain

**Definition of Done**:
- Lambda invoked automatically on ERROR logs
- DynamoDB contains incident records with correct schema
- No errors in Lambda logs

**Time Estimate**: 60 minutes
**Contingency Cut**: Skip enrichment fields, use minimal schema

---

### Phase 3: Visualization - CloudWatch Dashboard

**Objective**: Create monitoring dashboard with metrics and logs

**Scope**:
- Create CloudWatch Dashboard
- Add Error Count metric widget
- Add Logs Insights query widget
- Add alarm status widget

**Files to Create**:
```
infrastructure/
└── cloudformation/
    └── phase3-dashboard.yaml
```

**Deliverables**:
- [ ] Dashboard created with all widgets
- [ ] Error metrics display correctly
- [ ] Logs Insights queries work
- [ ] Alarm status visible

**Dashboard JSON Structure**:
```json
{
    "widgets": [
        {
            "type": "metric",
            "properties": {
                "title": "Error Count (1 min)",
                "metrics": [
                    ["LogMonitoring", "ErrorCount", {"stat": "Sum", "period": 60}]
                ]
            }
        },
        {
            "type": "log",
            "properties": {
                "title": "Recent Errors",
                "query": "fields @timestamp, @message | filter @message like /ERROR/ | sort @timestamp desc | limit 20"
            }
        },
        {
            "type": "alarm",
            "properties": {
                "title": "Alarm Status",
                "alarms": ["arn:aws:cloudwatch:REGION:ACCOUNT:alarm:ErrorAlarm"]
            }
        }
    ]
}
```

**Verification Method**:
```bash
# Deploy stack
aws cloudformation deploy --template-file phase3-dashboard.yaml --stack-name log-monitor-phase3

# Open dashboard
aws cloudwatch get-dashboard --dashboard-name LogMonitoringDashboard
```

**Technical Challenges**:
- Dashboard JSON escaping in CloudFormation
- Logs Insights query syntax
- Cross-referencing alarm ARNs

**Definition of Done**:
- Dashboard loads without errors
- Metrics show real data after test invocations
- Logs Insights returns recent errors

**Time Estimate**: 30 minutes
**Contingency Cut**: Use AWS Console to create dashboard manually

---

### Phase 4: Integration Testing and Refinement

**Objective**: Validate end-to-end flow and add robustness

**Scope**:
- Create integration tests
- Add Dead Letter Queue for Lambda
- Add error handling and retries
- Create cleanup script

**Files to Create**:
```
tests/
├── test_log_processor.py
└── test_integration.py
infrastructure/
└── scripts/
    └── cleanup.sh
```

**Deliverables**:
- [ ] All tests pass
- [ ] DLQ configured for failed Lambda invocations
- [ ] Cleanup script removes all resources

**Test Code Skeleton**:
```python
# tests/test_log_processor.py
import pytest
import json
import base64
import gzip
from lambda.log_processor.handler import lambda_handler

def create_test_event(messages):
    """Create a mock CloudWatch Logs event."""
    log_data = {
        'logGroup': '/aws/lambda/test',
        'logStream': 'test-stream',
        'logEvents': [
            {
                'id': f'event-{i}',
                'timestamp': 1704067200000 + i,
                'message': msg
            }
            for i, msg in enumerate(messages)
        ]
    }
    compressed = gzip.compress(json.dumps(log_data).encode())
    encoded = base64.b64encode(compressed).decode()
    return {'awslogs': {'data': encoded}}

class TestLogProcessor:
    def test_processes_error_logs(self, dynamodb_table):
        """Verify ERROR logs create incidents."""
        event = create_test_event(['ERROR: Connection failed'])
        result = lambda_handler(event, None)
        assert result['statusCode'] == 200
        body = json.loads(result['body'])
        assert body['incidents_created'] == 1

    def test_ignores_info_logs(self, dynamodb_table):
        """Verify INFO logs do not create incidents."""
        event = create_test_event(['INFO: Health check OK'])
        result = lambda_handler(event, None)
        body = json.loads(result['body'])
        assert body['incidents_created'] == 0

    def test_handles_burst_of_errors(self, dynamodb_table):
        """Verify multiple errors in single event are processed."""
        event = create_test_event([f'ERROR: Failure {i}' for i in range(10)])
        result = lambda_handler(event, None)
        body = json.loads(result['body'])
        assert body['incidents_created'] == 10
```

**Verification Method**:
```bash
# Run tests
pytest tests/ -v

# Test cleanup
./infrastructure/scripts/cleanup.sh --dry-run
```

**Technical Challenges**:
- Mocking DynamoDB in tests
- DLQ configuration in CloudFormation
- Idempotent cleanup script

**Definition of Done**:
- 100% test pass rate
- DLQ receives failed invocations
- Cleanup removes all resources

**Time Estimate**: 45 minutes
**Contingency Cut**: Skip DLQ, manual cleanup only

---

## Risk Assessment

| Risk | Likelihood | Impact | Early Warning | Mitigation |
|------|------------|--------|---------------|------------|
| IAM permission errors | High | Medium | Stack creation fails | Use managed policies, test incrementally |
| Free Tier exceeded | Medium | Low | Billing alert email | Set up budget alarm, monitor usage |
| Cold start delays | Medium | Low | Slow alarm response | Use provisioned concurrency (costs money) |
| Metric Filter pattern miss | High | High | No alarms triggered | Test with exact log format first |
| DynamoDB throttling | Low | Medium | ProvisionedThroughputExceeded | Use on-demand capacity |
| Lambda timeout | Low | Medium | Duration approaching limit | Increase timeout, optimize code |
| SNS delivery failure | Low | High | No emails received | Verify email, check spam folder |

### Contingency Actions

1. **If CloudFormation fails**: Fall back to AWS Console for manual resource creation
2. **If tests fail**: Skip test phase, validate manually
3. **If over time budget**: Skip Phase 4, deliver core functionality only

---

## Testing Strategy

### Testing Pyramid

```
        ┌─────────────────┐
        │   E2E Tests     │  ← Manual verification
        │   (1-2 tests)   │
        ├─────────────────┤
        │ Integration     │  ← AWS LocalStack or moto
        │   (3-5 tests)   │
        ├─────────────────┤
        │  Unit Tests     │  ← pytest with mocks
        │  (10+ tests)    │
        └─────────────────┘
```

### First Three Tests to Write

1. **`test_error_log_creates_incident`**
   - Given: CloudWatch Logs event with ERROR message
   - When: Lambda handler processes event
   - Then: DynamoDB contains incident record

2. **`test_info_log_ignored`**
   - Given: CloudWatch Logs event with INFO message
   - When: Lambda handler processes event
   - Then: No incident created

3. **`test_alarm_triggers_on_threshold`**
   - Given: Metric Filter counting errors
   - When: Error count >= 1 in 1 minute
   - Then: Alarm state transitions to ALARM

### Testing Framework

```bash
# Install test dependencies
pip install pytest pytest-mock moto boto3

# Run tests
pytest tests/ -v --cov=lambda
```

---

## First Concrete Task

### File to Create First

`lambda/test_emitter/handler.py`

### Function Signature

```python
def lambda_handler(event: dict, context) -> dict:
    """
    Emit test log messages to CloudWatch Logs.

    Args:
        event: Lambda event (can contain 'error_count' parameter)
        context: Lambda context

    Returns:
        dict with statusCode and emitted log summary
    """
```

### Starter Code (Copy-Paste Ready)

```python
import logging
import random

# Configure logging to output to CloudWatch Logs
logger = logging.getLogger()
logger.setLevel(logging.INFO)

def lambda_handler(event, context):
    """
    Emit test log messages to CloudWatch Logs.

    Usage:
        Invoke with {"error_count": 5} to emit 5 ERROR logs.
        Default behavior emits random mix of INFO and ERROR.
    """
    error_count = event.get('error_count', 0)

    # Always emit a health check
    logger.info("Health check OK - TestLogEmitter running")

    if error_count > 0:
        # Emit specified number of errors
        for i in range(error_count):
            logger.error(f"Simulated failure {i+1}: TimeoutError connecting to database")
    else:
        # Random behavior - 40% chance of error
        if random.random() < 0.4:
            logger.error("Simulated failure: ConnectionRefusedError")

    return {
        'statusCode': 200,
        'body': {
            'message': 'Log emission complete',
            'errors_emitted': error_count if error_count > 0 else 'random'
        }
    }
```

### Verification Method

```bash
# After deploying Lambda via CloudFormation:
aws lambda invoke --function-name TestLogEmitter --payload '{"error_count": 3}' response.json
cat response.json

# Check CloudWatch Logs:
aws logs filter-log-events --log-group-name /aws/lambda/TestLogEmitter --filter-pattern "ERROR"
```

### First Commit Message

```
feat(lambda): Add TestLogEmitter function for log generation

- Implement handler.py with configurable error emission
- Support both deterministic (error_count) and random modes
- Output logs compatible with ERROR metric filter pattern

Part of Phase 1: Foundation - Log Ingestion and Basic Alerting
```

---

## Learning Notes

### Concepts to Understand Before Coding

1. **CloudWatch Logs Structure**: Log Groups contain Log Streams contain Log Events
2. **IAM Trust Policies**: Lambda needs `logs.amazonaws.com` as trusted principal
3. **Metric Filter Patterns**: `[..., level="ERROR", ...]` or simple string matching
4. **CloudFormation Intrinsic Functions**: `!Ref`, `!GetAtt`, `!Sub`
5. **Base64 + GZIP**: CloudWatch Logs subscription events are compressed

### Helpful AWS Documentation

- [CloudWatch Logs Subscription Filters](https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/SubscriptionFilters.html)
- [Lambda Event Sources](https://docs.aws.amazon.com/lambda/latest/dg/invocation-eventsourcemapping.html)
- [CloudFormation Resource Types](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-template-resource-type-ref.html)

---

## Summary

| Phase | Deliverable | Verification |
|-------|-------------|--------------|
| 1 | Basic alerting pipeline | Email notification received |
| 2 | Log processing and storage | DynamoDB incident records |
| 3 | Monitoring dashboard | Dashboard displays metrics |
| 4 | Tests and robustness | All tests pass |

**Total Estimated Time**: 3 hours
**Minimum Viable Product**: Phase 1 + Phase 2 (1.75 hours)
