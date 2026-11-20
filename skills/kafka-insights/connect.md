# Connectors: MSK Connect and Kafka Connect

Two kinds of connectors are common with MSK:

- **MSK Connect**, the AWS-managed service. Read it with the AWS CLI (Tier 1).
- **Self-managed Kafka Connect** (on EC2, ECS, EKS or elsewhere). Read it with `GET` requests to its
  REST API, which needs network access to a Connect worker (Tier 2).

Ask which one the user means if it isn't clear. A quick `aws kafkaconnect list-connectors` shows
whether they have any MSK Connect connectors in the region.

## MSK Connect (AWS CLI)

Add `--region <region>` (and `--profile <name>`) and `--output json --no-cli-pager` to each.

```
aws kafkaconnect list-connectors [--connector-name-prefix <prefix>] --query "connectors[].{Name:connectorName,State:connectorState,Arn:connectorArn,Created:creationTime}"
aws kafkaconnect describe-connector --connector-arn <connector-arn>
aws kafkaconnect list-custom-plugins
aws kafkaconnect describe-custom-plugin --custom-plugin-arn <plugin-arn>
aws kafkaconnect list-worker-configurations
aws kafkaconnect describe-worker-configuration --worker-configuration-arn <arn>
```

- `connectorState` is `RUNNING`, `CREATING`, `UPDATING`, `DELETING` or `FAILED`.
  `stateDescription` explains a failure.
- `describe-connector` also shows the capacity (provisioned workers or auto scaling min/max
  workers and MCUs), the Kafka cluster and its auth, the plugin, the Kafka Connect version, and
  `logDelivery` (where worker logs go).
- `connectorConfiguration` and the worker configuration's `propertiesFileContent` (Base64) can
  hold secrets. Show only the keys that matter (`connector.class`, `tasks.max`, `topics`, converters,
  `errors.*`), and mask any value whose key contains `password`, `secret`, `token`, `key` or
  `credential`.

### Metrics

```
aws cloudwatch list-metrics --namespace AWS/KafkaConnect --dimensions Name=ConnectorName,Value=<connector-name> --query "Metrics[].MetricName" --output text
```

Then read values with `get-metric-statistics` (same form as in aws-commands.md, namespace
`AWS/KafkaConnect`, dimension `ConnectorName`). Look for task counts (running versus errored),
CPU and memory use, worker count, and record rates (source poll rate, sink send rate). Use the
metric names `list-metrics` returns rather than guessing.

### Error logs

If `logDelivery.workerLogDelivery.cloudWatchLogs` is enabled, read only recent error lines:

```
aws logs filter-log-events --log-group-name <log-group> --start-time <epoch-millis> --filter-pattern ERROR --max-items 20 --query "events[].message"
```

Summarize the errors (exception class, failing task, the main message). Don't repeat record
contents if a log line happens to include them. If logs go to S3 or Firehose, say where and that
this plugin doesn't read them.

## Self-managed Kafka Connect (REST API, GET only)

Ask the user for the Connect REST URL (often `http://<worker-host>:8083`). Use only `curl` with
`GET`, a timeout, and no request body:

```
curl -sS --max-time 15 <connect-url>/
curl -sS --max-time 15 "<connect-url>/connectors?expand=status"
curl -sS --max-time 15 <connect-url>/connectors/<name>/status
curl -sS --max-time 15 <connect-url>/connectors/<name>/topics
curl -sS --max-time 15 <connect-url>/connector-plugins
```

- `/` shows the Kafka Connect version and the Kafka cluster ID.
- `?expand=status` gives every connector's state and every task's state, worker and, for failed
  tasks, the `trace` (a Java stack trace). Summarize the trace: the exception and its message.
- `/connectors/<name>/topics` lists the topics the connector has used (Kafka 2.5+).
- Avoid `?expand=info`, `/config` and `/tasks`: they return full configuration, which often
  contains secrets. If the user asks for configuration, fetch `/connectors/<name>/config` once and
  mask secret values before showing anything.
- If the API needs authentication, ask the user to put the credentials in `~/.netrc` and add
  `--netrc` to the command. Never put a password in the command line.
- If `curl` times out or the connection is refused, the worker isn't reachable from this machine.
  Say so and stop.

## Insights

- **Failed first:** connectors or tasks in `FAILED`, then `PAUSED` or `UNASSIGNED`, then
  connectors where only some tasks run.
- **Common failure causes** to suggest from the error text: authentication or authorization to
  the source or sink, a missing topic with auto-create off, serialization errors (wrong converter
  or schema registry settings, often `DataException` or `SerializationException`), a missing
  plugin class, and network timeouts to the external system.
- **Throughput and lag:** a sink connector is a consumer group, normally named
  `connect-<connector-name>`. Check its lag with the consumer group commands in kafka-admin.md or
  the CloudWatch lag metrics in aws-commands.md.
- **Capacity:** MSK Connect at its maximum workers with high CPU or memory needs more capacity;
  `tasks.max` higher than the topic's partition count leaves sink tasks idle.

## Never run

- `aws kafkaconnect` `create-*`, `update-*`, `delete-*` or any command that changes state.
- `curl` with `-X POST`, `-X PUT`, `-X DELETE`, `-X PATCH`, `-d` or `--data`, or any request to
  `/restart`, `/pause`, `/resume`, `/stop` or `/offsets` changes.
