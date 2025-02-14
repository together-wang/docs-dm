---
title: Upstream Database Configuration File
summary: Learn the configuration file of the upstream database
aliases: ['/docs/tidb-data-migration/dev/source-configuration-file/']
---

# Upstream Database Configuration File

This document introduces the configuration file of the upstream database, including a configuration file template and the description of each configuration parameter in this file.

## Configuration file template

The following is a configuration file template of the upstream database:

```yaml
source-id: "mysql-replica-01"

# Whether to enable GTID.
enable-gtid: false

# Whether to enable relay log.
enable-relay: false
relay-binlog-name = "" # The file name from which DM-worker starts to pull the binlog.
relay-binlog-gtid = "" # The GTID from which DM-worker starts to pull the binlog.

from:
  host: "127.0.0.1"
  port: 3306
  user: "root"
  password: "ZqMLjZ2j5khNelDEfDoUhkD5aV5fIJOe0fiog9w=" # The user password of the upstream database. It is recommended to use the password encrypted with dmctl.
  security:                       # The TLS configuration of the upstream database
    ssl-ca: "/path/to/ca.pem"
    ssl-cert: "/path/to/cert.pem"
    ssl-key: "/path/to/key.pem"

# purge:
#   interval: 3600
#   expires: 0
#   remain-space: 15

# Configure binlog event filters. New in DM v2.0.2
# case-sensitive: false
# filters:
# - schema-pattern: dmctl
#   table-pattern: t_1
#   events: []
#   sql-pattern:
#   - alter table .* add column `aaa` int
#   action: Ignore
```

> **Note:**
>
> In DM v2.0.1, DO NOT set `enable-gtid` and `enable-relay` to `true` at the same time. Otherwise, it may cause loss of incremental data.

## Configuration parameters

This section describes each configuration parameter in the configuration file.

### Global configuration

| Parameter | Description |
| :------------ | :--------------------------------------- |
| `source-id` | Represents a MySQL instance ID. |
| `enable-gtid` | Determines whether to pull binlog from the upstream using GTID. The default value is `false`. In general, you do not need to configure `enable-gtid` manually. However, if GTID is enabled in the upstream database, and the primary/secondary switch is required, you need to set `enable-gtid` to `true`. |
| `enable-relay` | Determines whether to enable the relay log feature. The default value is `false`. |
| `relay-binlog-name` | Specifies the file name from which DM-worker starts to pull the binlog. For example, `"mysql-bin.000002"`. It only works when `enable_gtid` is `false`. If this parameter is not specified, DM-worker will pull the binlogs starting from the latest one. |
| `relay-binlog-gtid` | Specifies the GTID from which DM-worker starts to pull the binlog. For example, `"e9a1fc22-ec08-11e9-b2ac-0242ac110003:1-7849"`. It only works when `enable_gtid` is `true`. If this parameter is not specified, DM-worker will pull the binlogs starting from the latest GTID. |
| `host` | Specifies the host of the upstream database. |
| `port` | Specifies the port of the upstream database. |
| `user` | Specifies the username of the upstream database. |
| `password` | Specifies the user password of the upstream database. It is recommended to use the password encrypted with dmctl. |
| `security` | Specifies the TLS config of the upstream database. The configured file paths of the certificates must be accessible to all nodes. If the configured file paths are local paths, then all the nodes in the cluster need to store a copy of the certificates in the same path of each host.|

### Relay log cleanup strategy configuration (`purge`)

Generally, there is no need to manually configure these parameters unless there is a large amount of relay logs and disk capacity is insufficient.

| Parameter        | Description                           | Default value |
| :------------ | :--------------------------------------- | :-------------|
| `interval` | Sets the time interval at which relay logs are regularly checked for expiration, in seconds. | `3600`  |
| `expires` | Sets the expiration time for relay logs, in hours. The relay log that is not written by the relay processing unit, or does not need to be read by the existing data migration task will be deleted by DM if it exceeds the expiration time. If this parameter is not specified, the automatic purge is not performed. | `0` |
| `remain-space` | Sets the minimum amount of free disk space, in gigabytes. When the available disk space is smaller than this value, DM-worker tries to delete relay logs. | `15` |

> **Note:**
>
> The automatic data purge strategy only takes effect when `interval` is not 0 and at least one of the two configuration items `expires` and `remain-space` is not 0.

### Binlog event filter

Starting from DM v2.0.2, you can configure binlog event filters in the source configuration file.

| Parameter        | Description                                    |
| :------------ | :--------------------------------------- |
| `case-sensitive` | Determines whether the filtering rules are case-sensitive. The default value is `false`. |
| `filters` | Sets binlog event filtering rules. For details, see [Binlog event filter parameter explanation](key-features.md#parameter-explanation-2). |
