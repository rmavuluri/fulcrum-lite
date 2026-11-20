# Fulcrum Lite user guide

Fulcrum Lite lets you ask Claude about your Amazon MSK (Kafka) clusters, consumer groups,
connectors and AWS Glue Schema Registry in plain English. Claude answers by running read-only
commands in your terminal, and you approve each one.

- [Install](#install)
- [Quick start](#quick-start)
- [How it works: two tiers](#how-it-works-two-tiers)
- [Set up Tier 1: AWS CLI](#set-up-tier-1-aws-cli)
- [Set up Tier 2: Kafka tools (optional)](#set-up-tier-2-kafka-tools-optional)
- [Set up connectors (optional)](#set-up-connectors-optional)
- [What you can ask](#what-you-can-ask)
- [Troubleshooting](#troubleshooting)
- [IAM permissions explained](#iam-permissions-explained)
- [Safety and privacy](#safety-and-privacy)
- [FAQ](#faq)

## Install

In Claude Code, run `/plugin`, search for **Fulcrum Lite** and install it. Or:

```
/plugin marketplace add rmavuluri/fulcrum-lite
/plugin install fulcrum-lite@fulcrum-lite-marketplace
```

Fulcrum Lite runs commands on your machine, so use it in Claude Code (terminal, desktop app or
IDE). On Claude for web or mobile there's no local terminal, so it can't reach your AWS account.

## Quick start

1. Make sure the AWS CLI is signed in to the account with your cluster (`aws sts get-caller-identity`).
2. In Claude Code, run `/fulcrum-lite:getting-started`, or ask *"What can Fulcrum Lite do?"*.
   Claude shows what you can ask and, if you agree, checks your setup: sign-in, region, clusters,
   permissions, CloudWatch lag metrics, Glue, MSK Connect and (optionally) the Kafka tools. You get
   a table of what works and how to fix what doesn't.
3. Ask a question, for example *"Is my Kafka cluster in us-east-1 healthy?"*. Tell Claude your
   region and profile once; it reuses them for the rest of the conversation.

## How it works: two tiers

| | Tier 1: AWS CLI | Tier 2: Kafka tools (optional) |
|---|---|---|
| Talks to | AWS APIs | Your Kafka brokers directly |
| Works from | Anywhere with internet | A machine that can reach the brokers: inside the VPC, over VPN, or with public access |
| You install | AWS CLI v2 | Java + Apache Kafka CLI tools (+ the MSK IAM auth jar for IAM) |
| Consumer groups | Lag from CloudWatch, about once a minute | Live: members, client IDs, hosts, committed offsets and lag per partition |
| Brokers | CPU, disk, partitions, leaders, connections from CloudWatch | Live configs, partition sizes on disk, leader balance |
| Topics | Partitions, replication, configs (MSK topic APIs) | The same, live, on any cluster |
| Connectors | MSK Connect | Self-managed Kafka Connect (REST API) |

Claude uses Tier 1 by default. When a question needs Tier 2 it checks that the tools are installed
and the brokers are reachable; if not, it answers with Tier 1 and tells you what Tier 2 would add.

## Set up Tier 1: AWS CLI

1. Install [AWS CLI v2](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html).
2. Sign in the usual way: a named profile, IAM Identity Center (`aws sso login --profile <name>`),
   or environment variables.
3. Give that identity read-only permissions with
   [`iam/fulcrum-lite-readonly-policy.json`](../iam/fulcrum-lite-readonly-policy.json): create a
   customer managed policy from the file and attach it to your user, group or role (or add it to
   your Identity Center permission set). You can narrow `Resource` to your own clusters,
   registries and connectors.
4. Optional, for richer broker and topic metrics: turn on enhanced monitoring
   (`PER_BROKER` or `PER_TOPIC_PER_BROKER`) on the cluster. This has a CloudWatch cost.

## Set up Tier 2: Kafka tools (optional)

1. Install Java 11 or newer (17 recommended).
2. Get the Kafka CLI tools: download a 3.x release from
   https://kafka.apache.org/downloads and unpack it, or `brew install kafka` on macOS. Downloads
   name the tools `bin/kafka-topics.sh`; Homebrew names them `kafka-topics`.
3. **IAM authentication only:** download `aws-msk-iam-auth-<version>-all.jar` from
   https://github.com/aws/aws-msk-iam-auth/releases into Kafka's `libs/` folder (for Homebrew,
   the `libexec/libs` folder inside the Kafka install).
4. Create a client properties file, for example `~/kafka-client.properties`:

   IAM:
   ```
   security.protocol=SASL_SSL
   sasl.mechanism=AWS_MSK_IAM
   sasl.jaas.config=software.amazon.msk.auth.iam.IAMLoginModule required awsProfileName="<profile>";
   sasl.client.callback.handler.class=software.amazon.msk.auth.iam.IAMClientCallbackHandler
   ```
   (Leave out `awsProfileName="<profile>"` to use your default credentials.)

   Unauthenticated access over TLS:
   ```
   security.protocol=SSL
   ```

   Tier 2 supports IAM and unauthenticated access only. SASL/SCRAM and mutual TLS need a password
   or key stored on your machine, which this plugin doesn't use. On those clusters, Tier 1 still
   answers most questions.
5. Run Claude Code where it can reach the brokers: an EC2 instance (for example over SSM Session
   Manager) or a container in the cluster's VPC, a peered network or VPN, or your laptop if public access is on.
   The broker security group must allow your machine on the port for your auth method (IAM 9098,
   TLS 9094, plaintext 9092; public access: IAM 9198, TLS 9194).
6. Tell Claude where the tools and the properties file are, for example *"Kafka tools are in
   ~/kafka/bin and my client config is ~/kafka-client.properties"*. Claude gets the bootstrap
   servers from AWS.

## Set up connectors (optional)

- **MSK Connect:** nothing extra; Tier 1 permissions cover it. For error details, enable worker log
  delivery to CloudWatch Logs on the connector.
- **Self-managed Kafka Connect:** give Claude the REST URL, for example
  `http://connect.internal:8083`, reachable from where Claude Code runs. Only REST APIs that need
  no login are supported.

## What you can ask

### Clusters and brokers
- "List my MSK clusters" · "Describe the orders cluster"
- "Is my Kafka cluster healthy?" (state, offline and under-replicated partitions, controller,
  CPU, disk)
- "Which broker has the most disk used?" · "Are partition leaders balanced?"
- "How fast is disk growing? When will it fill up?"
- "What's in the cluster's broker configuration?" · "Is auto topic creation on?"
- "What operations ran on the cluster this week?"

### Topics
- "How many partitions does clickstream have?" · "What's the retention on orders?"
- "Is the payments topic under-replicated?" · "Are any partitions offline?"
- "Which topics have non-default settings?" · "Which topics have replication factor 1?"
- "How big is the events topic, and on which brokers?" (Tier 2)

### Consumer groups
- "Which consumer group is lagging?" · "Show lag for billing over the last hour"
- "Is the billing group's lag growing?"
- "Who is consuming payments, and which partitions does each member have?" (Tier 2)
- "Which groups have lag but no active members?" (Tier 2)
- "Why does this group keep rebalancing?" (Tier 2)

### Connectors
- "Is my S3 sink connector running?" · "List my MSK Connect connectors"
- "Why did the Debezium connector fail?" (state, error, recent error logs)
- "Which Kafka Connect tasks are failed?" (self-managed)
- "Is the sink connector keeping up?" (its consumer group lag)

### Glue Schema Registry
- "What schemas are in the payments registry?" · "What's the compatibility mode of OrderCreated?"
- "What changed in OrderCreated between v3 and v4?"
- "Would the schema in ./order-v5.avsc break consumers?"
- "Which topics have no schema? Which schemas have no topic?"
- "Which schemas use compatibility NONE?"

## Sample answers

These are real answers from a test cluster (2 × `kafka.t3.small`, Kafka 3.9, IAM auth), signed
in with only the plugin's read-only policy. They're shortened; names are the test cluster's own.

### "Is my Kafka cluster fulcrum-lite-test healthy? Any under-replicated partitions or broker skew?"

> Yes, **fulcrum-lite-test is healthy.** No partitions are under-replicated, under min ISR or
> offline, and the load is spread almost evenly across the two brokers.
>
> | Check | Value | Status |
> |---|---|---|
> | Cluster state | `ACTIVE` (Kafka 3.9.x, 2 × `kafka.t3.small`) | ✅ |
> | Offline partitions | 0 | ✅ |
> | Active controllers | 1 | ✅ |
> | `UnderReplicatedPartitions` / `UnderMinIsrPartitionCount`, both brokers | 0 throughout | ✅ |
>
> | Metric | Broker 1 | Broker 2 | Skew |
> |---|---|---|---|
> | Partition replicas | 65 | 65 | none |
> | Leaders | 32 | 33 | 1, as close to even as possible |
> | Data disk used % | 4.95 | 4.95 | none |
>
> 65 replicas per broker is far below the roughly 300 AWS recommends as a maximum for
> `kafka.t3.small`. With only 2 brokers and replication factor 2, losing either broker leaves
> every partition under-replicated. That's fine for a test cluster, but production usually has 3.

### "Which consumer group is lagging, and is the lag growing?"

> **`orders-archiver` and `orders-billing` are both lagging badly on `orders`. Their lag grows
> at about the same rate messages are produced, so neither group is consuming at all.**
> `payments-ledger` is mostly fine.
>
> | Group | Topic | Lag 10:15 | Lag 10:45 | Trend |
> |---|---|---|---|---|
> | `orders-archiver` | orders | 929,070 | **1,174,038** | ↑ ~9,000/min while producers are active; never falls |
> | `orders-billing` | orders | 868,596 | **1,101,965** | ↑ ~8,400/min while producers are active; never falls |
> | `payments-ledger` | payments | 98 | 3,081 | Steady near 100, rising in the last minute |
>
> `orders` received about 9,000 messages a minute, matching how fast lag grew. When producers
> went quiet, lag stayed flat instead of falling. CloudWatch can't show group members, so check
> them with `kafka-consumer-groups --describe --group orders-archiver --members --verbose`
> (Tier 2).

### "What changed in the orders-value schema? Is the compatibility mode safe?"

> Three versions exist, and each one only adds an optional field. Nothing has been removed or
> renamed. `BACKWARD` is acceptable today, but it's the weakest useful mode.
>
> | Version | Change |
> |---|---|
> | v1 | First version: `order_id: string`, `amount: double` |
> | v2 | **Added** `currency: string`, default `"USD"` |
> | v3 (latest) | **Added** `customer_id: ["null","string"]`, default `null` |
>
> `BACKWARD` only compares a new version with the one before it, and still allows removing
> fields. Your changes already meet `FULL_ALL`, so you could switch to it and lose nothing. This
> plugin is read-only, so here is the command for you to run yourself:
>
> ```bash
> aws glue update-schema --schema-id RegistryName=fulcrum-lite-test,SchemaName=orders-value --compatibility FULL_ALL
> ```

### "Who is consuming orders?" without the Kafka tools

> I found two consumer groups on `orders`: `orders-archiver` and `orders-billing`. I couldn't
> tell which members hold which partitions, because the Kafka CLI tools aren't on this machine's
> PATH. CloudWatch shows that a group exists, not who is in it. If Kafka is installed, tell me
> the path to its `bin` directory and to your client properties file, and I'll run
> `kafka-consumer-groups.sh --describe --group orders-billing --members --verbose`.

## Troubleshooting

| Symptom | Cause and fix |
|---|---|
| "aws: command not found" | Install AWS CLI v2. |
| Expired token / unable to locate credentials | Sign in again, for example `aws sso login --profile <name>`. |
| `AccessDenied` / not authorized to perform `<action>` | Your identity lacks that action. Attach the plugin's IAM policy, or add the named action. |
| "No clusters" but you have one | Wrong region or account. Tell Claude the region, and check `aws sts get-caller-identity`. |
| `list-topics` not supported | The MSK topic APIs don't support this cluster type or Kafka version. Use Tier 2 for topics; health and lag still work. |
| No lag for a consumer group in CloudWatch | MSK publishes lag only for groups that commit offsets to Kafka, and with some delay. Groups that store offsets elsewhere won't appear. Use Tier 2 to see the group directly. |
| No per-topic throughput | Enable `PER_TOPIC_PER_BROKER` enhanced monitoring on the cluster. |
| Kafka tool hangs, then times out | This machine can't reach the brokers. Check you're inside the VPC or on VPN, the security group allows your IP on the right port, and you're using the bootstrap string for your auth method. |
| `SaslAuthenticationException` / authentication failed (IAM) | The `aws-msk-iam-auth` jar isn't in `libs/`, or your IAM identity lacks `kafka-cluster:Connect`. |
| Cluster uses SASL/SCRAM or mutual TLS | Tier 2 doesn't support these, because they need a password or key from your machine. Tier 1 still works. |
| `TopicAuthorizationException` / `GroupAuthorizationException` | IAM identity lacks `kafka-cluster:DescribeTopic` or `kafka-cluster:DescribeGroup`. |
| Kafka Connect REST: connection refused / timeout | Wrong URL or port, or the worker isn't reachable from this machine. |

Running `/fulcrum-lite:getting-started` checks most of these for you.

## IAM permissions explained

Everything in [`iam/fulcrum-lite-readonly-policy.json`](../iam/fulcrum-lite-readonly-policy.json)
is read-only. You only need the groups for the features you use.

| Group | Actions | Used for |
|---|---|---|
| MSK | `kafka:ListClustersV2`, `DescribeClusterV2`, `ListNodes`, `GetBootstrapBrokers`, `ListClusterOperationsV2` | Clusters, brokers, endpoints, recent operations |
| MSK configuration | `kafka:DescribeConfiguration`, `DescribeConfigurationRevision` | The cluster's broker configuration |
| MSK topics | `kafka:ListTopics`, `DescribeTopic`, `DescribeTopicPartitions` | Topics, partitions, replication (Tier 1) |
| Kafka data plane (IAM auth only) | `kafka-cluster:Connect`, `DescribeCluster`, `DescribeClusterDynamicConfiguration`, `DescribeTopic`, `DescribeTopicDynamicConfiguration`, `DescribeGroup` | The MSK topic APIs and Tier 2 tools on IAM-auth clusters |
| MSK Connect | `kafkaconnect:List*` / `Describe*` for connectors, custom plugins and worker configurations | Connector state, capacity, plugins |
| CloudWatch Logs | `logs:FilterLogEvents` | Recent MSK Connect error log lines |
| Glue | `glue:ListRegistries`, `GetRegistry`, `ListSchemas`, `GetSchema`, `ListSchemaVersions`, `GetSchemaVersion`, `GetSchemaVersionsDiff`, `GetSchemaByDefinition`, `CheckSchemaVersionValidity`, `QuerySchemaVersionMetadata`, `GetTags` | Registries, schemas, versions, diffs, checks |
| CloudWatch | `cloudwatch:ListMetrics`, `GetMetricStatistics`, `GetMetricData` | Health, broker, lag and connector metrics |

## Safety and privacy

- **Read-only.** Claude runs only `list`, `describe` and `get` commands, Kafka tools in `--list` /
  `--describe` mode, and `GET` requests to Kafka Connect. The skill blocks creating, altering or
  deleting anything, resetting offsets, reassigning partitions, and restarting or pausing
  connectors, even if asked.
- **You approve every command.** The plugin pre-approves nothing.
- **No message contents.** It never consumes or produces Kafka records.
- **No secrets.** It never reads `~/.aws`, never prints your Kafka properties file, and masks
  secret-looking values in connector configurations.
- **No backend.** Requests go only to your AWS account and your own brokers and Connect workers.
  See [PRIVACY.md](../PRIVACY.md).

## FAQ

**Does it change anything in my cluster?** No. Every command is read-only, and Claude Code shows
you each one before it runs.

**Do I need the Kafka tools?** No. Tier 1 answers most questions. Install the tools when you want
consumer group members and per-partition lag, or live broker details.

**Does it cost anything?** The plugin is free. Its AWS API calls are small; CloudWatch metric
reads are billed per request beyond the CloudWatch free tier, which normal use rarely exceeds.

**Does it work with MSK Serverless?** Cluster details and CloudWatch metrics work. Some MSK topic
APIs and broker-level details don't apply to Serverless; Claude says when a command isn't
supported.

**Can it read or replay messages?** No, by design.

**Where do I report a problem?** https://github.com/rmavuluri/fulcrum-lite/issues
