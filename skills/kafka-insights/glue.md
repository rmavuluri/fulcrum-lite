# AWS Glue Schema Registry

Read-only AWS CLI commands for registries, schemas and versions, plus rules for checking schema
compatibility and matching schemas to topics. Add `--region <region>` (and `--profile <name>`) and
`--output json --no-cli-pager` to each command.

## Registries and schemas

```
aws glue list-registries
aws glue get-registry --registry-id RegistryName=<registry>
aws glue list-schemas --registry-id RegistryName=<registry>
aws glue get-schema --schema-id RegistryName=<registry>,SchemaName=<schema>
aws glue get-tags --resource-arn <registry-or-schema-arn>
```

- Without `--registry-id`, `list-schemas` lists schemas in every registry. The default registry is
  named `default-registry`.
- `get-schema` shows the data format (`AVRO`, `JSON`, `PROTOBUF`), the compatibility mode, the
  status and the latest and next version numbers.

## Versions and diffs

```
aws glue list-schema-versions --schema-id RegistryName=<registry>,SchemaName=<schema>
aws glue get-schema-version --schema-id RegistryName=<registry>,SchemaName=<schema> --schema-version-number LatestVersion=true
aws glue get-schema-version --schema-id RegistryName=<registry>,SchemaName=<schema> --schema-version-number VersionNumber=<n>
aws glue get-schema-version --schema-version-id <version-uuid>
aws glue get-schema-versions-diff --schema-id RegistryName=<registry>,SchemaName=<schema> --first-schema-version-number VersionNumber=<old> --second-schema-version-number VersionNumber=<new> --schema-diff-type SYNTAX_DIFF
aws glue query-schema-version-metadata --schema-id RegistryName=<registry>,SchemaName=<schema> --schema-version-number VersionNumber=<n>
```

- `get-schema-version --schema-version-id` resolves the version UUID that Glue serializers put in
  each record's header, which helps when a consumer error names a version ID.
- Explain a diff in plain words (fields added, removed, renamed or changed type, defaults added or
  removed) as well as showing it.
- `list-schema-versions` shows each version's `Status`; `FAILURE` versions were rejected and
  `PENDING` ones are still being checked.

## Check a proposed schema (read-only)

When the user has a new schema in a local file:

```
aws glue check-schema-version-validity --data-format <AVRO|JSON|PROTOBUF> --schema-definition file://<path>
aws glue get-schema-by-definition --schema-id RegistryName=<registry>,SchemaName=<schema> --schema-definition file://<path>
```

- `check-schema-version-validity` checks only that the definition parses. It doesn't check
  compatibility.
- `get-schema-by-definition` tells whether this exact definition is already registered, and which
  version it is. "Not found" is a normal answer.
- Then check compatibility yourself with the rules below, against the latest version (or every
  version for the `_ALL` modes). Glue makes the final decision only when the version is
  registered, which this plugin never does, so present the result as advice.

## Compatibility rules

Glue modes: `NONE`, `DISABLED`, `BACKWARD`, `BACKWARD_ALL`, `FORWARD`, `FORWARD_ALL`, `FULL`,
`FULL_ALL`. `BACKWARD` means consumers using the new schema can read data written with the old
one (upgrade consumers first). `FORWARD` means consumers still on the old schema can read data
written with the new one (upgrade producers first). `FULL` is both. Plain modes compare with the
latest version; `_ALL` modes compare with every earlier version. `NONE` checks nothing and
`DISABLED` blocks new versions.

**Avro**

| Change | BACKWARD | FORWARD | FULL |
|---|---|---|---|
| Add a field with a default | ok | ok | ok |
| Add a field without a default | breaks | ok | breaks |
| Remove a field that had a default | ok | ok | ok |
| Remove a field without a default | ok | breaks | breaks |
| Rename a field (no alias) | breaks | breaks | breaks |
| Promote type `int`→`long`/`float`/`double`, `long`→`float`/`double`, `float`→`double`, `string`↔`bytes` | ok | breaks | breaks |
| Any other type change | breaks | breaks | breaks |
| Add an enum symbol | ok | breaks unless the enum has a default | breaks unless the enum has a default |
| Remove an enum symbol | breaks unless the enum has a default | ok | breaks unless the enum has a default |
| Change a field from `T` to optional `["null", T]` | ok | breaks | breaks |

**Protobuf:** never reuse or change a field number, or change a field's type in an incompatible
way. Adding fields and removing fields (with their numbers reserved) are generally compatible in
both directions. Renaming a field keeps wire compatibility.

**JSON Schema:** adding an optional property is backward compatible only if the old schema
allowed it (open content model, `additionalProperties` not false); removing or narrowing a
property, adding a `required` entry, or narrowing a type or enum usually breaks. Say JSON Schema
checks are the least certain.

## Topics without schemas (and schemas without topics)

1. Get the topic list (`aws kafka list-topics` or `kafka-topics --list`), excluding internal
   topics (names starting with `__`, and Connect or Streams internals such as `connect-configs`,
   `connect-offsets`, `connect-status` or `*-changelog` / `*-repartition`).
2. Get the schema names (`aws glue list-schemas`, for one registry or all).
3. Match each topic to schemas named exactly like the topic (the Glue serializer's default naming),
   or `<topic>-value` / `<topic>-key`, ignoring case. Ask the user if they use another naming
   strategy.
4. Report three lists: topics with a schema, topics with no schema, and schemas with no topic
   (possibly retired topics or another cluster).

## Insights

- **Risky modes:** schemas in `NONE` (anything goes, consumers can break) or `DISABLED` (no new
  versions possible). Suggest `BACKWARD` as the common default.
- **Churn:** schemas with many versions in a short time, or a diff history with repeated breaking
  changes under `NONE`.
- **Stuck or failed versions:** `PENDING` or `FAILURE` in `list-schema-versions`, or a schema whose
  status isn't `AVAILABLE`.
- **Format mix:** a registry that mixes data formats for one topic family, which consumers often
  don't expect.

## Never run

`create-*`, `update-*`, `delete-*`, `register-schema-version`, `put-schema-version-metadata`,
`remove-schema-version-metadata`, `tag-resource`, `untag-resource` or any command that changes the
registry.
