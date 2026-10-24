# JVM Launch Audit — JessiJavaRunner.m

审计对象：`JESSI/JessiCore/JessiJavaRunner.m`（上游 `Baconium/JESSI`，branch `jvm-gc-config`）

## 启动入口

代码里有 **两条服务端启动路径**，分别对应 macOS 与 iOS：

| 路径 | 入口函数 | 构造方式 | 最终调用 |
|---|---|---|---|
| Server (macOS / Catalyst) | `jessi_spawn_server`（约 1600 行） | `NSMutableArray *args` | `jessi_spawn_external_java_args(args)` |
| Server (iOS) | `jessi_spawn_server`（约 1770 行） | `const char *jargv[80]` | `JLI_Launch`（dlsym 自 libjli） |
| Tool (macOS) | `jessi_spawn_tool`（约 1990 行） | `NSMutableArray *args` | `jessi_spawn_external_java_args(args)` |
| Tool (iOS) | `jessi_spawn_tool`（约 2100 行） | `const char *jargv[96]` | `JLI_Launch` |

注意：不是 `JNI_CreateJavaVM`，而是 `JLI_Launch`（OpenJDK launcher），因此“JNI 返回码”对应 `launchCtx.result`。

## GC 参数添加位置（修改前）

| 位置 | 代码 |
|---|---|
| Server macOS | `[args addObject:@"-XX:+UseSerialGC"];` |
| Server iOS | `jargv[idx++] = "-XX:+UseSerialGC";` |
| Tool macOS | `[args addObject:@"-XX:+UseSerialGC"];` |
| Tool iOS | `jargv[idx++] = "-XX:+UseSerialGC";` |
| Server macOS | `[args addObject:@"-XX:MaxGCPauseMillis=50"];`（无条件） |
| Server iOS | `jargv[idx++] = "-XX:MaxGCPauseMillis=50";`（无条件） |

结论：Server 的两条路径各自硬编码一次，必须同时修改，否则 iOS 路径仍会强制 SerialGC。

## Xms / Xmx 添加位置

- 值来源：`NSUserDefaults` → `jessi.maxHeapMB`、`jessi.initialHeapMB`
- Server macOS：`xmx` / `xms` 加入 `args`
- Server iOS：`xmx` / `xms` 加入 `jargv`
- Tool 路径固定 `-Xmx512M -Xms16M`（本次未改）

## extra args 合并位置

1. `<workingDir>/jessi-launch-args.txt`
2. `[JessiSettings shared].launchArguments`
3. `jessi_expand_argfile_entries(...)`
4. Server iOS 额外经过 `jessi_filter_extra_jvm_args(...)`

合并结果最后追加到 `args` / `jargv`。

## 最终 argv 生成位置

- Server macOS：`jessi_spawn_external_java_args(args)` 之前
- Server iOS：`jargv[idx++] = NULL;` → `jargc = idx - 1` → `jessi_dup_argv(...)` → `JLI_Launch`

## 本次修改点

1. 新增 `jessi_gc_mode_for_workdir()`：读取 `jessi.gcMode`（NSUserDefaults）或 `<workdir>/jessi-gc-mode.txt`，取值为 `auto|serial|g1`，默认 `auto`。
2. 新增 `jessi_prepare_jvm_extra_args()`：
   - 剥离 extra 中的 `-Xms*` / `-Xmx*`（结构化内存设置优先），写日志 `Ignored duplicate custom heap argument`
   - 收集 extra 中的 GC selector，最多保留一个
   - 结构化 GC 与 extra GC 冲突时返回错误，不启动 JVM
3. Server 两条路径都改为：先准备 extra/GC，再写入唯一 GC 参数
4. `MaxGCPauseMillis=50` 仅在 GC Mode = G1 时添加，且 extra 未自带
5. Tool 两条路径移除硬编码 `UseSerialGC`（不指定 GC，交给 JVM 默认）
6. 启动前写入 `jvm-launch.log`（工作目录）与 stderr：Effective JVM Configuration
7. `JLI_Launch` 返回非 0 时记录 result、Requested GC、时间戳
8. 不做自动 fallback，不做进程内二次创建 JVM
