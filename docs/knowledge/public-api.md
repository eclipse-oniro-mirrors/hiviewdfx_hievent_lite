# 公共 API 约束知识

本文记录公共 C API、数据结构布局、事件类型码和宏定义的兼容性约束。事件流水线见 `event-pipeline.md`，构建配置见 `build-and-config.md`。

## API 分层

公共 API 分两层：

| 层 | 路径 | 用途 |
| --- | --- | --- |
| 兼容头 | `interfaces/native/innerkits/event.h` | 仅 `#include "hiview_event.h"`，保持旧路径兼容 |
| 主头文件 | `interfaces/native/innerkits/hiview_event.h` | 所有公共类型、宏和函数声明 |

业务组件应使用 `event.h` 或 `hiview_event.h`，不要直接包含 `frameworks/` 下的内部头文件。

## 数据结构布局

### HiEventCommon（8 字节，紧凑排列）

```c
#pragma pack(1)
typedef struct {
    uint8  mark;       // EVENT_INFO_HEAD 或临时参数计数
    uint8  len;        // payload 长度
    uint16 eventId;    // 0-65535，全局唯一
    uint32 time;       // Unix 时间戳（秒）
} HiEventCommon;
```

- `#pragma pack(1)` 紧凑布局，字段顺序不可变。
- `mark` 在创建阶段临时存储剩余参数数，上报时恢复为 `EVENT_INFO_HEAD`。
- 写文件时在大端系统上对 `eventId` 和 `time` 做字节序转换。

### HiEvent（运行时结构）

```c
typedef struct {
    HiEventCommon common;
    uint8 type;        // 不写入文件
    uint8 *payload;    // T-L-V 编码数据
} HiEvent;
```

- `type` 和 `payload` 不参与文件序列化，仅运行时使用。
- `payload` 指向堆分配的缓冲区（`HiEventCreate`）或栈缓冲区（`HiEventPrintf`）。

**关键约束：** 修改任何字段顺序、类型或名称会破坏二进制序列化兼容性。

## 事件类型码

| 宏 | 值 | 说明 |
| --- | --- | --- |
| `HIEVENT_NONE` | 0 | 无事件 |
| `HIEVENT_FAULT` | 1 | 故障事件 |
| `HIEVENT_UE` | 2 | 用户行为事件 |
| `HIEVENT_STAT` | 4 | 统计事件 |

- 使用位掩码，通过 `eventType & HIEVENT_FAULT` 判断。
- `HIEVENT_COMPILE_TYPE` 默认为 `HIEVENT_FAULT | HIEVENT_UE | HIEVENT_STAT`，可通过编译参数覆盖。
- `IS_COMPILE_EVENT(t)` 宏检查某类型是否已编译。

**关键约束：** 类型值不可修改。新增类型需使用 8、16 等不冲突的位掩码值，并同步更新 `HIEVENT_COMPILE_TYPE` 默认值和编译开关宏。

## 宏接口

### 单参数宏（编译期裁剪）

| 宏 | 展开条件 | 行为 |
| --- | --- | --- |
| `HIEVENT_FAULT_REPORT(id, k, v)` | `IS_COMPILE_EVENT(HIEVENT_FAULT)` | `HiEventPrintf(HIEVENT_FAULT, id, k, v)` |
| `HIEVENT_UE_REPORT(id, k, v)` | `IS_COMPILE_EVENT(HIEVENT_UE)` | `HiEventPrintf(HIEVENT_UE, id, k, v)` |
| `HIEVENT_STAT_REPORT(id, k, v)` | `IS_COMPILE_EVENT(HIEVENT_STAT)` | `HiEventPrintf(HIEVENT_STAT, id, k, v)` |

未编译时展开为空，不产生任何代码。

### 多参数宏

| 宏 | 展开条件 | 行为 |
| --- | --- | --- |
| `HIEVENT_CREATE(type, id, num)` | `HIEVENT_COMPILE_TYPE > HIEVENT_NONE` | `HiEventCreate(type, id, num)` |
| `HIEVENT_PUT_INT_VALUE(pEvent, k, v)` | 同上 | `HiEventPutInteger(pEvent, k, v)` |
| `HIEVENT_REPORT(pEvent)` | 同上 | `HiEventReport(pEvent)` |

**关键约束：** 宏的参数个数和行为语义不可修改。新增宏需遵循相同的编译期裁剪模式。

## 函数接口

| 函数 | 参数约束 | 返回值 |
| --- | --- | --- |
| `HiEventPrintf(type, eventId, key, value)` | key: -1 无参数, 0-15 有效 | void |
| `HiEventCreate(type, eventId, num)` | num: 2-16, 超过返回 NULL | `HiEvent*` |
| `HiEventPutInteger(event, key, value)` | key: 0-15, event 非空 | void |
| `HiEventReport(event)` | event 非空, mark=0 时才输出 | void（自动释放 event） |
| `HiEventFlush(syncFlag)` | syncFlag: TRUE 同步/FALSE 异步 | void |
| `HiEventRegisterProc(func)` | 全局单回调 | void |
| `HiEventUnRegisterProc(func)` | 置 NULL | void |
| `HiEventFileAddWatcher(type, func, dest)` | type: HIEVENT_FAULT/UE/STAT | void |
| `HiEventFileRemoveWatcher(type, func)` | | void |
| `HiEventFileProc(type, dest, mode)` | mode: 0=copy, 1=rename | 0 成功, -1 失败 |
| `HiEventOutputFileLock/UnLock()` | | void |

## 回调类型

| 类型 | 签名 | 说明 |
| --- | --- | --- |
| `HieventProc` | `boolean (*)(const HiEvent *event)` | 返回 TRUE 跳过默认处理 |
| `FileProc` | `void (*)(const char *path, uint8 type, uint8 event)` | 文件满回调，event=0 表示文件满 |

**关键约束：** 回调类型签名不可修改。`HieventProc` 的返回值语义（TRUE=跳过）不可反转。

## 编译期裁剪

`HIEVENT_COMPILE_TYPE` 可被外部定义覆盖：

```c
#ifndef HIEVENT_COMPILE_TYPE
#define HIEVENT_COMPILE_TYPE (HIEVENT_FAULT | HIEVENT_UE | HIEVENT_STAT)
#endif
```

- 设为 `HIEVENT_NONE` 时，所有宏展开为空，不编译事件功能。
- 设为 `HIEVENT_FAULT` 时，仅编译故障事件打点。

**关键约束：** 修改默认值需评估对已有业务组件的影响。裁剪后宏为空实现，调用方代码不会报错但也不产生事件。

## 兼容性检查清单

修改公共 API 前确认：

1. 是否修改了函数签名（参数个数、类型、返回值）？
2. 是否修改了 `HiEventCommon`/`HiEvent` 结构体字段？
3. 是否修改了事件类型码值？
4. 是否修改了宏的参数个数或展开行为？
5. 是否新增了需要在 mini 版提供空实现的接口？
6. 是否修改了回调类型签名？
7. 是否修改了 `HIEVENT_COMPILE_TYPE` 默认值？

以上任一为"是"时，需评估对依赖本组件的上游业务组件的影响。
