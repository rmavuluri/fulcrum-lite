# Read-only AWS CLI commands

Every command here only reads. Add `--region <region>` (and `--profile <name>` if the user gave
one) and `--output json --no-cli-pager` to each. Placeholders are in `<angle brackets>`.

## Setup and identity

```
aws --version
aws configure list
aws sts get-caller-identity
```

If `aws` is missing, tell the user to install AWS CLI v2. If `get-caller-identity` fails, the user
needs to sign in (for example `aws sso login --profile <name>`), which they run themselves.

## Clusters

```
aws kafka list-clusters-v2 --query "ClusterInfoList[].{Name:ClusterName,Arn:ClusterArn,Type:ClusterType,State:State}"
aws kafka describe-cluster-v2 --cluster-arn <cluster-arn>
aws kafka list-cluster-operations-v2 --cluster-arn <cluster-arn> --max-items 10
```

`describe-cluster-v2` gives `ClusterInfo.State` (`ACTIVE`, `UPDATING`, `HEALING`, `FAILED`…),
the Kafka version, broker count and instance type (`Provisioned`), authentication, and the
enhanced monitoring level (`Provisioned.EnhancedMonitoring`).

## Brokers

```
aws kafka list-nodes --cluster-arn <cluster-arn> --query "NodeInfoList[].BrokerNodeInfo.{Id:BrokerId,Subnet:ClientSubnet,Endpoints:Endpoints}"
aws kafka get-bootstrap-brokers --cluster-arn <cluster-arn>
```

## Topics

```
aws kafka list-topics --cluster-arn <cluster-arn> [--topic-name-filter <prefix>]
aws kafka describe-topic --cluster-arn <cluster-arn> --topic-name <topic>
aws kafka describe-topic-partitions --cluster-arn <cluster-arn> --topic-name <topic>
```

- `list-topics` returns `TopicName`, `PartitionCount`, `ReplicationFactor` and
  `OutOfSyncReplicaCount` per topic. A non-zero `OutOfSyncReplicaCount` means under-replication.
- `describe-topic` returns `Configs` as Base64. Decode it to show retention and cleanup policy,
  for example with `--query Configs --output text` piped to `base64 --decode`.
- `describe-topic-partitions` returns `Leader`, `Replicas` and `Isr` per partition. A partition
  whose `Isr` is shorter than `Replicas` is under-replicated. A missing or `-1` leader means offline.

## Consumer groups and lag (CloudWatch)

The AWS CLI can't connect to Kafka, so consumer groups are found from the lag metrics MSK
publishes to CloudWatch (namespace `AWS/Kafka`). Discover the groups first:

```
aws cloudwatch list-metrics --namespace AWS/Kafka --metric-name SumOffsetLag --dimensions Name="Cluster Name",Value=<cluster-name>
```

Each result has the dimensions `Cluster Name`, `Consumer Group` and `Topic`. Then read lag for a
group and topic over the last 30 minutes (compute the UTC times yourself, ISO 8601):

```
aws cloudwatch get-metric-statistics --namespace AWS/Kafka --metric-name SumOffsetLag --dimensions Name="Cluster Name",Value=<cluster-name> Name="Consumer Group",Value=<group> Name=Topic,Value=<topic> --start-time <start> --end-time <end> --period 60 --statistics Maximum
```

Use `MaxOffsetLag` (worst partition, messages) or `EstimatedMaxTimeLag` (seconds) the same way.
If `list-metrics` returns nothing, either no consumer group has committed offsets recently, or
the cluster isn't reporting lag. Say so instead of guessing.

## Health and throughput metrics (CloudWatch)

Discover what the cluster reports, then read the values with `get-metric-statistics` using the
exact dimensions `list-metrics` returned:

```
aws cloudwatch list-metrics --namespace AWS/Kafka --dimensions Name="Cluster Name",Value=<cluster-name> --query "Metrics[].MetricName" --output text
```

| Metric | Meaning | Healthy |
|---|---|---|
| `ActiveControllerCount` | Controllers (sum across brokers) | exactly 1 |
| `OfflinePartitionsCount` | Partitions with no leader | 0 |
| `UnderReplicatedPartitions` | Per broker | 0 |
| `UnderMinIsrPartitionCount` | Per broker, below `min.insync.replicas` | 0 |
| `CpuUser`, `KafkaDataLogsDiskUsed` | Per broker, percent | under about 60 and 85 |
| `BytesInPerSec`, `MessagesInPerSec` | Per topic (needs per-topic monitoring) | — |

## Glue Schema Registry

```
aws glue list-registries
aws glue list-schemas --registry-id RegistryName=<registry>
aws glue get-schema --schema-id RegistryName=<registry>,SchemaName=<schema>
aws glue list-schema-versions --schema-id RegistryName=<registry>,SchemaName=<schema>
aws glue get-schema-version --schema-id RegistryName=<registry>,SchemaName=<schema> --schema-version-number LatestVersion=true
aws glue get-schema-version --schema-id RegistryName=<registry>,SchemaName=<schema> --schema-version-number VersionNumber=<n>
aws glue get-schema-versions-diff --schema-id RegistryName=<registry>,SchemaName=<schema> --first-schema-version-number VersionNumber=<old> --second-schema-version-number VersionNumber=<new> --schema-diff-type SYNTAX_DIFF
```

`get-schema` shows the data format, compatibility mode and latest version number. Explain a
diff in plain words (fields added, removed or changed type) as well as showing it.

## Never run

Anything else: `create-*`, `update-*`, `delete-*`, `put-*`, `register-*`, `reboot-*`,
`tag-*`, `untag-*`, `batch-*` or any command that changes state. This plugin is read-only.
