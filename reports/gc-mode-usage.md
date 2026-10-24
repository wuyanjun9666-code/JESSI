# GC Mode 使用说明（v1）

本版本移除了 JESSI 硬编码的 `-XX:+UseSerialGC`，改为可配置的 GC 模式。

## 可选值

| 值 | 最终 JVM 参数 |
|---|---|
| `auto`（默认） | 不添加任何 GC selector，由 JVM 自己选择 |
| `serial` | `-XX:+UseSerialGC` |
| `g1` | `-XX:+UseG1GC`，并在未自定义时附加 `-XX:MaxGCPauseMillis=50` |

最终参数中 **永远最多只有一个** GC selector。

## 如何设置

### 方式 A：服务器目录下的文件（推荐，iPad 可直接改）

在服务器工作目录（和 `server.jar` / `jessi-launch-args.txt` 同级）创建：

```
jessi-gc-mode.txt
```

内容只写一行：

```
g1
```

或 `auto` / `serial`。

保存后**完整停止并重新启动服务器**才会生效。

### 方式 B：NSUserDefaults（后续接 UI 用）

key：`jessi.gcMode`，值同上。优先级高于文件。

## 启动后怎么确认

服务器工作目录会生成：

```
jvm-launch.log
```

内容示例：

```
=== Effective JVM Configuration ===
GC Mode: G1
Heap Initial: 1024M
Heap Max: 5120M
Arguments:
/path/to/bin/java
-Xmx5120M
-Xms1024M
-XX:+UseG1GC
...
===================================
```

这里记录的是**真正传给 JLI_Launch 的参数**，不是 UI 里的输入值。

## 冲突处理

如果 GC Mode = `g1`，但自定义 Launch Arguments 里写了 `-XX:+UseSerialGC`，服务器会**拒绝启动**并写入：

```
Conflicting garbage collectors:
Configured GC: G1
Custom JVM argument: -XX:+UseSerialGC
```

不会再把 JVM 自己崩掉的错误丢给用户。

如果 GC Mode = `auto`，允许用户在自定义参数里指定一个 GC；指定两个不同 GC 同样会拒绝启动。

## 堆参数

结构化内存设置（JESSI 的 RAM slider / `jessi.maxHeapMB`、`jessi.initialHeapMB`）优先。

自定义参数里的 `-Xms` / `-Xmx` 会被忽略，并写入：

```
Ignored duplicate custom heap argument: -Xmx8192M
```

最终 `-Xms` / `-Xmx` 各只出现一次。

## 失败留痕

`JLI_Launch` 返回非 0 时，日志会记录：

```
JLI_Launch failed
Result: <code>
Requested GC: G1
```

v1 不做自动 fallback：启动失败后服务器保持停止，用户把模式改回 `auto` / `serial` 再启动。

## iOS 上的限制（重要）

- `spark` 的 profiler 不可用：其 `libasyncProfiler.so` 是 macOS Mach-O，iOS 拒绝加载
- `jfr start` 会造成严重停顿甚至崩溃，**不要用**
- `jessi-launch-args.txt` 与 `jessi-gc-mode.txt` 都在服务器工作目录，可通过「文件」App 访问
