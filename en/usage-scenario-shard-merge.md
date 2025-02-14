---
title: Data Migration Shard Merge Scenario
summary: Learn how to use Data Migration to migrate data in the shard merge scenario.
aliases: ['/docs/tidb-data-migration/dev/usage-scenario-shard-merge/']
---

# Data Migration Shard Merge Scenario

This document shows how to use Data Migration (DM) in the shard merge scenario.

The example used in this document is a simple scenario where the sharded schemas and sharded tables of three upstream MySQL instances need to be migrated to a downstream TiDB cluster.

For other scenarios, you can refer to [Best Practices of Data Migration in the Shard Merge Scenario](shard-merge-best-practices.md).

## Upstream instances

Assume that the upstream schemas are as follows:

- Instance 1

    | Schema | Tables|
    |:------|:------|
    | user  | information, log_north, log_bak |
    | store_01 | sale_01, sale_02 |
    | store_02 | sale_01, sale_02 |

- Instance 2

    | Schema | Tables|
    |:------|:------|
    | user  | information, log_east, log_bak |
    | store_01 | sale_01, sale_02 |
    | store_02 | sale_01, sale_02 |

- Instance 3

    | Schema | Tables|
    |:------|:------|
    | user  | information, log_south, log_bak |
    | store_01 | sale_01, sale_02 |
    | store_02 | sale_01, sale_02 |

## Migration requirements

1. Merge tables with the same name. For example, merge the `user`.`information` tables of three upstream instances to the downstream `user`.`information` table in TiDB.
2. Merge tables with different names. For example, merge the `user`.`log_{north|south|east}` tables of three upstream instances to the downstream `user`.`log_{north|south|east}` table in TiDB.
3. Merge sharded tables. For example, merge the `store_{01|02}`.`sale_{01|02}` tables of three upstream instances to the downstream `store`.`sale` table in TiDB.
4. Filter delete operations. For example, filter out all the delete operations in the `user`.`log_{north|south|east}` table of three upstream instances.
5. Filter delete operations. For example, filter out all the delete operations in the `user`.`information` table of three upstream instances.
6. Filter delete operations. For example, filter out all the delete operations in the `store_{01|02}`.`sale_{01|02}` table of three upstream instances.
7. Use wildcards to filter specific tables. For example, filter out the `user`.`log_bak` tables of three upstream instances using wildcard `user`.`log_*`.
8. Troubleshoot primary key conflicts. Because the `store_{01|02}`.`sale_{01|02}` tables have auto-increment primary keys of the `bigint` type, the conflict occurs when these tables are merged into TiDB. The following text will show you solutions to resolve and avoid the conflict.

## Downstream instances

Assume that the downstream schema after migration is as follows:

| Schema | Tables |
|:------|:------|
| user | information, log_north, log_east, log_south|
| store | sale |

## Migration solution

- To satisfy the migration Requirements #1 and #2, configure the [table routing rule](key-features.md#table-routing) as follows:

    ```yaml
    routes:
      ...
      user-route-rule:
        schema-pattern: "user"
        target-schema: "user"
    ```

- To satisfy the migration Requirement #3, configure the [table routing rule](key-features.md#table-routing) as follows:

    ```yaml
    routes:
      ...
      store-route-rule:
        schema-pattern: "store_*"
        target-schema: "store"
      sale-route-rule:
        schema-pattern: "store_*"
        table-pattern: "sale_*"
        target-schema: "store"
        target-table:  "sale"
    ```

- To satisfy the migration Requirements #4 and #5, configure the [binlog event filtering rule](key-features.md#binlog-event-filter) as follows:

    ```yaml
    filters:
      ...
      user-filter-rule:
        schema-pattern: "user"
        events: ["truncate table", "drop table", "delete", "drop database"]
        action: Ignore
    ```

    > **Note:**
    >
    > The migration Requirements #4 and #5 indicate that all the deletion operations in the `user` schema are filtered out, so a schema level filtering rule is configured here. And the deletion operations of tables in the `user` schema participating in the future migration will also be filtered out.

- To satisfy the migration Requirement #6, configure the [binlog event filter rule](key-features.md#binlog-event-filter) as follows:

    ```yaml
    filters:
      ...
      sale-filter-rule:
        schema-pattern: "store_*"
        table-pattern: "sale_*"
        events: ["truncate table", "drop table", "delete"]
        action: Ignore
      store-filter-rule:
        schema-pattern: "store_*"
        events: ["drop database"]
        action: Ignore
    ```

- To satisfy the migration Requirement #7, configure the [block and allow table lists](key-features.md#block-and-allow-table-lists) as follows:

    ```yaml
    block-allow-list:
      log-bak-ignored:
        ignore-tales:
        - db-name: "user"
          tbl-name: "log_bak"
    ```

- To satisfy the migration Requirement #8, first refer to [handling conflicts of auto-increment primary key](shard-merge-best-practices.md#handle-conflicts-of-auto-increment-primary-key) to solve conflicts. This guarantees that data is successfully migrated to the downstream when the primary key value of one sharded table is duplicate with that of another sharded table. Then, configure `ignore-checking-items` to skip checking the conflict of auto-increment primary key:

    {{< copyable "" >}}

    ```yaml
    ignore-checking-items: ["auto_increment_ID"]
    ```

## Migration task configuration

The complete configuration of the migration task is shown as below. For more details, see [Data Migration Task Configuration File](task-configuration-file.md).

```yaml
name: "shard_merge"
task-mode: all
meta-schema: "dm_meta"
ignore-checking-items: ["auto_increment_ID"]

target-database:
  host: "192.168.0.1"
  port: 4000
  user: "root"
  password: ""

mysql-instances:
  -
    source-id: "instance-1"
    route-rules: ["user-route-rule", "store-route-rule", "sale-route-rule"]
    filter-rules: ["user-filter-rule", "store-filter-rule" , "sale-filter-rule"]
    block-allow-list:  "log-bak-ignored"  # Use black-white-list if the DM's version <= v2.0.0-beta.2.
    mydumper-config-name: "global"
    loader-config-name: "global"
    syncer-config-name: "global"

  -
    source-id: "instance-2"
    route-rules: ["user-route-rule", "store-route-rule", "sale-route-rule"]
    filter-rules: ["user-filter-rule", "store-filter-rule" , "sale-filter-rule"]
    block-allow-list:  "log-bak-ignored"  # Use black-white-list if the DM's version <= v2.0.0-beta.2.
    mydumper-config-name: "global"
    loader-config-name: "global"
    syncer-config-name: "global"
  -
    source-id: "instance-3"
    route-rules: ["user-route-rule", "store-route-rule", "sale-route-rule"]
    filter-rules: ["user-filter-rule", "store-filter-rule" , "sale-filter-rule"]
    block-allow-list:  "log-bak-ignored"  # Use black-white-list if the DM's version <= v2.0.0-beta.2.
    mydumper-config-name: "global"
    loader-config-name: "global"
    syncer-config-name: "global"

# Other common configs shared by all instances.

routes:
  user-route-rule:
    schema-pattern: "user"
    target-schema: "user"
  store-route-rule:
    schema-pattern: "store_*"
    target-schema: "store"
  sale-route-rule:
    schema-pattern: "store_*"
    table-pattern: "sale_*"
    target-schema: "store"
    target-table:  "sale"

filters:
  user-filter-rule:
    schema-pattern: "user"
    events: ["truncate table", "drop table", "delete", "drop database"]
    action: Ignore
  sale-filter-rule:
    schema-pattern: "store_*"
    table-pattern: "sale_*"
    events: ["truncate table", "drop table", "delete"]
    action: Ignore
  store-filter-rule:
    schema-pattern: "store_*"
    events: ["drop database"]
    action: Ignore

block-allow-list:  # Use black-white-list if the DM's version <= v2.0.0-beta.2.
  log-bak-ignored:
    ignore-tales:
    - db-name: "user"
      tbl-name: "log_bak"

mydumpers:
  global:
    threads: 4
    chunk-filesize: 64

loaders:
  global:
    pool-size: 16
    dir: "./dumped_data"

syncers:
  global:
    worker-count: 16
    batch: 100
    max-retry: 100
```
