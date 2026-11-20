# Tier 2: read-only Kafka admin tools

The Apache Kafka CLI tools talk to the brokers directly, so they can show what the AWS CLI can't:
consumer group members and hosts, committed offsets and lag per partition, live configs and
partition sizes on disk. They need the tools installed **and** network access to the brokers.

## Tool names

Apache Kafka downloads name the tools `kafka-topics.sh`, `kafka-consumer-groups.sh` and so on,
inside `<kafka-dir>/bin`. Homebrew and some packages install them without `.sh` (`kafka-topics`).
Below, `<bin>` means whichever form the user has, for example `~/kafka/bin/kafka-topics.sh` or
`kafka-topics`. On Windows they're `.bat` files under `bin\windows`.

Every command takes `--bootstrap-server <bootstrap>` and, for an authenticated cluster,
`--command-config <client-config>`. Get `<bootstrap>`
from `aws kafka get-bootstrap-brokers` (see aws-commands.md) and ask the user for the path of
their client properties file. **Never print that file**: it can hold a password.

## Check Tier 2

Run these before the first Tier 2 answer in a conversation:

```
command -v kafka-topics.sh kafka-topics
java -version
<bin>/kafka-broker-api-versions.sh --bootstrap-server <bootstrap> --command-config <client-config> | grep -E "^[^ ].*id: "
```

- No tool found: ask the user where Kafka is installed, or give them the setup below.
- The last command lists one line per broker (`host:port (id: N rack: ...)`). If it hangs for more
  than about 30 seconds or reports a timeout or disconnect, this machine can't reach the brokers
  (wrong network, security group, or the wrong port for the auth method). Say so, answer with
  Tier 1, and don't retry in a loop.
- An authentication error with IAM usually means the IAM principal lacks `kafka-cluster:Connect`
  or the `aws-msk-iam-auth` jar isn't on the classpath.

## Setup (for the user to run)

Give the user these steps when Tier 2 isn't available. Don't run them yourself.

1. Install Java 11 or newer (17 recommended).
2. Download Apache Kafka (any 3.x release works against MSK; matching the cluster's version is
   tidiest) from https://kafka.apache.org/downloads and unpack it.
3. For IAM authentication, download the `aws-msk-iam-auth-<version>-all.jar` from
   https://github.com/aws/aws-msk-iam-auth/releases into `<kafka-dir>/libs/`.
4. Create a client properties file, for example `~/kafka-client.properties`. For IAM:

   ```
   security.protocol=SASL_SSL
   sasl.mechanism=AWS_MSK_IAM
   sasl.jaas.config=software.amazon.msk.auth.iam.IAMLoginModule required awsProfileName="<profile>";
   sasl.client.callback.handler.class=software.amazon.msk.auth.iam.IAMClientCallbackHandler
   ```

   Drop `awsProfileName="<profile>"` to use the default credential chain. For TLS only, use
   `security.protocol=SSL`. For SASL/SCRAM, use `security.protocol=SASL_SSL`,
   `sasl.mechanism=SCRAM-SHA-512` and the user's own `sasl.jaas.config` with their username and
   password (the user writes this file; never ask them to paste the password into the chat).
5. Run from a machine that can reach the brokers: an EC2 instance or container in the cluster's
   VPC, a peered network or VPN, or the public bootstrap string if public access is enabled.
6. Attach the `kafka-cluster:*` read actions from `iam/fulcrum-lite-readonly-policy.json` to the
   IAM principal (IAM auth only).

## Brokers

```
<bin>/kafka-broker-api-versions.sh --bootstrap-server <bootstrap> --command-config <client-config> | grep -E "^[^ ].*id: "
<bin>/kafka-configs.sh --bootstrap-server <bootstrap> --command-config <client-config> --describe --entity-type brokers --entity-name <broker-id>
<bin>/kafka-configs.sh --bootstrap-server <bootstrap> --command-config <client-config> --describe --entity-type brokers --entity-default
<bin>/kafka-log-dirs.sh --bootstrap-server <bootstrap> --command-config <client-config> --describe [--broker-list 1,2,3] [--topic-list <t1>,<t2>]
```

- `kafka-configs --describe` for one broker shows the dynamic (non-default) settings; add `--all`
  for every setting with its source.
- `kafka-log-dirs` prints a line of JSON (after a header line). Sum `size` per broker, per topic
  and per partition to show where disk is going; `offsetLag` > 0 is a replica that is behind.
  The output is large on big clusters: narrow it with `--topic-list` or `--broker-list`.
- KRaft clusters (Kafka 3.7+ on MSK in KRaft mode) also allow
  `<bin>/kafka-metadata-quorum.sh --bootstrap-server <bootstrap> --command-config <client-config> describe --status`.
  On a ZooKeeper-mode cluster it fails with `UnsupportedVersionException ... DESCRIBE_QUORUM`.
  That's expected: say the cluster uses ZooKeeper and move on.

## Topics

```
<bin>/kafka-topics.sh --bootstrap-server <bootstrap> --command-config <client-config> --list [--exclude-internal]
<bin>/kafka-topics.sh --bootstrap-server <bootstrap> --command-config <client-config> --describe --topic <topic>
<bin>/kafka-topics.sh --bootstrap-server <bootstrap> --command-config <client-config> --describe --under-replicated-partitions
<bin>/kafka-topics.sh --bootstrap-server <bootstrap> --command-config <client-config> --describe --under-min-isr-partitions
<bin>/kafka-topics.sh --bootstrap-server <bootstrap> --command-config <client-config> --describe --unavailable-partitions
<bin>/kafka-topics.sh --bootstrap-server <bootstrap> --command-config <client-config> --describe --topics-with-overrides
<bin>/kafka-configs.sh --bootstrap-server <bootstrap> --command-config <client-config> --describe --entity-type topics --entity-name <topic>
<bin>/kafka-get-offsets.sh --bootstrap-server <bootstrap> --command-config <client-config> --topic <topic> [--time -1|-2]
```

- `--describe` prints a summary line (partition count, replication factor, configs) and then one
  line per partition with `Leader`, `Replicas` and `Isr`. `Leader: none` or `-1` means offline.
- `--describe` without `--topic` covers every topic: use it only on small clusters, or with one of
  the health filters above, which print only the partitions with a problem (empty output means
  none).
- `kafka-get-offsets` (Kafka 3.0+; it uses `--command-config`) prints `topic:partition:offset`.
  `--time -1` gives the latest offsets, `--time -2` the earliest still kept. Latest minus earliest
  is roughly the number of messages retained per partition.

## Consumer groups

```
<bin>/kafka-consumer-groups.sh --bootstrap-server <bootstrap> --command-config <client-config> --list --state
<bin>/kafka-consumer-groups.sh --bootstrap-server <bootstrap> --command-config <client-config> --describe --group <group>
<bin>/kafka-consumer-groups.sh --bootstrap-server <bootstrap> --command-config <client-config> --describe --group <group> --members --verbose
<bin>/kafka-consumer-groups.sh --bootstrap-server <bootstrap> --command-config <client-config> --describe --group <group> --state
```

- `--list --state` gives every group and its state (`Stable`, `Empty`, `PreparingRebalance`,
  `CompletingRebalance`, `Dead`). `--state` with `--list` needs Kafka 2.6+ tools.
- `--describe --group` gives one row per partition: `CURRENT-OFFSET`, `LOG-END-OFFSET`, `LAG`,
  `CONSUMER-ID`, `HOST`, `CLIENT-ID`. `LAG` of `-` means the group never committed for that
  partition.
- `--members --verbose` gives each member's client ID, host and assigned partitions.
- `--describe --all-groups` exists but is slow and huge on big clusters: list first, then describe
  the groups the user cares about. To find who consumes a topic, describe the groups whose names
  suggest it, or ask the user; there is no "groups by topic" filter.

## Insights

- **Partition health:** any output from `--unavailable-partitions` is an outage for those
  partitions; `--under-min-isr-partitions` means producers using `acks=all` are failing.
- **Leader balance:** count `Leader:` per broker across partitions. A broker leading far more than
  its share does more work; preferred leader election (an admin action for the user) fixes it.
- **Disk skew:** from `kafka-log-dirs`, show the largest topics and the size per broker. Uneven
  totals point to uneven partition placement or one very large topic.
- **Lag per group:** total lag per group and per topic, and the top partitions by lag.
  - Lag on every partition: consumers are too slow for the input rate (scale consumers or
    partitions, or check downstream).
  - Lag on one or two partitions only: a hot key or a stuck consumer on that partition; name the
    `CONSUMER-ID` and `HOST`.
  - Lag with no `CONSUMER-ID`, or group state `Empty`, while the topic still receives data: nobody
    is consuming. Call this out first.
  - State repeatedly `PreparingRebalance` / `CompletingRebalance`: rebalance storms, often from
    `max.poll.interval.ms` timeouts or members restarting.
  - More members than partitions: the extra members sit idle.
- **Is lag growing?** Run `--describe --group` twice 30–60 seconds apart and compare per-partition
  `LAG` and `CURRENT-OFFSET`. Offsets not moving at all means the consumer is stuck, not just slow.
- **Topic overrides:** `--topics-with-overrides` shows topics with non-default settings; point out
  very short or infinite retention, `cleanup.policy=compact` on topics that look like event
  streams, and `min.insync.replicas` higher than the replication factor.
- **MSK's own overrides:** on MSK, every topic (including `__consumer_offsets`) lists
  `min.insync.replicas`, `message.format.version` and `unclean.leader.election.enable`, because
  MSK sets them from the cluster configuration. Don't report these as per-topic overrides the user
  made. If `unclean.leader.election.enable=true` (an MSK default), mention it once at cluster
  level: it can lose data when a broker fails. Report only the other settings as real overrides.

## Never run

- `kafka-topics` with `--create`, `--alter` or `--delete`.
- `kafka-consumer-groups` with `--reset-offsets`, `--delete` or `--delete-offsets`.
- `kafka-configs` with `--alter`, `--add-config` or `--delete-config`.
- `kafka-reassign-partitions`, `kafka-leader-election`, `kafka-preferred-replica-election`,
  `kafka-delete-records`, `kafka-acls` (any mode), `kafka-features`, `kafka-storage`,
  `kafka-transactions`, `kafka-mirror-maker`, `kafka-streams-application-reset`.
- Anything that reads or writes records: `kafka-console-consumer`, `kafka-console-producer`,
  `kafka-verifiable-consumer`, `kafka-verifiable-producer`, `kafka-producer-perf-test`,
  `kafka-consumer-perf-test`, `kafka-dump-log`.
