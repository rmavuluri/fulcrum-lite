---
name: kafka-insights
description: Answer questions about Amazon MSK / Kafka clusters and AWS Glue Schema Registry schemas by running read-only AWS CLI commands. Use when the user asks about MSK clusters, brokers, Kafka topics, partitions, replication or cluster health, consumer lag, or Glue schemas, versions, compatibility or schema changes.
when_to_use: 'Examples: "is my Kafka cluster healthy?", "how many partitions does clickstream have?", "is the orders topic under-replicated?", "which consumer group is lagging?", "what changed in the OrderCreated schema?", "list my MSK clusters"'
---

# Kafka insights with the AWS CLI

Answer MSK and Glue Schema Registry questions by running **read-only** `aws` CLI commands in the
user's terminal. The exact commands are in [commands.md](commands.md). Read it before running
anything.

## Rules

1. **Read-only, always.** Run only the commands listed in commands.md (`list-*`, `describe-*`,
   `get-*` for `kafka`, `glue` and `cloudwatch`, plus `aws sts get-caller-identity` and
   `aws configure list`). Never run a command that creates, updates, deletes, reboots, puts,
   registers or tags anything, even if the user asks. If they ask for a change, explain what
   command they could run themselves and label it clearly as theirs to run.
2. **Ask before guessing.** If the region or cluster is unclear, run `aws configure list` and
   `aws kafka list-clusters-v2` and ask which cluster to use when there is more than one. Reuse the
   user's answer (region, profile, cluster ARN) for the rest of the conversation.
3. **Pass the user's context explicitly.** Add `--region <region>` to every command, and
   `--profile <name>` if the user named a profile. Add `--output json --no-cli-pager`.
4. **Few commands.** Run the smallest set that answers the question. Use `--query` to trim output
   and filters such as `--topic-name-filter` on large clusters.
5. **Never print secrets.** Don't run commands that print credentials, and don't read files under
   `~/.aws`. `aws configure list` shows only masked values and is fine.

## Pick the command for the question

| Question | Commands (see commands.md) |
|---|---|
| Which clusters do I have? | `kafka list-clusters-v2` |
| Cluster details, version, brokers, auth, endpoints | `kafka describe-cluster-v2`, `kafka list-nodes`, `kafka get-bootstrap-brokers` |
| Is the cluster healthy? | Cluster `State` from `describe-cluster-v2`, `OutOfSyncReplicaCount` from `list-topics`, and the CloudWatch health metrics (`OfflinePartitionsCount`, `UnderReplicatedPartitions`, `ActiveControllerCount`) |
| Find topics, partition counts, replication factor | `kafka list-topics` (with `--topic-name-filter`) |
| Topic configuration (retention, cleanup policy) | `kafka describe-topic`, then decode `Configs` (Base64) |
| Partitions, leaders, ISR, under-replication of a topic | `kafka describe-topic-partitions` |
| Which consumer group is lagging? | CloudWatch `SumOffsetLag` / `MaxOffsetLag` / `EstimatedMaxTimeLag` (discover groups with `list-metrics` first) |
| Topic throughput | CloudWatch `BytesInPerSec` / `MessagesInPerSec` per topic (needs per-topic monitoring) |
| Recent cluster operations (upgrades, scaling) | `kafka list-cluster-operations-v2` |
| Registries, schemas, compatibility, versions | `glue list-registries`, `glue list-schemas`, `glue get-schema`, `glue list-schema-versions` |
| Schema definition | `glue get-schema-version` |
| What changed between schema versions? | `glue get-schema-versions-diff` |
| Access denied or credential errors | `aws sts get-caller-identity`, then point to `iam/fulcrum-lite-readonly-policy.json` in this plugin |

## How to answer

- Lead with the direct answer in one or two sentences, then a short markdown table of the relevant
  rows. Say when the data was fetched (the current time) and which cluster or registry it came
  from.
- **Under-replicated** means a partition's ISR is smaller than its replica list; **offline** means
  it has no leader. Call offline partitions out first.
- **Consumer lag** from CloudWatch is in messages (`SumOffsetLag`, `MaxOffsetLag`) or seconds
  (`EstimatedMaxTimeLag`), and is reported about once a minute. Say it comes from CloudWatch, give
  the time window, and say that lag growing across datapoints means consumers can't keep up.

## Limits to be honest about

- **No consumer group members or live offsets.** The AWS CLI doesn't talk to the Kafka brokers,
  so it can't list group members, client IDs or per-partition committed offsets. Lag comes only
  from CloudWatch, and only for groups MSK reports metrics for.
- **No producer list.** Kafka has no API that lists producers. Per-topic ingress
  (`BytesInPerSec`) is the best evidence, and only with per-topic monitoring enabled.
- **Topic APIs need a supported cluster.** If `list-topics` fails as unsupported for the cluster,
  say so and fall back to `describe-cluster-v2` and the CloudWatch metrics.
- **No message contents.** Never try to read messages. This plugin has no way to do it.
