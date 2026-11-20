# Changelog

All notable changes to this plugin are documented here. The format follows
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and the project uses
[Semantic Versioning](https://semver.org/). The version in `.claude-plugin/plugin.json` must
match the newest entry.

## [Unreleased]

## [0.2.0] - 2026-11-20

### Added

- `getting-started` skill: shows what you can ask and runs a read-only setup check (AWS sign-in,
  region, clusters, permissions, CloudWatch lag, Glue, MSK Connect, Kafka tools and broker access),
  then reports what works and how to fix the rest.
- User guide in `docs/USER_GUIDE.md`: setup for each tier, example questions, troubleshooting, and
  every IAM permission explained.
- Tier 2, optional: read-only Apache Kafka admin tools (`kafka-topics`, `kafka-consumer-groups`,
  `kafka-configs`, `kafka-log-dirs`, `kafka-get-offsets`, `kafka-broker-api-versions`) for live
  consumer group members, hosts, offsets and lag per partition, broker and topic configs,
  partition sizes on disk and leader balance. Claude checks the tools and broker access first and
  falls back to the AWS CLI.
- Connectors: MSK Connect (`aws kafkaconnect`, `AWS/KafkaConnect` metrics, error log lines) and
  self-managed Kafka Connect through `GET` requests to its REST API.
- Glue: registry details, tags, version metadata, lookup by version ID or definition, schema
  validity checks, Avro / Protobuf / JSON Schema compatibility rules, and topic-to-schema
  matching.
- MSK broker configuration (`describe-configuration-revision`) and per-broker CloudWatch metrics
  (partitions, leaders, connections, CPU system time).
- "Insights" sections telling Claude what to look for: broker skew, disk growth, lag patterns,
  stuck or empty consumer groups, rebalance storms, failed connectors, risky schema modes.
- Telling a stuck consumer group from a slow one, by comparing how fast lag grows with the
  topic's incoming message rate.
- Sample answers from a real test cluster in the user guide.

### Fixed

- Claude now runs one plain command per call (no loops or `&&` chains), so each command is easy
  to approve and matches allowlists.
- Topic overrides that MSK sets on every topic are no longer reported as user overrides.
  `kafka-metadata-quorum` failing on ZooKeeper clusters is explained as expected.

### Changed

- The skill is split into `aws-commands.md` (was `commands.md`), `kafka-admin.md`, `connect.md`
  and `glue.md`, each with its own "Never run" list.
- The IAM policy adds read-only actions for MSK configurations, consumer groups (IAM auth),
  MSK Connect, CloudWatch Logs filtering and more Glue reads.

## [0.1.0] - 2026-11-15

### Added

- `kafka-insights` skill: read-only AWS CLI commands for MSK clusters, brokers, topics and
  partitions, cluster health, CloudWatch consumer lag and throughput, and Glue Schema Registry
  registries, schemas, versions and diffs.
- Read-only IAM policy in `iam/`.

[Unreleased]: https://github.com/rmavuluri/fulcrum-lite/compare/v0.2.0...HEAD
[0.2.0]: https://github.com/rmavuluri/fulcrum-lite/compare/v0.1.0...v0.2.0
[0.1.0]: https://github.com/rmavuluri/fulcrum-lite/releases/tag/v0.1.0
