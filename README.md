# Real-Time Log Monitoring

A lightweight, serverless log monitoring and alerting system built on AWS. This project implements a "Datadog-style" pipeline for detecting errors in application logs in near-real time, triggering alerts, and visualizing trends through dashboards.

## Overview

This system provides:
- **Real-time error detection** via CloudWatch Metric Filters
- **Automated alerting** through SNS notifications
- **Incident persistence** in DynamoDB for tracking and analysis
- **Live dashboards** for monitoring trends and patterns
- **Optional analytics pipeline** using S3, Athena, and QuickSight

## Architecture

```
                                    ┌─────────────┐
                                    │    SNS      │
                                    │  (Alerts)   │
                                    └──────▲──────┘
                                           │
┌──────────────┐    ┌───────────────┐    ┌─┴───────────┐
│  CloudWatch  │───▶│ Metric Filter │───▶│   Alarm     │
│    Logs      │    │ (ERROR count) │    │ (Threshold) │
└──────┬───────┘    └───────────────┘    └─────────────┘
       │
       │ Subscription Filter
       ▼
┌──────────────┐    ┌───────────────┐
│   Lambda     │───▶│  DynamoDB     │
│  (Processor) │    │ (LogIncidents)│
└──────────────┘    └───────────────┘

Optional Analytics Path:
┌──────────────┐    ┌───────┐    ┌─────────┐    ┌────────────┐
│   Firehose   │───▶│  S3   │───▶│ Athena  │───▶│ QuickSight │
└──────────────┘    └───────┘    └─────────┘    └────────────┘
```

## AWS Services Used

### Core (Free Tier Friendly)
| Service | Purpose |
|---------|---------|
| CloudWatch Logs | Centralized log ingestion |
| Metric Filters | Pattern matching for ERROR logs |
| CloudWatch Alarms | Threshold-based alerting |
| SNS | Email/SMS notifications |
| Lambda | Log processing and enrichment |
| DynamoDB | Incident storage |
| CloudWatch Dashboards | Visualization |

### Optional (Analytics Extension)
| Service | Purpose |
|---------|---------|
| Kinesis Firehose | Log streaming to S3 |
| S3 | Long-term log storage |
| Glue | Data catalog |
| Athena | SQL queries on logs |
| QuickSight | Business intelligence dashboards |

## Project Structure

```
real-time-log-monitoring/
├── README.md
├── docs/
│   └── IMPLEMENTATION_PLAN.md
├── infrastructure/
│   ├── cloudformation/
│   │   ├── core-stack.yaml
│   │   └── analytics-stack.yaml (optional)
│   └── scripts/
│       ├── deploy.sh
│       └── cleanup.sh
├── lambda/
│   ├── log_processor/
│   │   ├── handler.py
│   │   └── requirements.txt
│   └── test_emitter/
│       └── handler.py
├── tests/
│   ├── test_log_processor.py
│   └── test_integration.py
└── .github/
    └── workflows/
        └── deploy.yaml
```

## Quick Start

### Prerequisites
- AWS Account (Free Tier eligible)
- AWS CLI configured with appropriate credentials
- Python 3.9+

### Deployment

1. Clone the repository:
```bash
git clone https://github.com/Sakeeb91/real-time-log-monitoring.git
cd real-time-log-monitoring
```

2. Deploy the core infrastructure:
```bash
cd infrastructure/scripts
chmod +x deploy.sh
./deploy.sh
```

3. Test the pipeline:
```bash
# Emit test error logs
aws lambda invoke --function-name TestLogEmitter output.json
```

4. Verify:
- Check your email for SNS alarm notification
- View CloudWatch Dashboard for metrics
- Query DynamoDB for incident records

## Testing Matrix

| Test Case | Action | Expected Result |
|-----------|--------|-----------------|
| Single error | Emit one ERROR log | Alarm triggers, email arrives, 1 DDB row |
| Burst of errors | Emit 10 errors quickly | One alarm state + multiple DDB rows |
| No matches | Emit only INFO logs | No alarm; DDB unchanged |
| Subscription broken | Remove subscription | Alarm path still works; processor stops |
| Permission issue | Remove sns:Publish | DDB rows written; SNS error in Lambda logs |
| Retention check | Set retention=1 day | Old logs auto-expire |

## Cleanup

To avoid ongoing costs, run:

```bash
cd infrastructure/scripts
chmod +x cleanup.sh
./cleanup.sh
```

Or manually delete:
1. Subscription filter, alarm, metric filter
2. LogIncidents DynamoDB table
3. LogAlertProcessor Lambda function and IAM role
4. SNS subscription/topic
5. (Optional) Log group or reduce retention
6. (Optional) Firehose, S3, Glue, Athena objects, QuickSight resources

## Cost Estimate

This project is designed to stay within AWS Free Tier limits:
- Lambda: 1M free requests/month
- DynamoDB: 25 GB storage, 25 WCU/RCU
- CloudWatch: 10 custom metrics, 10 alarms
- SNS: 1M publishes free

## License

MIT License - See LICENSE file for details.

## Contributing

Contributions are welcome. Please open an issue first to discuss proposed changes.
