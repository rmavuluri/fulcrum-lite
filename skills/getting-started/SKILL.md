---
name: getting-started
description: Show what Fulcrum Lite can do and check whether the user's machine and AWS account are set up for it, using only read-only commands. Use when the user asks what Fulcrum Lite (or this Kafka / MSK plugin) can do, how to set it up or get started, what they can ask, or why it isn't working, and on first use.
when_to_use: 'Examples: "what can Fulcrum Lite do?", "how do I set up Fulcrum Lite?", "check my Fulcrum Lite setup", "why can''t you see my Kafka cluster?", "what can I ask about MSK?"'
---

# Getting started with Fulcrum Lite

Do two things: show the user what they can ask, then (if they agree) check their setup and report
what works and what's missing. Everything here is read-only. The question-answering itself is done
by the `kafka-insights` skill; its reference files are in `../kafka-insights/`.

## 1. Show what they can ask

Show this list (keep the wording short; adapt the examples if the user mentioned their own topic,
group or registry names):

| Area | Example questions | Needs |
|---|---|---|
| Clusters and brokers | "List my MSK clusters" · "Is my cluster healthy?" · "Which broker has the most disk used?" · "When will the disk fill up?" · "What's in the broker config?" | AWS CLI |
| Topics | "How many partitions does orders have?" · "Is payments under-replicated?" · "Which topics have non-default settings?" · "How big is clickstream?" | AWS CLI; live configs and sizes need the Kafka tools |
| Consumer groups | "Which consumer group is lagging?" · "Is billing's lag growing?" · "Who is consuming payments, on which partitions?" · "Which groups have no active members?" | Lag: AWS CLI (CloudWatch). Members, hosts, per-partition lag: Kafka tools |
| Connectors | "Is my S3 sink connector running?" · "Why did the Debezium connector fail?" · "Which Connect tasks are failed?" | MSK Connect: AWS CLI. Self-managed Kafka Connect: its REST URL |
| Glue schemas | "What schemas are in the payments registry?" · "What changed in OrderCreated v3 to v4?" · "Would this new schema break consumers?" · "Which topics have no schema?" | AWS CLI |

Then say: everything is read-only, Claude Code asks before each command runs, and nothing reads
Kafka messages. Offer to check their setup now.

## 2. Check the setup (only if the user agrees)

Ask for the region and AWS profile first if they aren't known (`aws configure list` shows the
defaults). Add `--region <region>` and `--profile <name>` if given, plus
`--output json --no-cli-pager`, to every `aws` command. Run the checks in order, and skip the
ones that can't work after an earlier failure (for example, everything after a failed sign-in).

**AWS CLI (Tier 1)**

```
aws --version
aws configure list
aws sts get-caller-identity
aws kafka list-clusters-v2 --max-results 10 --query "ClusterInfoList[].{Name:ClusterName,Type:ClusterType,State:State,Arn:ClusterArn}"
```

If there is more than one cluster, ask which one to check. Then, for that cluster:

```
aws kafka describe-cluster-v2 --cluster-arn <cluster-arn> --query "ClusterInfo.{State:State,Type:ClusterType,Version:Provisioned.CurrentBrokerSoftwareInfo.KafkaVersion,Brokers:Provisioned.NumberOfBrokerNodes,Monitoring:Provisioned.EnhancedMonitoring,Auth:Provisioned.ClientAuthentication}"
aws kafka list-topics --cluster-arn <cluster-arn> --max-results 1
aws cloudwatch list-metrics --namespace AWS/Kafka --metric-name SumOffsetLag --dimensions Name="Cluster Name",Value=<cluster-name> --max-items 5
aws glue list-registries --max-results 5
aws kafkaconnect list-connectors --max-results 5
```

**Kafka tools (Tier 2, optional).** Ask whether they want to use them. If yes:

```
command -v kafka-topics.sh kafka-topics
java -version
```

If the tools are found, ask for the path of their Kafka client properties file (don't print it),
get the bootstrap string for their auth method with
`aws kafka get-bootstrap-brokers --cluster-arn <cluster-arn>`, and run:

```
<bin>/kafka-broker-api-versions.sh --bootstrap-server <bootstrap> --command-config <client-config> | grep -E "^[^ ].*id: "
```

Stop waiting after about 30 seconds: a hang means the brokers aren't reachable from this machine.

**Self-managed Kafka Connect (optional).** Only if the user has one and gives the REST URL:

```
curl -sS --max-time 15 <connect-url>/
```

## 3. Report

One table, one row per check, with ✅ works, ⚠️ works with a limit, or ❌ doesn't work, and a
short "what it means / how to fix" column. Then a one-line summary of which question areas work
right now. Common results:

| Result | Meaning and fix |
|---|---|
| `aws` not found | Install AWS CLI v2: https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html |
| `get-caller-identity` fails (expired token, no credentials) | Sign in first, for example `aws sso login --profile <name>` (the user runs it) |
| `AccessDenied` on any command | The identity is missing that action. Name the action from the error and point to `iam/fulcrum-lite-readonly-policy.json` in the plugin (also at https://github.com/rmavuluri/fulcrum-lite/blob/main/iam/fulcrum-lite-readonly-policy.json) |
| No clusters | Wrong region or account. Ask which region the cluster is in and check there |
| `list-topics` unsupported for the cluster | Topic questions need the Kafka tools for this cluster; health and lag still work through CloudWatch |
| No `SumOffsetLag` metrics | No consumer group has committed offsets recently, or the cluster doesn't report lag yet. Lag questions will need the Kafka tools |
| Enhanced monitoring `DEFAULT` | Per-broker and per-topic throughput metrics are limited. `PER_BROKER` or `PER_TOPIC_PER_BROKER` adds them (a cluster change the user would make, at extra CloudWatch cost) |
| No registries / no connectors | Fine if they don't use Glue or MSK Connect |
| Kafka tools not found, or no Java | Optional. Give the setup steps from "Setup (for the user to run)" in `../kafka-insights/kafka-admin.md` |
| Broker check hangs or times out | This machine can't reach the brokers: run from inside the VPC, over VPN, or use the public bootstrap string if public access is on |
| Broker check fails with an authentication error | IAM: the `kafka-cluster:Connect` permission or the `aws-msk-iam-auth` jar is missing. SASL/SCRAM or mutual TLS: not supported in Tier 2, use Tier 1 |
| Connect REST times out or is refused | The worker isn't reachable from this machine, or the URL or port is wrong |

Finish with two or three example questions that will work for them right now, using their real
cluster, registry or connector names.

## Rules

- Read-only only: the commands above, nothing else. Never install software, sign in, edit files or
  change AWS or Kafka resources. Give the user the command when something is theirs to run.
- Never print secrets: don't read files under `~/.aws`, don't print the Kafka client properties
  file, and never put a password on a command line.
- Never read Kafka messages.
- One plain command per call: no `;` or `&&` chains, loops or shell variables. Run independent
  checks as separate calls in parallel.
