---
title: 向量搜索限制
summary: 了解 TiDB 向量搜索功能的限制。
---

# 向量搜索限制

本文档介绍 TiDB 向量搜索的已知限制。

> **警告：**
>
> 向量搜索目前为实验特性，不建议在生产环境中使用。该功能可能会在未事先通知的情况下发生变化。如果发现 bug，请在 GitHub 上提 [issue](https://github.com/pingcap/tidb/issues) 反馈。

## 向量数据类型限制

- 向量最大支持 16383 维。
- 向量数据中不支持 `NaN`、`Infinity` 和 `-Infinity` 浮点数。
- 向量列不能作为主键或者主键的一部分。
- 向量列不能作为唯一索引或者唯一索引的一部分。
- 向量列不能作为分区键或者分区键的一部分。
- 向量数据类型不支持存储双精度浮点数。当向 TiDB 中的向量列插入或存储双精度浮点数时，TiDB 会将这些双精度浮点数自动转换为单精度浮点数。
- 目前 TiDB 不支持将向量类型的列修改为其他数据类型（如 `JSON`、`VARCHAR` 等）。

## 向量搜索索引限制

参考[向量搜索索引 - 使用限制](/vector-search/vector-search-index.md#使用限制)。

## 工具兼容性

- 确保使用 BR v8.4.0 及以上版本进行备份与恢复。不支持将带有向量数据类型的表恢复至 v8.4.0 之前的 TiDB 集群。
- TiDB Data Migration (DM) 不支持迁移或同步 MySQL 9.0 的向量数据类型到 TiDB。
- TiCDC 在同步向量数据到不支持向量数据类型的下游时会修改数据类型。详情参考[向量数据类型兼容性说明](/ticdc/ticdc-compatibility.md#向量数据类型兼容性说明)。

## 反馈

我们非常重视您的反馈意见。如果在开发的过程中遇到问题，可以在 [AskTUG](https://asktug.com/?utm_source=docs-cn-dev-guide) 上进行提问，寻求帮助。
