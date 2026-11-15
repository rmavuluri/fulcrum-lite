# Fulcrum Lite privacy policy

Last updated: 2026-11-15

Fulcrum Lite is an open-source Claude Code plugin made only of instructions (a skill). It contains
no program, runs no server, and has no service, account or backend. The author receives no data
from you.

## What it reads

When you ask a question, Claude runs read-only AWS CLI commands that you approve, against the AWS
account your AWS CLI is signed in to:

- MSK cluster details: name, state, Kafka version, brokers and endpoints, recent operations.
- Topic metadata: names, partitions, replicas, in-sync replicas and topic configuration.
- CloudWatch metrics for the cluster: consumer lag, partition health, CPU, disk and throughput.
- Glue Schema Registry registries, schema names, versions, definitions and compatibility.

It never reads Kafka message contents, and it does not collect names, email addresses or other
personal data.

## Credentials

The AWS CLI uses its own configuration on your machine. Fulcrum Lite never asks for, stores or
prints secret keys, and the skill tells Claude not to read files under `~/.aws`.

## Where data goes

Requests go only to AWS in your own account. Command output is returned to your Claude Code
session, where it is handled under your agreement with Anthropic. Fulcrum Lite sends no analytics
or telemetry, contacts no other service, and stores nothing.

## Contact

Questions or issues: https://github.com/rmavuluri/fulcrum-lite/issues
