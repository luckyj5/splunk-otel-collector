# AWS Fargate Deployment
Familiarity with AWS Fargate (Fargate) is assumed. Consult the 
[User Guide for AWS Fargate](https://docs.aws.amazon.com/AmazonECS/latest/userguide/what-is-fargate.html)
for further reading.

Unless stated otherwise, it is assumed that the
[Splunk OpenTelemetry Collector](https://github.com/signalfx/splunk-otel-collector)
(Collector) is deployed to an ECS Task as a **sidecar** container alongside monitored
application containers.

The instructions herein apply to release v0.30.0 and above of the Collector which corresponds
to image tag 0.30.0 and above in the image repository
[here](https://quay.io/repository/signalfx/splunk-otel-collector?tab=tags).

Overall, you deploy the Collector to Fargate by adding a container definition for it in which you:
- specify the Collector image.
- specify the configuration file to use by setting environment variable `SPLUNK_CONFIG` with the file path.
- set `SPLUNK_CONFIG` with `/etc/otel/collector/fargate_config.yaml` to use the default configuration file.
- optionally use environment variable `SPLUNK_CONFIG_YAML` instead of `SPLUNK_CONFIG` to specify
  configuration YAML directly (No need for a file).
- use extension `ecs_observer` in custom configuration to discover targets.
- enable ECS read-only permissions in the task role when using `ecs_observer`.
- filter the targets for `ecs_observer` to be within task when the Collector is a sidecar.
- note that `ecs_observer` is currently limited to Prometheus targets.

## Default Configuration
In the container definition for the Collector:
- Map the endpoint ports in the default configuration file (i.e. `13133`, `6060`,
  `55679`, `14250`, `6832`, `6831`, `14268`, `8888`, `7276`, `9943`, `9411`, `9080`).
- Assign the default configuration file path `/etc/otel/collector/fargate_config.yaml` to environment variable `SPLUNK_CONFIG`.
- Optionally assign a list of metrics you want excluded to environment variable `METRICS_TO_EXCLUDE`.
- Optionally assign a list of images whose metrics you want excluded to environment variable `IMAGES_TO_EXCLUDE`.

The default configuration file is located at `/etc/otel/collector/fargate_config.yaml`
in the Collector image according to the Collector image Dockerfile
[here](../../cmd/otelcol/Dockerfile). See the contents of the default configuration file
[here](../../cmd/otelcol/config/collector/fargate_config.yaml). Note that receivers
`hostmetrics` and `smartagent/ecs-metadata` are specified.

## Custom Configuration
For the Collector to pick up custom configuration in a file, you need to assign the file path
to environment variable `SPLUNK_CONFIG` in the container definition for the Collector. In
Fargate, this means having your custom configuration file in a volume attached to the task
and assigning the path to `SPLUNK_CONFIG`.

### ecs_observer
Add extension
[Amazon Elastic Container Service Observer](https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/extension/observer/ecsobserver#amazon-elastic-container-service-observer)
(`ecsobserver` or `ecs_observer`) to your custom configuration in order to discover metrics
targets. The `ecs_observer` can discover targets in running tasks, filtered by service names,
task definitions and container labels. The `ecs_observer` is currently limited to discovering
Prometheus targets and requires the read-only permissions below. You can add the permissions
to the task role by adding them to a customer-managed policy that is attached to the task role.
```text
ecs:List*
ecs:Describe*
```

In the example below, the `ecs_observer` is configured to find Prometheus targets in
cluster `lorem-ipsum-cluster` of region `us-west-2` where the task arn pattern is 
`^arn:aws:ecs:us-west-2:906383545488:task-definition/lorem-ipsum-task:[0-9]+$`,
and write the results in file `/etc/ecs_sd_targets.yaml`. The `prometheus` receiver is
configured to read targets from the results file. Note that the values for `access_token`
and `realm` are read from environment variables `SPLUNK_ACCESS_TOKEN` and `SPLUNK_REALM`
respectively, which must be specified in your container definition.

```yaml
extensions:
  ecs_observer:
    refresh_interval: 10s
    cluster_name: 'lorem-ipsum-cluster'
    cluster_region: 'us-west-2'
    result_file: '/etc/ecs_sd_targets.yaml'
    task_definitions:
      - arn_pattern: "^arn:aws:ecs:us-west-2:906383545488:task-definition/lorem-ipsum-task:[0-9]+$"
        metrics_ports: [9113]
        metrics_path: /metrics
receivers:
  signalfx:
  prometheus:
    config:
      scrape_configs:
        - job_name: 'lorem-ipsum-nginx'
          scrape_interval: 10s
          file_sd_configs:
            - files:
                - '/etc/ecs_sd_targets.yaml'
processors:
  batch:
  resourcedetection:
    detectors: [ecs]
    override: false    
exporters:
  signalfx:
    access_token: ${SPLUNK_ACCESS_TOKEN}
    realm: ${SPLUNK_REALM}
service:
  extensions: [ecs_observer]
  pipelines:
    metrics:
      receivers: [prometheus]
      processors: [batch, resourcedetection]
      exporters: [signalfx]
```
For the sidecar deployment of the Collector, that is, the Collector and the monitored
containers are in the same task, the discovered targets must be within the task. Note that
the task arn pattern for `ecs_observer` in the example above filters the discovered targets
to running revisions of task `lorem-ipsum-task`. This means that the `ecs_observer` will
discover targets outside the task in which the Collector is running when multiple revisions
of task `lorem-ipsum-task` are running. One way to solve this is to use the complete task arn
as shown below. This however adds the overhead of updating the task arn pattern in your 
configuration to keep pace with task revisions.

```yaml
...
     - arn_pattern: "^arn:aws:ecs:us-west-2:906383545488:task-definition/lorem-ipsum-task:3$"
...
```

### Direct Configuration
Since access to the filesystem is not readily available in Fargate, it may be convenient to
specify the configuration YAML directly at the commandline using environment variable
`SPLUNK_CONFIG_YAML`. For instance, you could provide custom configuration by:
- Creating a parameter in the AWS Systems Manager Parameter Store for your custom configuration
  YAML.
- Use the `ValueFrom` feature to assign the parameter to `SPLUNK_CONFIG_YAML` in the container
  definition for the Collector.
- Add policy `AmazonSSMReadOnlyAccess` to the task role in order for the task to have read
  access to the Parameter Store.

### Standalone Task
Extension `ecs_observer` is capable of scanning for targets in the entire cluster for a given
region. This allows you to deploy the Collector container in a standalone task separate from the 
monitored application containers and collect telemetry data. This is in contrast to the sidecar 
deployment whereby the Collector container is in the same task as the monitored application
containers. Do not configure the ECS
[resourcedetection](https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/processor/resourcedetectionprocessor#resource-detection-processor) 
processor for the standalone task since it would detect resources in the standalone Collector
task itself which you are not monitoring.