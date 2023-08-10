AWS Fargate is a serverless compute engine for building applications without managing servers.
This template will create a new collector container to send your AWS ECS Fargate traces and metrics to Logz.io using AWS OTel Collector.
ECS logs will be sent to logz.io with aws firehose

**Only `awsecscontainermetrics` receiver metrics are collected by default**

**Make your ecs fargate logs are being sent to cloudwatch**

### Before you begin
These are the prerequisites you’ll need before you can begin:

* An Amazon ECS Fargate cluster
* Applications instrumented with OpenTelemetry SDK (for Traces)

Next, you'll need to configure your CloudFormation template and point the OTLP exporter.

#### Deploy the CloudFormation template

Click on the **Launch Stack** button below to deploy the CloudFormation template. This template will create the required resources and configurations for the AWS OTel Collector.

| [![Deploy to AWS](https://dytvr9ot2sszz.cloudfront.net/logz-docs/lights/LightS-button.png)]() |  |


#### Point the OTLP exporter to the new collector container

Update the OTLP exporter configuration in your applications to point to the new collector container running in your ECS Fargate tasks.

## Parameters

The CloudFormation template requires the following parameters:
### Traces and metrics
|`ClusterName`|The name of your ECS cluster from which you want to collect metrics.|
|`TaskRoleArn`|Specifies whether to create default IAM roles (True or False).|
|`ExecutionRoleArn`|The role ARN you want to use as the ECS task role. (Optional, only required if you set `CreateIAMRoles` to False)|
|`SecurityGroups`|The role ARN you want to use as the ECS execution role. (Optional, only required if you set `CreateIAMRoles` to False)|
|`Subnets`|The list of SecurityGroupIds in your Virtual Private Cloud (VPC).|
|`LogzioTracingToken`|Your Logz.io tracing account token.|
|`LogzioMetricsToken`|Your Logz.io metrics account token.|
|`LogzioRegion`|Your Logz.io region. for example: `us`|
|`LogzioListenerUrl`|Your Logz.io listener URL. for example: `https://listener.logz.io:8053`|
### Logs
|`logzioLogsToken`|Your Logz.io logs account token.|
|`LogzioListener`|Your Logz.io listener URL. for example: `https://aws-firehose-logs-listener.logz.io`|
|`logzioType`|The log type you'll use with this shipping method. This can be a built-in log type, or your custom log type.|
|`services`|A comma-separated list of services you want to collect logs from.|
|`customLogGroups`|A comma-separated list of custom log groups you want to collect logs from.|
|`triggerLambdaTimeout`|The amount of seconds that Lambda allows a function to run before stopping it, for the trigger function.|
|`triggerLambdaMemory`|Trigger function's allocated CPU proportional to the memory configured, in MB.|
|`triggerLambdaLogLevel`|Log level for the Lambda function. Can be one of: debug, info, warn, error, fatal, panic.|
|`httpEndpointDestinationIntervalInSeconds`|The length of time, in seconds, that Kinesis Data Firehose buffers incoming data before delivering it to the destination.|
|`httpEndpointDestinationSizeInMBs`|The size of the buffer, in MBs, that Kinesis Data Firehose uses for incoming data before delivering it to the destination.|


## Resources and Configuration

The CloudFormation template creates several resources to set up the AWS OTel Collector and send metrics and traces to Logz.io. Here is a summary of the resources created by the template and the permissions granted to the IAM policies:

### AWS::ECS::TaskDefinition
The `ECSTaskDefinition` resource defines the container properties, such as the image, command, environment variables, and log configuration for the AWS OTel Collector container. It also sets the task and execution roles, network mode, and required compatibilities.

### AWS::ECS::Service
The `ECSReplicaService` resource creates an Amazon ECS service with the specified task definition, desired count, scheduling strategy, and network configuration. It associates the service with the provided ECS cluster.

### AWS::SSM::Parameter
The `CollectorConfigParameter` resource creates an AWS Systems Manager (SSM) parameter to store the collector's configuration. The configuration defines the receivers, processors, exporters, and service pipelines for traces and metrics. The applied configuration will be:

```yaml
receivers:
  awsxray:
    endpoint: 0.0.0.0:2000
    transport: udp
  otlp:
    protocols:
      grpc:
        endpoint: "0.0.0.0:4317"
      http:
        endpoint: "0.0.0.0:4318"
  awsecscontainermetrics:
processors:
  batch:
    send_batch_size: 10000
    timeout: 1s
exporters:
  logzio/traces:
    account_token: ${LOGZIO_TRACING_TOKEN}
    region: ${LOGZIO_REGION}
  prometheusremotewrite:
    endpoint: ${LOGZIO_LISTENER_URL}
    external_labels:
      aws_env: fargate-ecs
    headers:
      Authorization: "Bearer ${LOGZIO_METRICS_TOKEN}"
    resource_to_telemetry_conversion:
      enabled: true
service:
  pipelines:
    traces:
      receivers: [ awsxray,otlp ]
      processors: [ batch ]
      exporters: [ logzio/traces ]
    metrics:
      receivers: [ otlp, awsecscontainermetrics ]
      exporters: [ prometheusremotewrite ]
  telemetry:
    logs:
      level: "debug"
```
### AWS::Lambda::Function
The `logzioFirehoseSubscriptionFiltersFunction` Lambda function is designed to handle the process of filtering log data for Logz.io, and add and remove cloudwatch logs subscription filters

### AWS::Events::Rule
The `logGroupCreationEvent` rule listens for AWS API calls via CloudTrail, specifically focusing on CreateLogGroup events from the logs.amazonaws.com event source. When such an event occurs, it triggers the Logz.io subscription filter function.

### AWS::KinesisFirehose::DeliveryStream
A Kinesis Firehose delivery stream configured to send logs to Logz.io. Logs that fail to deliver are backed up to an S3 bucket.

### AWS::S3::Bucket
An S3 bucket used to store backup logs that fail to deliver via Kinesis Firehose.

### IAM Roles
1.  **ECSTaskRole**: This role is assumed by the ECS tasks and allows them to call AWS services on your behalf. The role is created with the following properties:
    *   RoleName: AWSOTelRole
2.  **ECSExecutionRole**: The ECS container agent assumes this role, allowing it to make calls to the Amazon ECS API on your behalf. The role is created with the following properties:
    *   RoleName: AWSOTelExecutionRole
3. **firehoseSubscriptionFilterLambdaRole**: This IAM role is assumed by the Lambda function when it runs. It has permissions to:
    * Create, access, and manage CloudWatch logs.
    * Manage CloudWatch Logs Subscription Filters.
    * Assume other roles.
4. **logzioS3DestinationFirehoseRole**: IAM role assumed by Kinesis Firehose when interacting with the S3 bucket for backup purposes. The role has permissions to perform basic S3 operations like `PutObject`, `GetObject`, and `ListBucket`.

### IAM Policies
1.  **AWSOpenTelemetryPolicy**: This policy is attached to the ECSTaskRole and grants the following permissions:
    *   Allows the collector to create, describe, and put log events into CloudWatch Logs.
    *   Allows the collector to send trace data to AWS X-Ray using PutTraceSegments and PutTelemetryRecords actions.
    *   Allows the collector to fetch sampling rules, targets, and statistic summaries from AWS X-Ray.
    *   Allows the collector to get parameters from AWS Systems Manager (SSM) Parameter Store.
2.  **AmazonECSTaskExecutionRolePolicy**: This managed policy is attached to the ECSExecutionRole and grants the following permissions:

    *   Allows the collector to register and deregister tasks and task definitions in Amazon ECS.
    *   Allows the collector to access and manage container instances, clusters, and services.
    *   Allows the collector to access and manage Elastic Load Balancing resources.
3.  **CloudWatchLogsFullAccess**: This managed policy is attached to the ECSExecutionRole and grants full access to Amazon CloudWatch Logs.

4.  **AmazonSSMReadOnlyAccess**: This managed policy is attached to the ECSExecutionRole and grants read-only access to the AWS Systems Manager services and resources.


By creating and attaching these IAM roles and policies, the AWS OTel Collector is granted the necessary permissions to collect and send metrics and traces from your ECS Fargate cluster to Logz.io. The template also configures the collector to use the specified Logz.io tokens, region, and listener URL.

### Ensuring Connectivity Between the Application Containers and the Collector Container

To ensure seamless connectivity between your application containers and the AWS OTel Collector container, you need to properly configure your Amazon VPC, subnets, and security groups.

### Amazon VPC and Subnets
The application containers and the AWS OTel Collector container must be launched in the same Amazon VPC and share at least one common subnet. This ensures that they can communicate with each other using private IP addresses. When deploying the CloudFormation template, make sure to provide the correct VPC subnet IDs in the `Subnets` parameter.

### Security Groups
To allow your application containers to send traces and metrics to the AWS OTel Collector container, you need to configure the security groups for both sets of containers.

1.  **Application Containers Security Group**: Modify the security group associated with your application containers to allow outbound traffic to the OTel Collector's ports (4317 for gRPC and 4318 for HTTP). Ensure that the rules are restricted to the IP address range of the VPC or the security group associated with the OTel Collector container. This configuration allows your application containers to send data to the collector container.

2.  **AWS OTel Collector Container Security Group**: Create or modify the security group for the AWS OTel Collector container to allow inbound traffic on ports 4317 (gRPC) and 4318 (HTTP) from the application containers' security group. This configuration ensures the collector container accepts incoming data from your application containers.


When deploying the CloudFormation template, provide the collector container's security group ID in the `SecurityGroups` parameter.

By properly configuring your Amazon VPC, subnets, and security groups, you can ensure that your application containers can send metrics and traces to the AWS OTel Collector container, which in turn forwards the data to Logz.io.