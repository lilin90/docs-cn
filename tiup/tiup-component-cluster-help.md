---
title: tiup cluster help
summary: tiup-cluster 是一个命令行工具，提供丰富的帮助信息。用户可以通过 `help` 命令或 `--help` 参数获取帮助信息。`tiup cluster help <command>` 等同于 `tiup cluster <command> --help`。语法为 `tiup cluster help [command] [flags]`。使用 `-h` 或 `--help` 输出帮助信息，默认关闭。输出为 `[command]` 或 tiup-cluster 的帮助信息。
---

# tiup cluster help

tiup-cluster 在命令行界面为用户提供了丰富的帮助信息，这些帮助信息可以通过 `help` 命令或者 `--help` 参数获得。`tiup cluster help <command>` 基本等价于 `tiup cluster <command> --help`。

## 语法

```shell
tiup cluster help [command] [flags]
```

`[command]` 用于指定要查看哪个命令的帮助信息，若不指定，则查看 tiup-cluster 自身的帮助信息。

### -h, --help

- 输出帮助信息。
- 数据类型：`BOOLEAN`
- 该选项默认关闭，默认值为 `false`。在命令中添加该选项，并传入 `true` 值或不传值，均可开启此功能。

## 输出

`[command]` 或 tiup-cluster 的帮助信息。
