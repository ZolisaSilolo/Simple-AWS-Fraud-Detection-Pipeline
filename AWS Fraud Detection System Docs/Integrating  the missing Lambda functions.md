Let's integrate the missing Lambda functions and dynamic partition keys into the CloudFormation template. We'll add a PredictionLambda to invoke the SageMaker endpoint and a ResponseLambda to handle the results.
below is the code:

AWSTemplateFormatVersion: '2010-09-09'
Description: AWS Cloud-Native Fraud Detection Pipeline

Parameters:
  EnvironmentName:
    # ... (Existing parameters)

Resources:
  # ... (Existing Resources: IngestionStream, IngestionLambdaRole, IngestionLambda, IngestionApi, ApiResource, ApiMethod, FeatureStoreBucket, ModelBucket, SageMakerExecutionRole, FraudDetectionEndpoint, FraudDetectionEndpointConfig, FraudDetectionModel, TransactionTable, AlertTopic)

  # --- NEW: Prediction Lambda ---
  PredictionLambdaRole:
    Type: AWS::IAM::Role
    Properties:
      AssumeRolePolicyDocument:
        # ... (Standard Lambda AssumeRole policy)
      Policies:
        - PolicyName: kinesis-sagemaker-access
          PolicyDocument:
            Version: '2012-10-17'
            Statement:
              - Effect: Allow
                Action:
                  - kinesis:DescribeStream
                  - kinesis:GetRecords
                  - kinesis:GetShardIterator
                  - sagemaker:InvokeEndpoint
                Resource:
                  - !GetAtt IngestionStream.Arn
                  - !GetAtt FraudDetectionEndpoint.Arn # Add SageMaker Endpoint ARN
        - PolicyName: lambda-basic # ... (Standard Lambda logging policy)

  PredictionLambda:
    Type: AWS::Lambda::Function
    Properties:
      FunctionName: !Sub ${EnvironmentName}-prediction
      Runtime: python3.9
      Handler: prediction.handler # Assuming handler file is prediction.py
      Role: !GetAtt PredictionLambdaRole.Arn
      Code:
        ZipFile: | #  Replace with deployment from S3 in production
          import json
          import boto3
          kinesis = boto3.client('kinesis')
          sagemaker_runtime = boto3.client('sagemaker-runtime')

          def handler(event, context):
            for record in event['Records']:
              payload = json.loads(record['kinesis']['data'])

              # Invoke SageMaker Endpoint
              response = sagemaker_runtime.invoke_endpoint(
                EndpointName=!Sub ${EnvironmentName}-fraud-detection, # Or retrieve from environment variables for better decoupling
                ContentType='application/json',
                Body=json.dumps(payload)
              )
              prediction = json.loads(response['Body'].read().decode())

              # Invoke Response Lambda (Asynchronously recommended for decoupling)
              lambda_client = boto3.client('lambda')
              lambda_client.invoke(
                  FunctionName=!Sub ${EnvironmentName}-response,
                  InvocationType='Event',  # Asynchronous invocation
                  Payload=json.dumps({'prediction': prediction, 'transaction': payload})
              )

  # --- NEW: Response Lambda ---
  ResponseLambda:
    Type: AWS::Lambda::Function
    Properties:
        # ... (Similar structure to other Lambda functions)
      FunctionName: !Sub ${EnvironmentName}-response
      Handler: response.handler
      Role: !GetAtt ResponseLambdaRole.Arn # Use existing ResponseLambdaRole
      Code:
        ZipFile: | #  Replace with deployment from S3 in production
          import json
          import boto3
          dynamodb = boto3.resource('dynamodb')
          sns = boto3.client('sns')

          def handler(event, context):
            prediction = event['prediction']
            transaction = event['transaction']

            if prediction['is_fraudulent']:  # Replace with your model's output format
              table = dynamodb.Table('FraudTransactions')  # Or use Ref to the table
              table.put_item(Item=transaction)  # Assumes transaction has a 'transactionId'

              sns.publish(
                  TopicArn=!Ref AlertTopic,
                  Message=json.dumps({'message': 'Fraud detected!', 'transaction': transaction})
              )
            return {'statusCode': 200, 'body': 'Response processed'}



  # --- Event Source Mapping for Prediction Lambda ---
  PredictionLambdaEventSourceMapping:
    Type: AWS::Lambda::EventSourceMapping
    Properties:
      FunctionName: !Ref PredictionLambda
      EventSourceArn: !GetAtt IngestionStream.Arn
      StartingPosition: LATEST  # Or TRIM_HORIZON for processing existing data
      BatchSize: 100 # Adjust based on throughput and Lambda execution time
      Enabled: True

  # ... (Existing Outputs)

----
Key Changes and Explanations:

PredictionLambda: This Lambda function is triggered by the Kinesis stream. It retrieves records, invokes the SageMaker endpoint, and asynchronously invokes ResponseLambda. Asynchronous invocation is crucial for decoupling and scalability.

ResponseLambda: Handles the prediction result. If fraud is detected, it stores the transaction in DynamoDB and publishes an alert to SNS.

PredictionLambdaRole: A new IAM role with permissions to read from Kinesis and invoke the SageMaker endpoint.

PredictionLambdaEventSourceMapping: This resource connects the PredictionLambda to the Kinesis stream, enabling event-driven invocation.

Dynamic Partition Key (In IngestionLambda):

partition_key = transaction_data.get('transactionId', 'default_key') # Use transactionId if available, otherwise a default
response = kinesis.put_record(
    # ... other parameters
    PartitionKey=partition_key
)

This uses the transactionId (if present in the incoming data) as the partition key. Provide a default if it's missing. This is a basic example; more sophisticated key generation strategies might be beneficial.

Model Endpoint Name: I recommend passing the SageMaker endpoint name to the PredictionLambda via environment variables. This promotes better decoupling and makes the Lambda function more reusable.

Deployment & Testing:

After deploying this updated template, test the pipeline by sending transaction data to the API Gateway endpoint. Monitor CloudWatch logs for all Lambda functions to debug any issues. Ensure data is flowing through Kinesis, predictions are being made, and responses are being handled correctly. Verify DynamoDB entries and SNS alerts for fraudulent transactions.

This revised solution provides a more complete and robust fraud detection pipeline. Remember to replace placeholder values (like prediction.handler and the model's output format) with your specific implementation details. Consider adding more advanced error handling and security measures as you move towards production.