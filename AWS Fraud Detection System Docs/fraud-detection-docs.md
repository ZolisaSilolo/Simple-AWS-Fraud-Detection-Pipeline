# AWS Cloud-Native Fraud Detection Pipeline

## Overview
This project implements a serverless, real-time fraud detection system using AWS services. The solution provides scalable transaction processing, machine learning-based fraud detection, and automated alert mechanisms while maintaining cost-effectiveness.


## Architecture
The system consists of four main components:

### 1. Data Ingestion Layer
- **API Gateway**: Provides a RESTful API endpoint for transaction ingestion
- **Lambda**: Processes incoming transactions and streams them to Kinesis
- **Kinesis Data Streams**: Handles real-time data streaming with on-demand scaling

Key optimizations:
- ON_DEMAND mode for Kinesis to optimize costs
- Optimized Lambda configuration (128MB memory, Python 3.11)
- Compressed API responses for reduced data transfer costs

### 2. Feature Storage Layer
- **S3 Feature Store**: Stores historical transaction data and features
- Implements intelligent data lifecycle management:
  - 0-30 days: STANDARD storage
  - 30-90 days: STANDARD_IA storage
  - 90+ days: INTELLIGENT_TIERING
  - Automatic deletion after 365 days

### 3. Model Infrastructure
- **SageMaker Endpoint**: Hosts the fraud detection model
- **S3 Model Storage**: Stores model artifacts and versions
- Cost-effective configuration:
  - Uses `ml.t2.medium` instances
  - Serverless configuration with auto-scaling
  - Maximum concurrency of 50 requests
  - 1024MB memory allocation

### 4. Fraud Response System
- **DynamoDB**: Stores flagged transactions with automatic TTL
- **SNS**: Handles real-time fraud alerts

## Security Features
1. **IAM Role Configuration**
   - Principle of least privilege
   - Managed policies for reduced maintenance
   - Service-specific role permissions

2. **Data Protection**
   - S3 bucket security:
     ```yaml
     PublicAccessBlockConfiguration:
       BlockPublicAcls: true
       BlockPublicPolicy: true
       IgnorePublicAcls: true
       RestrictPublicBuckets: true
     ```
   - Encryption at rest
   - Secure API endpoints

## Cost Optimization Strategies

### 1. Compute Optimization
```yaml
IngestionStream:
  StreamModeDetails:
    StreamMode: ON_DEMAND  # Pay per use

IngestionLambda:
  MemorySize: 128  # Optimized for cost
  Timeout: 10
```

### 2. Storage Optimization
```yaml
FeatureStoreBucket:
  LifecycleConfiguration:
    Rules:
      - Transitions:
          - TransitionInDays: 30
            StorageClass: STANDARD_IA
          - TransitionInDays: 90
            StorageClass: INTELLIGENT_TIERING
```

### 3. ML Infrastructure
```yaml
FraudDetectionEndpointConfig:
  ProductionVariants:
    ServerlessConfig:
      MaxConcurrency: 50
      MemorySizeInMB: 1024
```

## Deployment Instructions

### Prerequisites
- AWS CLI installed and configured
- Appropriate AWS permissions
- Python 3.9 or higher

### Deployment Steps
1. Clone the repository:
   ```bash
   git clone [repository-url]
   cd aws-fraud-detection
   ```

2. Deploy the CloudFormation stack:
   ```bash
   aws cloudformation create-stack \
     --stack-name fraud-detection \
     --template-body file://template.yaml \
     --capabilities CAPABILITY_IAM
   ```

3. Monitor the deployment:
   ```bash
   aws cloudformation describe-stacks \
     --stack-name fraud-detection
   ```

### Testing the Deployment

1. Get the API endpoint:
   ```bash
   aws cloudformation describe-stacks \
     --stack-name fraud-detection \
     --query 'Stacks[0].Outputs[?OutputKey==`IngestionApiUrl`].OutputValue' \
     --output text
   ```

2. Send a test transaction:
   ```bash
   curl -X POST \
     -H "Content-Type: application/json" \
     -d '{"amount": 1000, "merchant": "Test Store", "card_present": true}' \
     [API-ENDPOINT]
   ```

## Monitoring and Maintenance

### Key Metrics to Monitor
1. API Gateway
   - Latency
   - Error rates
   - Request count

2. Lambda
   - Duration
   - Error rate
   - Throttles

3. SageMaker
   - Invocation metrics
   - Model latency
   - Instance utilization

### Cost Monitoring
- Enable AWS Cost Explorer
- Set up AWS Budgets
- Monitor usage patterns

### Recommended Alarms
```yaml
# Example CloudWatch Alarms
- API Gateway 5XX errors > 1%
- Lambda duration > 5 seconds
- SageMaker endpoint latency > 100ms
```

## Future Improvements
1. Add API authentication
2. Implement request validation
3. Add model A/B testing capability
4. Implement cross-region disaster recovery
5. Add real-time monitoring dashboard

## Contributing
Please read [CONTRIBUTING.md] for details on our code of conduct and the process for submitting pull requests.

## License
This project is licensed under the MIT License - see the [LICENSE.md] file for details.

