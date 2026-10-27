# 事件输出后端知识

本文记录完整版与 mini 版两套输出后端的差异、缓存机制、文件写入、UART 输出和刷盘逻辑。事件创建和编码见 `event-pipeline.md`。

## 两套输出后端

通过 `HIEVENT_LITE_MINI` 宏在编译期选择输出后端：

| 维度 | 完整版（`hiview_output_event.c`） | mini 版（`hiview_output_event_mini.c`） |
| --- | --- | --- |
| 文件系统 | 支持（cache + file + UART） | 不支持（仅 UART/printf） |
| 事件缓存 | 三类独立缓存（fault/ue/stat） | 无缓存 |
| 文件写入 | 支持，含重试初始化 | 空实现 |
| 文件监听 | 支持（`RegisterFileWatcher`） | 空实现 |
| 文件处理 | 支持（`ProcFile` copy/rename） | 空实现 |
| 互斥锁 | 支持（`HIVIEW_MutexInit`/`Lock`/`Unlock`） | 无 |
| 刷盘 | `FlushEvent` 同步/异步 | 无 |
| 自定义处理器 | 支持 | 支持 |
| 初始化 | `InitCoreEventOutput` + `InitEventOutput` | 空函数 |

**关键约束：** 两套实现必须保持 `hiview_output_event.h` 中声明的接口签名完全一致。mini 版的空实现函数不得添加实际逻辑。

## 缓存机制

完整版为三类事件维护独立的全局缓存和文件：

```
g_faultEventCache -> g_faultEventFile (HIVIEW_FAULT_EVENT_FILE)
g_ueEventCache    -> g_ueEventFile    (HIVIEW_UE_EVENT_FILE)
g_statEventCache  -> g_statEventFile  (HIVIEW_STAT_EVENT_FILE)
```

- 每类缓存大小由 `EVENT_CACHE_SIZE`（默认 256）控制。
- 文件大小由 `FAULT_EVENT_FILE_SIZE`/`UE_EVENT_FILE_SIZE`/`STAT_EVENT_FILE_SIZE`（默认均 1024）控制。
- `GetEventCache(type, &c, &f)` 按位掩码路由到对应缓存和文件。

### 缓存写入流程（`OutputEvent`）

1. 若自定义处理器 `g_hieventOutputProc` 返回 `TRUE`，跳过默认处理。
2. 大端转换 `eventId` 和 `time`（仅 `LITTLE_ENDIAN_SYSTEM`）。
3. 若缓存剩余空间不足，先触发输出再写入（`reachMaxThreshold` 路径）。
4. 写入 `HiEventCommon`（8 字节）+ payload 到缓存。
5. 若 `usedSize >= HIVIEW_HIEVENT_FILE_BUF_SIZE`（默认 128）或 `OUTPUT_OPTION_DEBUG`，触发输出消息。
6. 根据 `outputOption` 发送 `HIVIEW_MSG_OUTPUT_EVENT_BIN_FILE` 或 `HIVIEW_MSG_OUTPUT_EVENT_FLOW`。

**关键约束：** 缓存满时先输出再写入；输出后仍空间不足则丢弃事件并打印警告。不要在此路径增加阻塞操作。

## 输出选项

`g_hiviewConfig.outputOption` 控制输出目标：

| 选项 | 行为 | 事件支持 |
| --- | --- | --- |
| `OUTPUT_OPTION_BIN_FILE` | 写入二进制事件文件 | 支持 |
| `OUTPUT_OPTION_TEXT_FILE` | 写入文本文件 | 不支持，退化为二进制 |
| `OUTPUT_OPTION_FLOW` | UART 实时输出明文 | 支持 |
| `OUTPUT_OPTION_DEBUG` | UART 实时输出（每次触发） | 支持 |

事件不支持文本格式，`OUTPUT_OPTION_TEXT_FILE` 和 `OUTPUT_OPTION_BIN_FILE` 走相同路径。

## 文件写入（`Output2Flash`）

1. 加锁 `g_eventFlushInfo.mutex`。
2. 从缓存读取所有完整事件到临时缓冲区。
3. 若文件句柄无效（`fhandle < 0`），调用 `ReInitHiEventFile` 重试初始化。
4. 批量写入临时缓冲区到文件。
5. 若 `eventType & SYNC_FILE`，调用 `HIVIEW_FileSync` 同步刷盘。
6. 释放临时缓冲区并解锁。

### 重试初始化

每类事件维护独立重试计数器（`g_retryFaultInitCount` 等），上限 `MAX_RETRY_COUNT`（100）：

- `InitHiviewFile` 失败时计数器递增。
- 成功时计数器清零。
- 超过上限后不再重试，该类事件文件写入静默失败。

**关键约束：** 不要移除重试上限，否则文件系统未就绪时会无限重试。

## UART 实时输出（`OutputEventRealtime`）

1. 加锁 `g_eventFlushInfo.mutex`。
2. 从缓存逐条读取事件。
3. 调用 `EventContentFmt` 格式化为明文字符串。
4. 通过 `HIVIEW_UartPrint` 输出。
5. 解锁。

`EventContentFmt` 输出格式：`EVENT: time=%02u:%02u:%02u id=%u type=%u data=%p\n`

## 刷盘（`FlushEvent`）

系统重启前调用 `HiEventFlush(syncFlag)`：

- `syncFlag == FALSE`：异步发送消息（`HiviewSendMessage`）。
- `syncFlag == TRUE`：直接调用 `OutputEvent2Flash` 或 `OutputEventRealtime`。

分别对 fault、ue、stat 三类事件执行刷盘。

## 自定义处理器

`HiviewRegisterHieventProc(func)` 注册全局回调 `g_hieventOutputProc`：

- 在 `OutputEvent` 入口处调用，返回 `TRUE` 时跳过平台默认处理。
- 全局仅支持一个回调，注册新的会覆盖旧的。
- `HiviewUnRegisterHieventProc` 置 NULL（忽略参数 func）。

**关键约束：** 回调中不要执行阻塞操作或长时间处理，否则影响后续事件输出。

## 文件监听与处理

| 接口 | 功能 | mode 说明 |
| --- | --- | --- |
| `HiviewRegisterHieventFileWatcher` | 注册文件满回调 | 回调签名 `FileProc(path, type, event)` |
| `HiEventFileProcImp` | 处理事件文件 | 0=复制保留源文件，1=重命名移动 |
| `HiEventOutputFileLockImp`/`UnLockImp` | 文件操作互斥锁 | 保护文件并发访问 |

## 测试指引

- 完整版输出变更：验证缓存写入、文件刷盘、UART 输出路径。
- mini 版变更：验证 `OutputEvent` 通过 `printf` 输出，文件相关接口为空实现。
- 重试逻辑变更：模拟文件初始化失败，验证重试计数和上限行为。
- 刷盘变更：验证 `HiEventFlush(TRUE)` 同步刷盘和 `HiEventFlush(FALSE)` 异步消息。
- 涉及真实文件系统时补充 `board-verification.md` 中的板侧证据。
