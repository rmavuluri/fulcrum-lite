# Tier 1: read-only AWS CLI commands for MSK

Every command here only reads. Add `--region <region>` (and `--profile <name>` if the user gave
one) and `--output json --no-cli-pager` to each. Placeholders are in `<angle brackets>`. Glue
commands are in [glue.md](glue.md) and MSK Connect commands in [connect.md](connect.md).

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
the Kafka version, broker count and instance type (`Provisioned`), authentication, encryption, the
enhanced monitoring level (`Provisioned.EnhancedMonitoring`) and the MSK configuration in use
(`Provisioned.CurrentBrokerSoftwareInfo.ConfigurationArn` and `ConfigurationRevision`).

## Brokers

```
aws kafka list-nodes --cluster-arn <cluster-arn> --query "NodeInfoList[].{Id:BrokerNodeInfo.BrokerId,Type:InstanceType,Subnet:BrokerNodeInfo.ClientSubnet,Endpoints:BrokerNodeInfo.Endpoints}"
aws kafka get-bootstrap-brokers --cluster-arn <cluster-arn>
```

`get-bootstrap-brokers` returns one string per auth method: `BootstrapBrokerStringSaslIam`
(port 9098), `BootstrapBrokerStringSaslScram` (9096), `BootstrapBrokerStringTls` (9094),
`BootstrapBrokerString` (plaintext, 9092), and `...Public...` variants (91xx) if public access is
on. Tier 2 uses the IAM string, or the TLS or plaintext string for unauthenticated access. SCRAM
and mutual TLS aren't supported in Tier 2.

### Broker configuration (the cluster's MSK configuration)

Take the ARN and revision from `describe-cluster-v2`. If there is none, the cluster runs the MSK
defaults.

```
aws kafka describe-configuration --arn <configuration-arn>
aws kafka describe-configuration-revision --arn <configuration-arn> --revision <n> --query ServerProperties --output text
```

`ServerProperties` is Base64; decode it (pipe to `base64 --decode`) to show the
`server.properties` lines. Point out `auto.create.topics.enable`, `default.replication.factor`,
`min.insync.replicas`, `num.partitions`, `log.retention.hours` / `log.retention.ms` and
`unclean.leader.election.enable` when present.

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
the cluster isn't reporting lag. Say so instead of guessing. For members, hosts and lag per
partition, use Tier 2.

## Health, broker and throughput metrics (CloudWatch)

Discover what the cluster reports, then read the values with `get-metric-statistics` using the
exact dimensions `list-metrics` returned (`Cluster Name`, plus `Broker ID` or `Topic`):

```
aws cloudwatch list-metrics --namespace AWS/Kafka --dimensions Name="Cluster Name",Value=<cluster-name> --query "Metrics[].MetricName" --output text
```

| Metric | Level | Meaning | Healthy |
|---|---|---|---|
| `ActiveControllerCount` | cluster (sum across brokers) | Controllers | exactly 1 |
| `OfflinePartitionsCount` | cluster | Partitions with no leader | 0 |
| `GlobalPartitionCount`, `GlobalTopicCount` | cluster | Partitions (excluding replicas) and topics | — |
| `UnderReplicatedPartitions` | broker | Partitions behind on replication | 0 |
| `UnderMinIsrPartitionCount` | broker | Below `min.insync.replicas` (producers with `acks=all` fail) | 0 |
| `CpuUser` + `CpuSystem` | broker | CPU percent | under about 60 together |
| `KafkaDataLogsDiskUsed` | broker | Data disk percent | under about 85 |
| `PartitionCount`, `LeaderCount` | broker | Partition replicas and leaders on the broker | evenly spread |
| `ConnectionCount`, `ClientConnectionCount` | broker | Open connections | steady |
| `BytesInPerSec`, `BytesOutPerSec`, `MessagesInPerSec` | broker, or topic (needs per-topic monitoring) | Throughput | — |

For a trend, use `--period 300` over the last 24 hours, or `--period 3600` over 7 days.

## Insights

- **Health first:** cluster not `ACTIVE`, `OfflinePartitionsCount` > 0, `ActiveControllerCount`
  ≠ 1, then under-replicated or under-min-ISR partitions, and which brokers they sit on.
- **Hot or unbalanced brokers:** compare CPU, disk, `PartitionCount`, `LeaderCount` and
  `BytesInPerSec` across brokers. One broker clearly above the rest points to uneven partition or
  leader placement, or a hot topic.
- **Disk:** if `KafkaDataLogsDiskUsed` grows steadily, estimate when it reaches 85% from the
  growth per day, and mention retention, storage auto scaling or more storage as options.
- **Too many partitions:** compare `PartitionCount` per broker with the AWS recommended maximum
  for the broker size (see "Best practices" in the MSK developer guide; for example roughly 300
  for `kafka.t3.small`, 1000 for `kafka.m5.large`). Say these are AWS guidelines.
- **Risky configuration:** `auto.create.topics.enable=true`, `default.replication.factor` or a
  topic replication factor of 1, `min.insync.replicas` of 1 with replication factor 3,
  `unclean.leader.election.enable=true`.
- **Lag: stuck or just slow?** Lag that keeps rising without ever dropping only means the group
  consumes more slowly than producers write. It doesn't mean the group has stopped. To tell the
  two apart, compare how fast lag grows with the topic's incoming message rate
  (`MessagesInPerSec` for that topic, per minute). If lag grows at roughly the full incoming rate,
  the group isn't consuming at all (stuck, or no members). If it grows more slowly, the group is
  consuming, just too slowly: the difference is its consume rate. CloudWatch can't show members;
  confirm with Tier 2 (`--describe --group <g> --members`) when it's available.
- **Groups that move together:** if several groups on one topic jump at the same minute, look at
  the topic's incoming rate first. A burst from producers explains it better than every consumer
  failing at once.
- **Recent changes:** if something looks wrong, check `list-cluster-operations-v2` for an
  operation that started around the same time.

## Never run

Anything else: `create-*`, `update-*`, `delete-*`, `put-*`, `register-*`, `reboot-*`,
`tag-*`, `untag-*`, `batch-*`, `start-*`, `stop-*` or any command that changes state. This plugin is
read-only.
