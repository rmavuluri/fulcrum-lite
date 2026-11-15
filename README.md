# Fulcrum Lite

Ask Claude about your Amazon MSK (Kafka) clusters and AWS Glue Schema Registry in plain English.
Fulcrum Lite is a skills-only Claude Code plugin: Claude answers by running **read-only AWS CLI
commands** in your terminal, and you approve each one. There is no server and nothing to install
besides the AWS CLI.

> Need consumer group members, live per-partition offsets and lag straight from the brokers? Use
> the full [Fulcrum](https://github.com/rmavuluri/fulcrum) plugin, which talks to Kafka directly.

## What you can ask

- "List my MSK clusters" / "Describe the orders cluster": state, Kafka version, brokers,
  authentication, bootstrap endpoints, recent operations.
- "How many partitions does clickstream have?" / "Is the orders topic under-replicated?": topics,
  partition counts, replication factor, out-of-sync replicas, leaders and ISR, topic configuration.
- "Which consumer group is lagging?": consumer groups and lag from the CloudWatch metrics MSK
  publishes (`SumOffsetLag`, `MaxOffsetLag`, `EstimatedMaxTimeLag`).
- "Is my Kafka cluster healthy?": cluster state, offline and under-replicated partitions, active
  controller, CPU and disk.
- "What schemas are in the payments registry?" / "What changed in OrderCreated between v3 and v4?":
  registries, schemas, compatibility, versions, definitions and diffs.

## Requirements

- [AWS CLI v2](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html),
  signed in to the account that owns the cluster (the CLI's usual configuration: a profile, SSO
  or environment variables). Tell Claude which region and profile to use.
- Read-only IAM permissions: [`iam/fulcrum-lite-readonly-policy.json`](iam/fulcrum-lite-readonly-policy.json).
  Narrow `Resource` to your cluster, topics and registries if you like.
- The MSK topic commands (`list-topics`, `describe-topic`, `describe-topic-partitions`) need a
  cluster type and Kafka version that supports them. Per-topic throughput needs per-topic
  enhanced monitoring.

## Install

```
/plugin marketplace add rmavuluri/fulcrum-lite
/plugin install fulcrum-lite@fulcrum-lite-marketplace
```

Then just ask a question, or invoke the skill with `/fulcrum-lite:kafka-insights`.

## Safety

- Read-only by design: the skill only runs `list`, `describe` and `get` commands for MSK, Glue and
  CloudWatch, plus `aws sts get-caller-identity` and `aws configure list`. It refuses to create,
  change or delete anything.
- Claude Code asks you before each command runs. The plugin pre-approves nothing.
- It never reads Kafka messages, never reads files under `~/.aws`, and never prints credentials.

## Limits

- The AWS CLI doesn't connect to Kafka, so there are no consumer group members, client IDs or
  committed offsets. Lag comes only from CloudWatch, about once a minute.
- Kafka has no API that lists producers; per-topic ingress is the closest evidence.

## License

MIT. See [LICENSE](LICENSE). Privacy: [PRIVACY.md](PRIVACY.md).
