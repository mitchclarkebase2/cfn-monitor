# cfn-monitor

[![Build Status](https://travis-ci.org/base2Services/cfn-monitor.svg?branch=master)](https://travis-ci.org/base2Services/cfn-monitor)

## About

CloudWatch monitoring tool can query a cloudformation stack and return
monitorable resources that can be placed into a config file. This config
can then be used to generate a cloudformation stack to create and manage
cloudwatch alarms.

It is packaged as a docker container `base2/cfn-monitor` and
can be run by volume mounting in a local directory to access the config
or by using within AWS CodePipeline.

## Commands

```bash
Commands:
  cfn_monitor --version, -v   # print the version
  cfn_monitor deploy          # Deploys gerenated cfn templates to S3 bucket
  cfn_monitor generate        # Generate monitoring cloudformation templates
  cfn_monitor help [COMMAND]  # Describe available commands or one specific command
  cfn_monitor query           # Queries a cloudformation stack for monitorable resources
```

## Run With Docker

The docker image will manage the runtime environment and dependencies.
You can pass in your AWS credentials and set the region and profile with environment variables.

Example
```bash
docker run -it --rm \
  -v $(pwd):/src \
  -v $HOME/.aws:/root/.aws \
  -e AWS_REGION=us-east-1 \
  -e AWS_PROFILE=default \
  base2/cfn-monitor cfn_monitor <command> [parameters]
```

## Configuration files

There are 2 config files that can be utilised to configure your cloudwatch alarms.

- alarms.yaml - Configure the resources you want to monitor
- template.yaml - Create or override alarm templates

The bellow config structure allows for multiple monitoring stacks to be kept with
a single code repository separated by directories parameterised as `<application>`.

```
.
├── _application_1
|   ├── alarms.yaml
|   └── template.yaml
└── _application_2
    ├── alarms.yaml
    └── template.yaml
```

## Generate

The generate command takes you alarm configuration and turns it into
cloudformation templates to be deployed into AWS.

```bash
Usage:
  cfn_monitor generate

Options:
  a, [--application=APPLICATION]  # application name

Description:
  Generates cloudformation templates from the alarm configuration and output to the output/ directory.
```

### alarms.yml

This file is used to configure the AWS resources you want to monitor with CloudWatch.

```YAML
source_bucket: [Name of S3 bucket where CloudFormation templates will be deployed]
source_region: [Region of source_bucket]

resources:
  [nested stack name].[resource name]: [template name]
```

Example:

```YAML
source_bucket: source.example.com

resources:
  RDSStack.RDS: RDSInstance
```

#### Resources
Resources are referenced by the CloudFormation logical resource ID used to create them. Nested stacks are also referenced by their CloudFormation logical resource ID. See example above.

#### Target group configuration:
Target group alarms in CloudWatch require dimensions for both the target group and its associated load balancer.
To configure a target group alarm provide the logical ID of the target group (including any stacks it's nested under) followed by "/", followed by the logical ID of the load balancer (also including any stacks it's nested under).

Example:
```YAML
resources:
  LoadBalancerStack.WebDefTargetGroup/LoadBalancerStack.WebLoadBalancer: ApplicationELBTargetGroup
```

#### Custom Metrics
Custom metrics are configured with a similar syntax to resources. Use `metrics` instead of `resources`.

Example:

```YAML
metrics:
  MyCustomMetric: MyCustomMetricTemplate
```

#### Endpoints
HTTP endpoint monitoring and alerting is enabled by configuring resources under `endpoints`. Each endpoint will create a cloudwatch event, scheduled to trigger the `aws-lambda-http-check` lambda function deployed with this stack. Alarms will be configured (based on the specified template) to alert on the cloudwatch metrics generated by the lambda function.

Example:

```YAML
endpoints:
  http://www.base2services.com:
    template: HttpCheck
    statusCode: 200
    bodyRegex: 'DevOps'
```

```YAML
endpoints:
  http://www.base2services.com:
    template: HttpCheck
    statusCode: 200
    bodyRegex: 'DevOps'
    payload: id_=123
    method: POST
```

Supported parameters:

Key | Value | Default
--- | --- | ---
statusCode | The expected response code | 200
bodyRegex | A regex expected in the response body | Disabled
timeOut | A timeout value for the endpoint monitoring | 120 seconds
scheduleExpression | A cron expression used to schedule the endpoint monitoring | Every minute
environments | A string or array of environment names. Monitoring will only be deployed for these environments (if specified) | All environments


#### Multiple templates
You can specify multiple templates for the resource by providing a list/array. You may want to do this if you want to deploy some custom alarms in addition to the default alarms for a resource.

Example:
```YAML
resources:
  RDSStack.RDS: [ 'RDSInstance', 'MyRDSInstance' ]
```
or
```YAML
resources:
  RDSStack.RDS:
    - RDSInstance
    - MyRDSInstance
```

#### Auto generate alarms config for resources
You can query an existing stack for monitorable resources using the `query` command.
This will provide a list of resources in the correct config syntax,
including the nested stacks and the default templates for those resources.

Example:

```bash
Usage:
  cfn_monitor query

Options:
  a, [--application=APPLICATION]  # application name
  s, [--stack=STACK]              # cfn stack name

Description:
  This will provide a list of resources in the correct config syntax,
  including the nested stacks and the default templates for those resources.
```

Make sure you query a prod sized stack so that all conditional resources are included.
The output will list all monitorable resources found in the stack, the coverage your current `alarms.yml` config provides, and a list of any resources missing from your current `alarms.yml` config.

#### Templates
The "template" value you specify for a resource refers to either a default templates, or a custom/override template in your own `templates.yml`. This template can contain multiple alarms. The example below shows the default `RDSInstance` template, which has 2 alarms (`FreeStorageSpaceCrit` and `FreeStorageSpaceTask`). Using the `RDSInstance` template in this example will create 2 CloudWatch alarms for the `RDS` resource in the `RDSStack` nested stack.

Example: `alarms.yml`
```YAML
resources:
  RDSStack.RDS: RDSInstance
```
Example: `templates.yml`
```YAML
templates:
  RDSInstance: # AWS::RDS::DBInstance
    FreeStorageSpaceCrit:
      AlarmActions: crit
      Namespace: AWS/RDS
      MetricName: FreeStorageSpace
      ComparisonOperator: LessThanThreshold
      DimensionsName: DBInstanceIdentifier
      Statistic: Minimum
      Threshold: 50000000000
      Threshold.development: 10000000000
      EvaluationPeriods: 1
    FreeStorageSpaceTask:
      AlarmActions: task
      Namespace: AWS/RDS
      MetricName: FreeStorageSpace
      ComparisonOperator: LessThanThreshold
      DimensionsName: DBInstanceIdentifier
      Statistic: Minimum
      Threshold: 100000000000
      Threshold.development: 20000000000
      EvaluationPeriods: 1
```

#### Globally overriding a template

You can override a default template in your own `templates.yml` file if all instances of a particular resource require a non standard configuration.

Example:
```YAML
templates:
  RDSInstance:
    FreeStorageSpaceCrit:
      Threshold: 80000000000
```

This configuration will be merged over the default `RDSInstance` template resulting in the following:

```YAML
templates:
  RDSInstance:
    FreeStorageSpaceCrit:
      AlarmActions: crit
      Namespace: AWS/RDS
      MetricName: FreeStorageSpace
      ComparisonOperator: LessThanThreshold
      DimensionsName: DBInstanceIdentifier
      Statistic: Minimum
      Threshold: 80000000000
      Threshold.development: 10000000000
      EvaluationPeriods: 1
    FreeStorageSpaceTask:
      AlarmActions: task
      Namespace: AWS/RDS
      MetricName: FreeStorageSpace
      ComparisonOperator: LessThanThreshold
      DimensionsName: DBInstanceIdentifier
      Statistic: Minimum
      Threshold: 100000000000
      Threshold.development: 20000000000
      EvaluationPeriods: 1
```

#### Create a custom template
If the default template for your resource is completely inappropriate, you can create your own custom template in the `monitoring/templates.yml` file.

Example:

```YAML
templates:
  MyRDSInstance:
    DatabaseConnections:
      AlarmActions: crit
      Namespace: AWS/RDS
      MetricName: DatabaseConnections
      ComparisonOperator: MoreThanThreshold
      DimensionsName: DBInstanceIdentifier
      Statistic: Average
      Threshold: 20
      EvaluationPeriods: 5
```

#### Inherit a template
If you have multiple instances of a particular resource and you want to adjust the configuration for only some of them, you can create your own custom template which inherits the configuration of a default template.

Example:
```YAML
templates:
  MyRDSInstance:
    template: RDSInstance
    FreeStorageSpaceCrit:
      Threshold: 80000000000
```
The above example creates a new template `MyRDSInstance` which can now be used by one or many resources. The `MyRDSInstance` template inherits all of the alarms and configuration from `RDSInstance`, but sets `Threshold` to `80000000000` for the `FreeStorageSpaceCrit` alarm.

#### Environment type mappings
You can create environment type mappings if alarm configurations need to differ between different environment types. This may be useful in situations where development type environments are running different resource quantities or sizes.

Example:
```YAML
templates:
  RDSInstance:
    FreeStorageSpaceCrit:
      Threshold: 40000000000
      Threshold.development: 20000000000
      Threshold.staging: 30000000000
      EvaluationPeriods: 5
```
The above example shows different `Threshold` values for `EnvironmentType` values of `production` (default), `development` or `staging`.
Any value can be specified using the `.envType` syntax and the necessary mappings and `EnvironmentType` will be generated when rendered.
The `EvaluationPeriods` value for `development` and `staging` type environments will be `5` in the above example as no `.envType` values where provided for this parameter.

Supported Parameters:
Parameter | Mapping support
--- | ---
ActionsEnabled | true
AlarmActions | false
AlarmDescription | false
ComparisonOperator | true
Dimensions | false
EvaluateLowSampleCountPercentile | false
EvaluationPeriods | true
ExtendedStatistic | false
InsufficientDataActions | false
MetricName | true
Namespace | true
OKActions | false
Period | true
Statistic | true
Threshold | true
TreatMissingData | true
Unit | false

#### Template variables

The following variables can be used in templates:

Variable Key | Variable Value
--- | ---
${name} | Metric/Resource Name (from alarms.yml)
${metric} | Metric Name (from alarms.yml)
${resource} | Resource Name (from alarms.yml)
${templateName} | Template Name (from templates.yml)
${alarmName} | Alarm Name (from templates.yml)

Example:

`alarms.yml`
```YAML
metrics:
  Metric1: MyCustomMetric
```
`templates.yml`
```YAML
templates:
  MyCustomMetric:
    ItemCountHigh:
      MetricName: ${metric}
      AlarmDescription: '#{templateName} #{alarmName} - #{name}'
```
Result:
```YAML
templates:
  MyCustomMetric:
    ItemCountHigh:
      MetricName: Metric1
      AlarmDescription: 'MyCustomMetric ItemCountHigh - Metric1'
```

#### Alarm Actions
There are 3 classes of alarm actions: `crit`, `warn` and `task`.

Action | Process
--- | ---
crit | Alert on-call technician
warn | Create alarm in pager service but do not alert on-call technician
task | Create support ticket for investigation

An SNS topic is required per alarm action, these topics and their subscriptions are managed outside this stack

### Deployment

The rendered CloudFormation templates should be deployed to `[source_bucket]/cloudformation/monitoring/`.

```bash
Usage:
  cfn_monitor deploy

Options:
  a, [--application=APPLICATION]  # application name

Description:
  Deploys gerenated cloudformation templates to the specified S3 source_bucket
```

Launch the Monitoring stack in the desired account with the following CloudFormation parameters:

Parameter Key | Parameter Value
--- | ---
EnvironmentType | `production` / `development` / `custom env type`
MonitoredStack | The name of the stack you want monitored. EG `prod`
MonitoringDisabled | `true` for disables alerts, `false` for enabled alerts
SnsTopicCrit | SNS topic used by crit type alarms
SnsTopicTask | SNS topic used by task type alarms
SnsTopicWarn | SNS topic used by warn type alarms

### Disabling Monitoring
It is possible to globally disable / snooze / downtime all alarms by setting the `MonitoringDisabled` CloudFormation parameter to `true`.
This will disable alarm actions without removing removing them.

### Disabling and excluding alarms
To disable or prevent creation of a specific alarm, specify either of the following parameters:
```YAML
templates:
  MyAutoScalingGroup:
    template: AutoScalingGroup
    CPUUtilizationHighBase:
      CreateAlarm: false    # Don't create the alarm
      DisableAlarm: true    # Create the alarm but disable it
```
