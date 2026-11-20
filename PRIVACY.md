# Fulcrum Lite privacy policy

Last updated: 2026-11-19

Fulcrum Lite is an open-source Claude Code plugin made only of instructions (a skill). It contains
no program, runs no server, and has no service, account or backend. The author receives no data
from you.

## What it reads

When you ask a question, Claude runs read-only commands that you approve: AWS CLI commands against
the AWS account your AWS CLI is signed in to and, if you have set them up, Apache Kafka admin tools
against your own Kafka brokers and `GET` requests to your own Kafka Connect REST API. It can read:

- MSK cluster details: name, state, Kafka version, brokers and endpoints, broker configuration,
  recent operations.
- Topic metadata: names, partitions, replicas, in-sync replicas, offsets, sizes on disk and topic
  configuration.
- Consumer group metadata: group names and states, committed offsets and lag, and each member's
  client ID, consumer ID and host address.
- Connector status: MSK Connect and Kafka Connect connectors, tasks, capacity, error messages and
  recent MSK Connect error log lines. Secret-looking configuration values are masked.
- CloudWatch metrics for the cluster and connectors: lag, partition health, CPU, memory, disk and
  throughput.
- Glue Schema Registry registries, schema names, versions, definitions, metadata, tags and
  compatibility, and schema files you point it to on your machine.

It never reads Kafka message contents, and it does not collect names, email addresses or other
personal data beyond the client IDs and host addresses Kafka reports for consumers.

## Credentials

The AWS CLI and the Kafka tools use their own configuration on your machine. Fulcrum Lite never
asks for, stores or prints secret keys or passwords. The skill tells Claude not to read files under
`~/.aws`, not to print your Kafka client properties file, and never to put a password on a command
line.

## Where data goes

Requests go only to AWS in your own account and to your own Kafka brokers and Kafka Connect
workers. Command output is returned to your Claude Code session, where it is handled under your
agreement with Anthropic. Fulcrum Lite sends no analytics or telemetry, contacts no other service,
and stores nothing.

## Contact

Questions or issues: https://github.com/rmavuluri/fulcrum-lite/issues
