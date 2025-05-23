---
title: 使用 TiDB Data Migration 处理出错的 DDL 语句
summary: 了解在使用 TiDB Data Migration 迁移数据时，如何处理出错的 DDL 语句。
aliases: ['/docs-cn/tidb-data-migration/dev/skip-or-replace-abnormal-sql-statements/']
---

# 使用 TiDB Data Migration 处理出错的 DDL 语句

本文介绍了如何使用 TiDB Data Migration (DM) 来处理出错的 DDL 语句。

目前，TiDB 并不完全兼容所有的 MySQL 语法（详见 [DDL 的限制](/mysql-compatibility.md#ddl-的限制)）。当使用 DM 从 MySQL 迁移数据到 TiDB 时，如果 TiDB 不支持对应的 DDL 语句，可能会造成错误并中断迁移任务。在这种情况下，DM 提供 `binlog` 命令来恢复迁移。

## 使用限制

如果业务不能接受下游 TiDB 跳过异常的 DDL 语句，也不接受使用其他 DDL 语句作为替代，不接受插入其他 DDL 语句，则不适合使用此方式进行处理。

比如：`DROP PRIMARY KEY`，这种情况下，只能在下游重建一个（DDL 执行完后的）新表结构对应的表，并将原表的全部数据重新导入该新表。

## 支持场景

迁移过程中，上游执行了 TiDB 不支持的 DDL 语句并迁移到了 DM，造成迁移任务中断。

- 如果业务能接受下游 TiDB 不执行该 DDL 语句，则使用 `binlog skip <task-name>` 跳过对该 DDL 语句的迁移以恢复迁移任务。
- 如果业务能接受下游 TiDB 执行其他 DDL 语句来作为替代，则使用 `binlog replace <task-name>` 替代该 DDL 的迁移以恢复迁移任务。
- 如果业务能接受下游 TiDB 插入执行其他 DDL 语句，则使用 `binlog inject <task-name>` 插入其他 DDL 以恢复迁移任务。

## 命令介绍

使用 dmctl 手动处理出错的 DDL 语句时，主要使用的命令包括 `query-status`、`binlog`。

### query-status

`query-status` 命令用于查询当前 MySQL 实例内子任务及 relay 单元等的状态和错误信息，详见[查询状态](/dm/dm-query-status.md)。

### binlog

`binlog` 命令管理和查看 binlog 操作。命令仅在 DM v6.0 及其以后版本支持，之前版本可使用 `handle-error` 命令。

`binlog` 命令用法如下：

```bash
binlog -h
```

```
manage or show binlog operations

Usage:
  dmctl binlog [command]

Available Commands:
  inject      inject the current error event or a specific binlog position (binlog-pos) with some ddls
  list        list error handle command at binlog position (binlog-pos) or after binlog position (binlog-pos)
  replace     replace the current error event or a specific binlog position (binlog-pos) with some ddls
  revert      revert the current binlog operation or a specific binlog position (binlog-pos) operation
  skip        skip the current error event or a specific binlog position (binlog-pos) event

Flags:
  -b, --binlog-pos string   position used to match binlog event if matched the binlog operation will be applied. The format like "mysql-bin|000001.000003:3270"
  -h, --help                help for binlog

Global Flags:
  -s, --source strings   MySQL Source ID.

Use "dmctl binlog [command] --help" for more information about a command.
```

`binlog` 支持如下子命令：

+ `inject`：在 DDL binlog 位置插入 DDL 语句，binlog 位置指定方式参考 `-b, --binlog-pos`。
+ `list`：查看 binlog 位置以及此位置之后的有效 inject/skip/replace 操作，binlog 位置指定方式参考 `-b, --binlog-pos`。
+ `replace`：替代 DDL binlog 位置的 DDL 语句，binlog 位置指定方式参考 `-b, --binlog-pos`。
+ `revert`：重置 binlog 位置的 inject/skip/replace 操作，仅在先前的操作没有最终生效前执行，binlog 位置指定方式参考 `-b, --binlog-pos`
+ `skip`：跳过 binlog 位置的 DDL 语句, binlog 位置指定方式参考 `-b, --binlog-pos`。

`binlog` 支持如下参数：

+ `-b, --binlog-pos`：
    - 类型：string。
    - 指定 binlog 位置，表示操作将在 `binlog-pos` 与 binlog event 的 position 匹配时生效。若不指定，DM 会默认置为当前出错的 DDL 语句的 binlog 位置。
    - 格式：`binlog-filename:binlog-pos`，例如 `mysql-bin|000001.000003:3270`。
    - 在迁移执行出错后，binlog position 可直接从 `query-status` 返回的 `startLocation` 中的 `position` 获得；在迁移执行出错前，binlog position 可在上游 MySQL 中使用 [`SHOW BINLOG EVENTS`](https://dev.mysql.com/doc/refman/8.0/en/show-binlog-events.html) 获得。

+ `-s, --source strings`：
    - 类型：string。
    - `source` 指定预设操作将生效的 MySQL 实例。

+ 其他参数参考 `-h` 提示。

## 使用示例

### 迁移中断执行跳过操作

如需在迁移中断时执行跳过操作，可使用 `binlog skip` 命令：

```bash
binlog skip -h
```

```
skip the current error event or a specific binlog position (binlog-pos) event

Usage:
  dmctl binlog skip <task-name> [flags]

Flags:
  -h, --help   help for skip

Global Flags:
  -b, --binlog-pos string   position used to match binlog event if matched the binlog operation will be applied. The format like "mysql-bin|000001.000003:3270"
  -s, --source strings      MySQL Source ID.
```

#### 非合库合表场景

假设现在需要将上游的 `db1.tbl1` 表迁移到下游 TiDB，初始时表结构为：

{{< copyable "sql" >}}

```sql
SHOW CREATE TABLE db1.tbl1;
```

```
+-------+--------------------------------------------------+
| Table | Create Table                                     |
+-------+--------------------------------------------------+
| tbl1  | CREATE TABLE `tbl1` (
  `c1` int NOT NULL,
  `c2` decimal(11,3) DEFAULT NULL,
  PRIMARY KEY (`c1`)
) ENGINE=InnoDB DEFAULT CHARSET=latin1 |
+-------+--------------------------------------------------+
```

此时，上游执行以下 DDL 操作修改表结构（将列的 DECIMAL(11, 3) 修改为 DECIMAL(10, 3)）：

{{< copyable "sql" >}}

```sql
ALTER TABLE db1.tbl1 CHANGE c2 c2 DECIMAL (10, 3);
```

则会由于 TiDB 不支持该 DDL 语句而导致 DM 迁移任务中断，使用 `query-status <task-name>` 命令可看到如下错误：

```
ERROR 8200 (HY000): Unsupported modify column: can't change decimal column precision
```

假设业务上可以接受下游 TiDB 不执行此 DDL 语句（即继续保持原有的表结构），则可以通过使用 `binlog skip <task-name>` 命令跳过该 DDL 语句以恢复迁移任务。操作步骤如下：

1. 使用 `binlog skip <task-name>` 跳过当前错误的 DDL 语句

    {{< copyable "" >}}

    ```bash
    » binlog skip test
    ```

    ```
    {
        "result": true,
        "msg": "",
        "sources": [
            {
                "result": true,
                "msg": "",
                "source": "mysql-replica-01",
                "worker": "worker1"
            }
        ]
    }
    ```

2. 使用 `query-status <task-name>` 查看任务状态

    {{< copyable "" >}}

    ```bash
    » query-status test
    ```

    <details><summary> 执行结果 </summary>

    ```
    {
        "result": true,
        "msg": "",
        "sources": [
            {
                "result": true,
                "msg": "",
                "sourceStatus": {
                    "source": "mysql-replica-01",
                    "worker": "worker1",
                    "result": null,
                    "relayStatus": null
                },
                "subTaskStatus": [
                    {
                        "name": "test",
                        "stage": "Running",
                        "unit": "Sync",
                        "result": null,
                        "unresolvedDDLLockID": "",
                        "sync": {
                            "masterBinlog": "(DESKTOP-T561TSO-bin.000001, 2388)",
                            "masterBinlogGtid": "143bdef3-dd4a-11ea-8b00-00155de45f57:1-10",
                            "syncerBinlog": "(DESKTOP-T561TSO-bin.000001, 2388)",
                            "syncerBinlogGtid": "143bdef3-dd4a-11ea-8b00-00155de45f57:1-4",
                            "blockingDDLs": [
                            ],
                            "unresolvedGroups": [
                            ],
                            "synced": true,
                            "binlogType": "remote",
                            "totalRows": "4",
                            "totalRps": "0",
                            "recentRps": "0"
                        }
                    }
                ]
            }
        ]
    }
    ```

    </details>

    可以看到任务运行正常，错误的 DDL 被跳过。

#### 合库合表场景

假设现在存在如下四个上游表需要合并迁移到下游的同一个表 ``` `shard_db`.`shard_table` ```，任务模式为悲观协调模式：

- MySQL 实例 1 内有 `shard_db_1` 库，包括 `shard_table_1` 和 `shard_table_2` 两个表。
- MySQL 实例 2 内有 `shard_db_2` 库，包括 `shard_table_1` 和 `shard_table_2` 两个表。

初始时表结构为：

{{< copyable "sql" >}}

```sql
SHOW CREATE TABLE shard_db.shard_table;
```

```
+-------+-----------------------------------------------------------------------------------------------------------+
| Table | Create Table                                                                                              |
+-------+-----------------------------------------------------------------------------------------------------------+
| tb    | CREATE TABLE `shard_table` (
  `id` int DEFAULT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=latin1 COLLATE=latin1_bin |
+-------+-----------------------------------------------------------------------------------------------------------+
```

此时，在上游所有分表上都执行以下 DDL 操作修改表字符集

{{< copyable "sql" >}}

```sql
ALTER TABLE `shard_db_*`.`shard_table_*` CHARACTER SET LATIN1 COLLATE LATIN1_DANISH_CI;
```

则会由于 TiDB 不支持该 DDL 语句而导致 DM 迁移任务中断，使用 `query-status` 命令可以看到 MySQL 实例 1 的 `shard_db_1`.`shard_table_1` 表和 MySQL 实例 2 的 `shard_db_2`.`shard_table_1` 表报错：

```
{
    "Message": "cannot track DDL: ALTER TABLE `shard_db_1`.`shard_table_1` CHARACTER SET UTF8 COLLATE UTF8_UNICODE_CI",
    "RawCause": "[ddl:8200]Unsupported modify charset from latin1 to utf8"
}
```

```
{
    "Message": "cannot track DDL: ALTER TABLE `shard_db_2`.`shard_table_1` CHARACTER SET UTF8 COLLATE UTF8_UNICODE_CI",
    "RawCause": "[ddl:8200]Unsupported modify charset from latin1 to utf8"
}
```

假设业务上可以接受下游 TiDB 不执行此 DDL 语句（即继续保持原有的表结构），则可以通过使用 `binlog skip <task-name>` 命令跳过该 DDL 语句以恢复迁移任务。操作步骤如下：

1. 使用 `binlog skip <task-name>` 跳过 MySQL 实例 1 和实例 2 当前错误的 DDL 语句

    {{< copyable "" >}}

    ```bash
    » binlog skip test
    ```

    ```
    {
        "result": true,
        "msg": "",
        "sources": [
            {
                "result": true,
                "msg": "",
                "source": "mysql-replica-01",
                "worker": "worker1"
            },
            {
                "result": true,
                "msg": "",
                "source": "mysql-replica-02",
                "worker": "worker2"
            }
        ]
    }
    ```

2. 使用 `query-status <task-name>` 查看任务状态，可以看到 MySQL 实例 1 的 `shard_db_1`.`shard_table_2` 表和 MySQL 实例 2 的 `shard_db_2`.`shard_table_2` 表报错：

    ```
    {
        "Message": "cannot track DDL: ALTER TABLE `shard_db_1`.`shard_table_2` CHARACTER SET UTF8 COLLATE UTF8_UNICODE_CI",
        "RawCause": "[ddl:8200]Unsupported modify charset from latin1 to utf8"
    }
    ```

    ```
    {
        "Message": "cannot track DDL: ALTER TABLE `shard_db_2`.`shard_table_2` CHARACTER SET UTF8 COLLATE UTF8_UNICODE_CI",
        "RawCause": "[ddl:8200]Unsupported modify charset from latin1 to utf8"
    }
    ```

3. 继续使用 `binlog skip <task-name>` 跳过 MySQL 实例 1 和实例 2 当前错误的 DDL 语句

    {{< copyable "" >}}

    ```bash
    » binlog skip test
    ```

    ```
    {
        "result": true,
        "msg": "",
        "sources": [
            {
                "result": true,
                "msg": "",
                "source": "mysql-replica-01",
                "worker": "worker1"
            },
            {
                "result": true,
                "msg": "",
                "source": "mysql-replica-02",
                "worker": "worker2"
            }
        ]
    }
    ```

4. 使用 `query-status <task-name>` 查看任务状态

    {{< copyable "" >}}

    ```bash
    » query-status test
    ```

    <details><summary> 执行结果 </summary>

    ```
    {
        "result": true,
        "msg": "",
        "sources": [
            {
                "result": true,
                "msg": "",
                "sourceStatus": {
                    "source": "mysql-replica-01",
                    "worker": "worker1",
                    "result": null,
                    "relayStatus": null
                },
                "subTaskStatus": [
                    {
                        "name": "test",
                        "stage": "Running",
                        "unit": "Sync",
                        "result": null,
                        "unresolvedDDLLockID": "",
                        "sync": {
                            "masterBinlog": "(DESKTOP-T561TSO-bin.000001, 2388)",
                            "masterBinlogGtid": "143bdef3-dd4a-11ea-8b00-00155de45f57:1-10",
                            "syncerBinlog": "(DESKTOP-T561TSO-bin.000001, 2388)",
                            "syncerBinlogGtid": "143bdef3-dd4a-11ea-8b00-00155de45f57:1-4",
                            "blockingDDLs": [
                            ],
                            "unresolvedGroups": [
                            ],
                            "synced": true,
                            "binlogType": "remote",
                            "totalRows": "4",
                            "totalRps": "0",
                            "recentRps": "0"
                        }
                    }
                ]
            },
            {
                "result": true,
                "msg": "",
                "sourceStatus": {
                    "source": "mysql-replica-02",
                    "worker": "worker2",
                    "result": null,
                    "relayStatus": null
                },
                "subTaskStatus": [
                    {
                        "name": "test",
                        "stage": "Running",
                        "unit": "Sync",
                        "result": null,
                        "unresolvedDDLLockID": "",
                        "sync": {
                            "masterBinlog": "(DESKTOP-T561TSO-bin.000001, 2388)",
                            "masterBinlogGtid": "143bdef3-dd4a-11ea-8b00-00155de45f57:1-10",
                            "syncerBinlog": "(DESKTOP-T561TSO-bin.000001, 2388)",
                            "syncerBinlogGtid": "143bdef3-dd4a-11ea-8b00-00155de45f57:1-4",
                            "blockingDDLs": [
                            ],
                            "unresolvedGroups": [
                            ],
                            "synced": true,
                            "binlogType": "remote",
                            "totalRows": "4",
                            "totalRps": "0",
                            "recentRps": "0"
                        }
                    }
                ]
            }
        ]
    }
    ```

    </details>

    可以看到任务运行正常，无错误信息。四条 DDL 全部被跳过。

### 迁移中断执行替代操作

如需在迁移中断时执行替代操作，可使用 `binlog replace` 命令：

```bash
binlog replace -h
```

```
replace the current error event or a specific binlog position (binlog-pos) with some ddls

Usage:
  dmctl binlog replace <task-name> <replace-sql1> <replace-sql2>... [flags]

Flags:
  -h, --help   help for replace

Global Flags:
  -b, --binlog-pos string   position used to match binlog event if matched the binlog operation will be applied. The format like "mysql-bin|000001.000003:3270"
  -s, --source strings      MySQL Source ID.
```

#### 非合库合表场景

假设现在需要将上游的 `db1.tbl1` 表迁移到下游 TiDB，初始时表结构为：

{{< copyable "sql" >}}

```sql
SHOW CREATE TABLE db1.tbl1;
```

```
+-------+-----------------------------------------------------------------------------------------------------------+
| Table | Create Table                                                                                              |
+-------+-----------------------------------------------------------------------------------------------------------+
| tb    | CREATE TABLE `tbl1` (
  `id` int DEFAULT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=latin1 COLLATE=latin1_bin |
+-------+-----------------------------------------------------------------------------------------------------------+
```

此时，上游执行以下 DDL 操作增加新列，并添加 UNIQUE 约束

{{< copyable "sql" >}}

```sql
ALTER TABLE `db1`.`tbl1` ADD COLUMN new_col INT UNIQUE;
```

则会由于 TiDB 不支持该 DDL 语句而导致 DM 迁移任务中断，使用 `query-status` 命令可看到如下错误：

```
{
    "Message": "cannot track DDL: ALTER TABLE `db1`.`tbl1` ADD COLUMN `new_col` INT UNIQUE KEY",
    "RawCause": "[ddl:8200]unsupported add column 'new_col' constraint UNIQUE KEY when altering 'db1.tbl1'",
}
```

我们将该 DDL 替换成两条等价的 DDL。操作步骤如下：

1. 使用如下命令替换错误的 DDL 语句

    {{< copyable "" >}}

    ```bash
    » binlog replace test "ALTER TABLE `db1`.`tbl1` ADD COLUMN `new_col` INT;ALTER TABLE `db1`.`tbl1` ADD UNIQUE(`new_col`)";
    ```

    ```
    {
        "result": true,
        "msg": "",
        "sources": [
            {
                "result": true,
                "msg": "",
                "source": "mysql-replica-01",
                "worker": "worker1"
            }
        ]
    }
    ```

2. 使用 `query-status <task-name>` 查看任务状态

    {{< copyable "" >}}

    ```bash
    » query-status test
    ```

    <details><summary> 执行结果 </summary>

    ```
    {
        "result": true,
        "msg": "",
        "sources": [
            {
                "result": true,
                "msg": "",
                "sourceStatus": {
                    "source": "mysql-replica-01",
                    "worker": "worker1",
                    "result": null,
                    "relayStatus": null
                },
                "subTaskStatus": [
                    {
                        "name": "test",
                        "stage": "Running",
                        "unit": "Sync",
                        "result": null,
                        "unresolvedDDLLockID": "",
                        "sync": {
                            "masterBinlog": "(DESKTOP-T561TSO-bin.000001, 2388)",
                            "masterBinlogGtid": "143bdef3-dd4a-11ea-8b00-00155de45f57:1-10",
                            "syncerBinlog": "(DESKTOP-T561TSO-bin.000001, 2388)",
                            "syncerBinlogGtid": "143bdef3-dd4a-11ea-8b00-00155de45f57:1-4",
                            "blockingDDLs": [
                            ],
                            "unresolvedGroups": [
                            ],
                            "synced": true,
                            "binlogType": "remote",
                            "totalRows": "4",
                            "totalRps": "0",
                            "recentRps": "0"
                        }
                    }
                ]
            }
        ]
    }
    ```

    </details>

    可以看到任务运行正常，错误的 DDL 已被替换且执行成功。

#### 合库合表场景

假设现在存在如下四个上游表需要合并迁移到下游的同一个表 ``` `shard_db`.`shard_table` ```，任务模式为悲观协调模式：

- MySQL 实例 1 内有 `shard_db_1` 库，包括 `shard_table_1` 和 `shard_table_2` 两个表。
- MySQL 实例 2 内有 `shard_db_2` 库，包括 `shard_table_1` 和 `shard_table_2` 两个表。

初始时表结构为：

{{< copyable "sql" >}}

```sql
SHOW CREATE TABLE shard_db.shard_table;
```

```
+-------+-----------------------------------------------------------------------------------------------------------+
| Table | Create Table                                                                                              |
+-------+-----------------------------------------------------------------------------------------------------------+
| tb    | CREATE TABLE `shard_table` (
  `id` int DEFAULT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=latin1 COLLATE=latin1_bin |
+-------+-----------------------------------------------------------------------------------------------------------+
```

此时，在上游所有分表上都执行以下 DDL 操作增加新列，并添加 UNIQUE 约束：

{{< copyable "sql" >}}

```sql
ALTER TABLE `shard_db_*`.`shard_table_*` ADD COLUMN new_col INT UNIQUE;
```

则会由于 TiDB 不支持该 DDL 语句而导致 DM 迁移任务中断，使用 `query-status` 命令可以看到 MySQL 实例 1 的 `shard_db_1`.`shard_table_1` 表和 MySQL 实例 2 的 `shard_db_2`.`shard_table_1` 表报错：

```
{
    "Message": "cannot track DDL: ALTER TABLE `shard_db_1`.`shard_table_1` ADD COLUMN `new_col` INT UNIQUE KEY",
    "RawCause": "[ddl:8200]unsupported add column 'new_col' constraint UNIQUE KEY when altering 'shard_db_1.shard_table_1'",
}
```

```
{
    "Message": "cannot track DDL: ALTER TABLE `shard_db_2`.`shard_table_1` ADD COLUMN `new_col` INT UNIQUE KEY",
    "RawCause": "[ddl:8200]unsupported add column 'new_col' constraint UNIQUE KEY when altering 'shard_db_2.shard_table_1'",
}
```

我们将该 DDL 替换成两条等价的 DDL。操作步骤如下：

1. 使用如下命令分别替换 MySQL 实例 1 和实例 2 中错误的 DDL 语句

    {{< copyable "" >}}

    ```bash
    » binlog replace test -s mysql-replica-01 "ALTER TABLE `shard_db_1`.`shard_table_1` ADD COLUMN `new_col` INT;ALTER TABLE `shard_db_1`.`shard_table_1` ADD UNIQUE(`new_col`)";
    ```

    ```
    {
        "result": true,
        "msg": "",
        "sources": [
            {
                "result": true,
                "msg": "",
                "source": "mysql-replica-01",
                "worker": "worker1"
            }
        ]
    }
    ```

    {{< copyable "" >}}

    ```bash
    » binlog replace test -s mysql-replica-02 "ALTER TABLE `shard_db_2`.`shard_table_1` ADD COLUMN `new_col` INT;ALTER TABLE `shard_db_2`.`shard_table_1` ADD UNIQUE(`new_col`)";
    ```

    ```
    {
        "result": true,
        "msg": "",
        "sources": [
            {
                "result": true,
                "msg": "",
                "source": "mysql-replica-02",
                "worker": "worker2"
            }
        ]
    }
    ```

2. 使用 `query-status <task-name>` 查看任务状态，可以看到 MySQL 实例 1 的 `shard_db_1`.`shard_table_2` 表和 MySQL 实例 2 的 `shard_db_2`.`shard_table_2` 表报错：

    ```
    {
        "Message": "detect inconsistent DDL sequence from source ... ddls: [ALTER TABLE `shard_db`.`tb` ADD COLUMN `new_col` INT UNIQUE KEY] source: `shard_db_1`.`shard_table_2`], right DDL sequence should be ..."
    }
    ```

    ```
    {
        "Message": "detect inconsistent DDL sequence from source ... ddls: [ALTER TABLE `shard_db`.`tb` ADD COLUMN `new_col` INT UNIQUE KEY] source: `shard_db_2`.`shard_table_2`], right DDL sequence should be ..."
    }
    ```

3. 使用如下命令继续分别替换 MySQL 实例 1 和实例 2 中错误的 DDL 语句

    {{< copyable "" >}}

    ```bash
    » binlog replace test -s mysql-replica-01 "ALTER TABLE `shard_db_1`.`shard_table_2` ADD COLUMN `new_col` INT;ALTER TABLE `shard_db_1`.`shard_table_2` ADD UNIQUE(`new_col`)";
    ```

    ```
    {
        "result": true,
        "msg": "",
        "sources": [
            {
                "result": true,
                "msg": "",
                "source": "mysql-replica-01",
                "worker": "worker1"
            }
        ]
    }
    ```

    {{< copyable "" >}}

    ```bash
    » binlog replace test -s mysql-replica-02 "ALTER TABLE `shard_db_2`.`shard_table_2` ADD COLUMN `new_col` INT;ALTER TABLE `shard_db_2`.`shard_table_2` ADD UNIQUE(`new_col`)";
    ```

    ```
    {
        "result": true,
        "msg": "",
        "sources": [
            {
                "result": true,
                "msg": "",
                "source": "mysql-replica-02",
                "worker": "worker2"
            }
        ]
    }
    ```

4. 使用 `query-status <task-name>` 查看任务状态

    {{< copyable "" >}}

    ```bash
    » query-status test
    ```

    <details><summary> 执行结果 </summary>

    ```
    {
        "result": true,
        "msg": "",
        "sources": [
            {
                "result": true,
                "msg": "",
                "sourceStatus": {
                    "source": "mysql-replica-01",
                    "worker": "worker1",
                    "result": null,
                    "relayStatus": null
                },
                "subTaskStatus": [
                    {
                        "name": "test",
                        "stage": "Running",
                        "unit": "Sync",
                        "result": null,
                        "unresolvedDDLLockID": "",
                        "sync": {
                            "masterBinlog": "(DESKTOP-T561TSO-bin.000001, 2388)",
                            "masterBinlogGtid": "143bdef3-dd4a-11ea-8b00-00155de45f57:1-10",
                            "syncerBinlog": "(DESKTOP-T561TSO-bin.000001, 2388)",
                            "syncerBinlogGtid": "143bdef3-dd4a-11ea-8b00-00155de45f57:1-4",
                            "blockingDDLs": [
                            ],
                            "unresolvedGroups": [
                            ],
                            "unresolvedGroups": [
                            ],
                            "synced": true,
                            "binlogType": "remote",
                            "totalRows": "4",
                            "totalRps": "0",
                            "recentRps": "0"
                        }
                    }
                ]
            },
            {
                "result": true,
                "msg": "",
                "sourceStatus": {
                    "source": "mysql-replica-02",
                    "worker": "worker2",
                    "result": null,
                    "relayStatus": null
                },
                "subTaskStatus": [
                    {
                        "name": "test",
                        "stage": "Running",
                        "unit": "Sync",
                        "result": null,
                        "unresolvedDDLLockID": "",
                        "sync": {
                            "masterBinlog": "(DESKTOP-T561TSO-bin.000001, 2388)",
                            "masterBinlogGtid": "143bdef3-dd4a-11ea-8b00-00155de45f57:1-10",
                            "syncerBinlog": "(DESKTOP-T561TSO-bin.000001, 2388)",
                            "syncerBinlogGtid": "143bdef3-dd4a-11ea-8b00-00155de45f57:1-4",
                            "blockingDDLs": [
                            ],
                            "unresolvedGroups": [
                            ],
                            "unresolvedGroups": [
                            ],
                            "synced": try,
                            "binlogType": "remote",
                            "totalRows": "4",
                            "totalRps": "0",
                            "recentRps": "0"
                        }
                    }
                ]
            }
        ]
    }
    ```

    </details>

    可以看到任务运行正常，无错误信息。四条 DDL 全部被替换。

### 其他命令

binlog 其他命令的使用，请参考上述 `binlog skip`、`binlog replace` 命令的使用方式。
