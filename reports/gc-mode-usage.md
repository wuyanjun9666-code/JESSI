# GC Mode 使用说明（v1）

本版本移除了 **Minecraft Server 长期运行路径** 上硬编码的 `-XX:+UseSerialGC`，改为可配置的 GC 模式。
Java Tool 路径（建服/安装器等短生命周期任务）保持上游原有 SerialGC 行为，不在本次改动范围。

## 可选值

| 值 | 最终 JVM 参数 |
|---|---|
| `serial`（默认） | `-XX:+UseSerialGC` |
| `auto` | 不添加任何 GC selector，由 JVM 自己选择 |
| `g1` | `-XX:+UseG1GC`，并在未自定义时附加 `-XX:MaxGCPauseMillis=50` |

最终参数中 **永远最多只有一个** GC selector。

## 配置优先级

```
1. <server工作目录>/jessi-gc-mode.txt     ← 单服务器设置（优先）
2. NSUserDefaults  key: jessi.gcMode      ← 全局设置
3. serial                                 ← 默认
```

单服务器文件会覆盖全局设置。

## 如何设置

在服务器工作目录（和 `server.jar` / `jessi-launch-args.txt` 同级）创建：

```
jessi-gc-mode.txt
```

内容一行：

```
g1
```

或 `auto` / `serial`。保存后**完整停止并重新启动服务器**才生效。

读取时会 trim 空白并转小写。只接受 `auto` / `serial` / `g1`。
非法值不会被静默替换，会写日志：

```
Invalid GC mode: xxx
Falling back to Serial
```

## 启动后怎么确认

服务器工作目录生成：

```
jvm-launch.log
```

示例：

```
=== Effective JVM Configuration ===
GC Mode: G1
GC Mode Source: server-file
Heap Initial: 1024M
Heap Max: 5120M
Arguments:
/path/to/bin/java
-Xmx5120M
-Xms1024M
-XX:+UseG1GC
-XX:ReservedCodeCacheSize=32M
...
===================================
```

`GC Mode Source` 取值：

```
server-file
user-defaults
default
```

这里记录的是**真正传给 JLI_Launch 的参数**。

## 脱敏

`jvm-launch.log` 会对包含以下关键词的 `-Dkey=value` 做脱敏：

```
password passwd token secret apikey api_key auth credential
```

输出形如：

```
-Dexample.password=[REDACTED]
```

GC、Heap、Code Cache 等普通参数正常记录。

## 冲突处理

如果 GC Mode = `g1`，但自定义 Launch Arguments 里写了 `-XX:+UseSerialGC`，服务器**拒绝启动**并写：

```
Conflicting garbage collectors:
Configured GC: G1
Custom JVM argument: -XX:+UseSerialGC
```

`auto` 模式允许用户在 extra 里指定一个 GC；出现两个不同 GC 同样拒绝启动。

## 堆参数

结构化内存设置（`jessi.maxHeapMB`、`jessi.initialHeapMB`）优先。
自定义参数里的 `-Xms` / `-Xmx` 会被忽略并写：

```
Ignored duplicate custom heap argument: -Xmx8192M
```

启动前校验：

```
Xms > 0
Xmx > 0
Xms <= Xmx
```

不满足时不会调用 JVM：

```
Invalid heap configuration: Initial heap exceeds maximum heap.
```

## 失败留痕

`JLI_Launch` 返回非 0：

```
JLI_Launch failed
Result: <code>
Requested GC: G1
```

v1 不做自动 fallback：失败后服务器保持停止，把模式改回 `serial` 再启动。

## Code Cache（本次未改）

上游逻辑保持不变：

- iOS ≤ 18：未自定义时 `-XX:ReservedCodeCacheSize=64M`
- iOS 26+ 且设备需要 TXM workaround：未自定义时 `-XX:ReservedCodeCacheSize=32M`
- 其它情况：不显式添加，使用 JVM 默认

因此设备上看到 32M 属于 iOS 26+ TXM 分支，不是本次改动引入的。

## iOS 上的限制

- `spark` 的 profiler 不可用：其 `libasyncProfiler.so` 是 macOS Mach-O，iOS 拒绝加载
- `jfr start` 会造成严重停顿甚至崩溃，不要用
- `jessi-launch-args.txt` 与 `jessi-gc-mode.txt` 都在服务器工作目录，可通过「文件」App 访问
