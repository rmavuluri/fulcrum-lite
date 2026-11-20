# Fulcrum Lite

Ask Claude about your Amazon MSK (Kafka) clusters, consumer groups, connectors and AWS Glue Schema
Registry in plain English. Fulcrum Lite is a skills-only Claude Code plugin: Claude answers by
running **read-only commands** in your terminal, and you approve each one. There is no server and
nothing to install besides the tools below.

> Want a packaged MCP server instead of CLI commands? See the full
> [Fulcrum](https://github.com/rmavuluri/fulcrum) plugin.

## Two tiers

| | Tier 1: AWS CLI | Tier 2: Kafka admin tools (optional) |
|---|---|---|
| Needs | AWS CLI v2, signed in | Apache Kafka CLI tools + network access to the brokers |
| Works from | Anywhere | Inside the VPC, over VPN, or with public access |
| Consumer groups | Lag from CloudWatch (about once a minute) | Live: members, hosts, client IDs, offsets and lag per partition |
| Extra | MSK Connect, Glue, CloudWatch trends | Live configs, partition sizes on disk, self-managed Kafka Connect |

Claude uses Tier 1 by default and Tier 2 when the question needs it and the tools are available.
If Tier 2 isn't set up, Claude answers with Tier 1 and says what Tier 2 would add.

## What you can ask

- **Clusters and brokers:** "List my MSK clusters", "Is my Kafka cluster healthy?", "Which broker
  has the most disk used?", "Are leaders balanced?", "What's in the cluster's broker config?",
  "When will the disk fill up?"
- **Topics:** "How many partitions does clickstream have?", "Is the orders topic under-replicated?",
  "Which topics have non-default settings?", "How big is the payments topic?"
- **Consumer groups:** "Which consumer group is lagging?", "Who is consuming payments and on which
  partitions?", "Is the billing group's lag growing?", "Which groups have no active members?"
- **Connectors:** "Is my S3 sink connector running?", "Why did the Debezium connector fail?",
  "Which Kafka Connect tasks are failed?" (MSK Connect and self-managed Kafka Connect)
- **Glue Schema Registry:** "What schemas are in the payments registry?", "What changed in
  OrderCreated between v3 and v4?", "Would this new schema break consumers?", "Which topics have no
  schema?", "Which schemas have compatibility NONE?"

## Requirements

- [AWS CLI v2](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html),
  signed in to the account that owns the cluster (a profile, SSO or environment variables). Tell
  Claude which region and profile to use.
- Read-only IAM permissions: [`iam/fulcrum-lite-readonly-policy.json`](iam/fulcrum-lite-readonly-policy.json).
  Narrow `Resource` to your clusters, topics, groups, connectors and registries if you like. The
  `kafka-cluster:*` actions are only needed for Tier 2 with IAM authentication.
- The MSK topic APIs (`list-topics`, `describe-topic`, `describe-topic-partitions`) need a
  cluster type and Kafka version that supports them. Per-topic throughput needs per-topic
  enhanced monitoring.

### Tier 2 setup (optional)

1. Java 11+ and an [Apache Kafka](https://kafka.apache.org/downloads) 3.x download (or
   `brew install kafka`).
2. For IAM auth, the `aws-msk-iam-auth-<version>-all.jar` from
   [aws-msk-iam-auth releases](https://github.com/aws/aws-msk-iam-auth/releases) in Kafka's
   `libs/` folder.
3. A client properties file, for IAM:

   ```
   security.protocol=SASL_SSL
   sasl.mechanism=AWS_MSK_IAM
   sasl.jaas.config=software.amazon.msk.auth.iam.IAMLoginModule required;
   sasl.client.callback.handler.class=software.amazon.msk.auth.iam.IAMClientCallbackHandler
   ```

4. Run Claude Code on a machine that can reach the brokers, and tell Claude where the Kafka tools
   and the properties file are. For self-managed Kafka Connect, also give it the REST URL.

## Install

From the Claude plugin directory (search for **Fulcrum Lite**), or:

```
/plugin marketplace add rmavuluri/fulcrum-lite
/plugin install fulcrum-lite@fulcrum-lite-marketplace
```

Then run `/fulcrum-lite:getting-started` (or ask *"What can Fulcrum Lite do?"*): Claude shows
what you can ask and, if you agree, checks your setup and tells you what works and how to fix what
doesn't. After that, just ask questions, or invoke the skill with `/fulcrum-lite:kafka-insights`.

The full **[user guide](docs/USER_GUIDE.md)** covers setup for each tier, every kind of question,
troubleshooting and each IAM permission.

## Safety

- Read-only by design. The skill runs only `list`, `describe` and `get` commands, Kafka admin
  tools in `--list` / `--describe` mode, and `GET` requests to Kafka Connect. It refuses to create,
  alter, delete, reset offsets, reassign partitions, restart connectors or change anything.
- Claude Code asks you before each command runs. The plugin pre-approves nothing.
- It never reads Kafka messages, never reads files under `~/.aws`, never prints your Kafka client
  properties file, and masks secret-looking values in connector configurations.

## Limits

- Kafka has no API that lists producers; per-topic ingress is the closest evidence.
- Schema compatibility checks are Claude's reading of the rules. Glue decides only when a version is
  registered, which this plugin never does.
- Tier 2 needs network access to the brokers; AWS permissions alone aren't enough.

## License

MIT. See [LICENSE](LICENSE). Privacy: [PRIVACY.md](PRIVACY.md).
