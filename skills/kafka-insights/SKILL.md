---
name: kafka-insights
description: Answer questions about Amazon MSK / Kafka clusters, brokers, topics, consumer groups, connectors (MSK Connect and Kafka Connect) and AWS Glue Schema Registry schemas by running read-only AWS CLI commands and, when available, read-only Kafka admin tools. Use when the user asks about MSK clusters, brokers, Kafka topics, partitions, replication or cluster health, consumer groups, members or lag, connectors and their tasks, or Glue schemas, versions, compatibility, schema changes or which topics have schemas.
when_to_use: 'Examples: "is my Kafka cluster healthy?", "how many partitions does clickstream have?", "is the orders topic under-replicated?", "which consumer group is lagging?", "who is consuming payments and on which partitions?", "is my S3 sink connector running?", "which broker has the most disk used?", "would this new schema break consumers?", "what changed in the OrderCreated schema?", "which topics have no schema?", "list my MSK clusters"'
---

# Kafka insights

Answer questions about MSK, Kafka Connect and the Glue Schema Registry by running **read-only**
commands in the user's terminal. The user approves every command. There are two tiers:

| Tier | Tools | Works when | Reference |
|---|---|---|---|
| 1. AWS CLI | `aws kafka`, `aws cloudwatch`, `aws glue`, `aws kafkaconnect` | AWS CLI v2 is signed in. Works from anywhere. | [aws-commands.md](aws-commands.md), [glue.md](glue.md), [connect.md](connect.md) |
| 2. Kafka admin | `kafka-topics`, `kafka-consumer-groups`, `kafka-configs`, `kafka-log-dirs`, `kafka-get-offsets`, `kafka-broker-api-versions`, Kafka Connect REST `GET` | The Apache Kafka CLI tools are installed **and** this machine can reach the brokers (inside the VPC, over VPN, or with public access). | [kafka-admin.md](kafka-admin.md), [connect.md](connect.md) |

Read the reference file for a topic before running anything from it. If the user asks what this
plugin can do or wants to check their setup, use the `getting-started` skill instead.

## Rules

1. **Read-only, always.** Run only the commands listed in the reference files. Never run a
   command that creates, alters, deletes, resets, reassigns, restarts, pauses, produces, consumes,
   registers or tags anything, even if the user asks. Each file ends with a "Never run" list. If
   the user asks for a change, show the command they could run themselves and label it clearly as
   theirs to run.
2. **Never read message contents.** Don't run `kafka-console-consumer`, `kafka-console-producer`,
   `kafka-dump-log` or any tool that reads or writes records. Topic metadata, offsets and lag only.
3. **Never install software or edit files yourself.** If a tool is missing, give the user the
   setup steps from the reference file to run themselves.
4. **Never print secrets.** Don't read files under `~/.aws`, don't print the contents of a Kafka
   client properties file, and mask any configuration value whose key contains `password`,
   `secret`, `token`, `key` or `credential` (show `****`). Never put a password on a command line.
5. **Ask before guessing.** If the region, cluster, registry or Connect URL is unclear, list the
   choices and ask. Reuse the user's answers (region, profile, cluster ARN, bootstrap servers,
   client config path, Connect URL) for the rest of the conversation.
6. **Few, scoped commands.** Run the smallest set that answers the question. Prefer one topic or
   one group over `--all-groups` / every topic on large clusters; trim AWS output with `--query`.
7. **One plain command per call.** Don't chain commands with `;` or `&&`, and don't use `for`
   loops, subshells or shell variables. Each command should be easy to read and approve, and
   should match a user's allowlist such as `Bash(aws glue get-*)`. Run independent commands as
   separate calls in parallel. A single pipe into `base64 --decode` or `grep` is fine.

## Choose the tier

1. Start with **Tier 1** for anything it can answer: clusters, brokers, topics, partitions, health,
   CloudWatch lag, MSK Connect and Glue.
2. Use **Tier 2** when the question needs data only Kafka itself has: consumer group members,
   client IDs and hosts, committed offsets and lag per partition, live broker and topic configs,
   partition sizes on disk, leader balance, or a self-managed Kafka Connect cluster. Also use it
   for topics when the cluster doesn't support the MSK topic APIs.
3. Before the first Tier 2 command, check it can work (see "Check Tier 2" in kafka-admin.md). If
   the tools are missing or the brokers can't be reached, answer with Tier 1, say what Tier 2 would
   add, and point to the setup steps. Don't keep retrying a broker that times out.

## Pick the command for the question

| Question | Tier 1 | Tier 2 |
|---|---|---|
| Which clusters do I have? Details, version, auth, endpoints | `kafka list-clusters-v2`, `describe-cluster-v2`, `get-bootstrap-brokers` | — |
| Is the cluster healthy? | `describe-cluster-v2` state, `list-topics` out-of-sync count, CloudWatch health metrics | `kafka-topics --describe --unavailable-partitions` / `--under-replicated-partitions` / `--under-min-isr-partitions` |
| Brokers and their load | `kafka list-nodes`, CloudWatch per-broker CPU, disk, partitions, leaders, connections | `kafka-broker-api-versions`, `kafka-log-dirs` (size on disk), leader counts from `kafka-topics --describe` |
| Broker configuration | `kafka describe-configuration-revision` (the cluster's MSK configuration) | `kafka-configs --describe --entity-type brokers` |
| Topics, partitions, replication factor | `kafka list-topics` | `kafka-topics --list` / `--describe` |
| Topic configuration (retention, cleanup) | `kafka describe-topic` (Base64 `Configs`) | `kafka-configs --describe --entity-type topics` |
| Partition leaders, ISR, under-replication | `kafka describe-topic-partitions` | `kafka-topics --describe --topic <t>` |
| Topic size and message counts | CloudWatch per-topic bytes and messages in | `kafka-log-dirs`, `kafka-get-offsets` |
| Which consumer groups exist? | CloudWatch `list-metrics` for `SumOffsetLag` | `kafka-consumer-groups --list --state` |
| Which consumer group is lagging? | CloudWatch `SumOffsetLag` / `MaxOffsetLag` / `EstimatedMaxTimeLag` | `kafka-consumer-groups --describe --group <g>` (lag per partition) |
| Who is consuming a topic? Members, hosts, assignments | — | `kafka-consumer-groups --describe --group <g> --members --verbose` |
| Is my connector running? Failed tasks? | MSK Connect: `kafkaconnect list-connectors`, `describe-connector`, CloudWatch `AWS/KafkaConnect`, error logs | Self-managed Connect: REST `GET /connectors?expand=status` |
| Registries, schemas, versions, compatibility, diffs | see glue.md | — |
| Would this new schema break consumers? | `glue get-schema` + latest version, then the rules in glue.md | — |
| Which topics have no schema (and the reverse)? | topic list + `glue list-schemas`, matching rules in glue.md | `kafka-topics --list` |
| Recent cluster operations (upgrades, scaling) | `kafka list-cluster-operations-v2` | — |
| Access denied or credential errors | `aws sts get-caller-identity`, then `iam/fulcrum-lite-readonly-policy.json` in this plugin | same; MSK IAM auth needs the `kafka-cluster:*` read actions in that policy |

## How to answer

- Lead with the direct answer in one or two sentences, then a short markdown table of the relevant
  rows. Say when the data was fetched (the current time), which cluster, registry or Connect
  cluster it came from, and which tier (CloudWatch or live from the brokers).
- Call the worst problems out first: offline partitions, failed connectors or tasks, groups with
  lag but no members, then under-replication and lag.
- Each reference file has an **Insights** section listing what to look for. Use it to go one step
  beyond the raw output: say what the numbers mean and what the user could check or change next.
- **Lag** from CloudWatch is reported about once a minute; lag from `kafka-consumer-groups` is live.
  One sample shows a size, not a trend: to say whether lag is growing, compare two samples taken
  30–60 seconds apart (or several CloudWatch datapoints) and say so.

## Limits to be honest about

- **No producer list.** Kafka has no API that lists producers. Per-topic ingress
  (`BytesInPerSec`) is the best evidence, and only with per-topic monitoring enabled.
- **Tier 2 needs network access** to the brokers. AWS permissions alone aren't enough.
- **Topic APIs need a supported cluster.** If `aws kafka list-topics` fails as unsupported, say so
  and use Tier 2 or the CloudWatch metrics.
- **Schema compatibility checks are advice.** Glue enforces compatibility only when a version is
  registered, which this plugin never does. Say the result is Claude's reading of the rules.
- **No message contents.** Never read records. This plugin has no way to do it.
