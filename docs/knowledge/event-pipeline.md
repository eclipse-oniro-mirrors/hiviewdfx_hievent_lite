# 事件打点流水线知识

本文只记录事件创建、编码、参数添加和上报主链路中容易被改错的边界。输出后端差异见 `event-output-modes.md`，公共 API 约束见 `public-api.md`。

## 主链路

事件打点应保持阶段清晰：

1. 检查 `g_hiviewConfig.eventSwitch` 是否开启。
2. 创建事件对象（`HiEventCreate`）或快速打点（`HiEventPrintf`）。
3. 编码参数为 T-L-V 格式并写入 payload（`HiEventEncode`、`HiEventPutInteger`）。
4. 上报事件触发输出（`HiEventReport` / `OutputEvent`）。
5. 释放事件内存（`HiEventReport` 自动释放）。

策略决策不要下沉到编码或输出代码。改变事件类型、参数编码或输出选项时，先确认配置开关状态，再执行对应操作。

## 单参数快速打点

`HiEventPrintf(type, eventId, key, value)` 是最常用的打点方式，对应宏 `HIEVENT_FAULT_REPORT`、`HIEVENT_UE_REPORT`、`HIEVENT_STAT_REPORT`。

- `key` 为 -1 时表示无参数，payload 为 NULL。
- `key` 为 0-15 时编码单个 T-L-V 到栈缓冲区 `encodeBuffer[5]`。
- 事件结构体 `HiEvent e` 在栈上构造，直接传给 `OutputEvent`，无需手动释放。

宏通过 `HIEVENT_COMPILE_TYPE` 控制编译期裁剪：若对应类型未编译，宏展开为空。

## 多参数事件

`HiEventCreate(type, eventId, num)` 创建携带 2-16 个参数的事件：

- `num` 超过 `EVENT_VALUE_MAX_NUM`（16）时返回 NULL。
- 事件对象和 payload 均通过 `HIVIEW_MemAlloc` 在 `MEM_POOL_HIVIEW_ID` 内存池分配。
- `common.mark` 在创建时临时存储剩余参数计数，每调用一次 `HiEventPutInteger` 递减 1。
- `common.len` 初始为 0，每次 `HiEventPutInteger` 累加编码长度。
- 最后一个参数（`mark` 降为 1 时）编码时 `tag.last = 1`，其余参数 `tag.last = 0`。

**关键约束：** `HiEventPutInteger` 在 `key < 0` 或 `mark == 0`（参数已用完）时静默返回。不要依赖调用顺序以外的保护。

## T-L-V 编码格式

`HiEventEncode(k, v, last, encodeOut)` 将 key-value 编码为变长 T-L-V：

| 值范围 | tag.len | payload 字节数 | 总编码长度 |
| --- | --- | --- | --- |
| 0 - 0xFF | 1 | 1 | 2 |
| 0 - 0xFFFF | 2 | 2 | 3 |
| 0 - 0x00FFFFFF | 3 | 3 | 4 |
| 其他（含负数） | 4 | 4 | 5 |

- `HiEventTag` 结构为 `{ last:1, id:4, len:3 }`，占 1 字节，紧凑排列。
- 负数值始终使用 4 字节编码。
- 编码使用小端字节序（`GET_UINT32_BYTE1` 取最低字节）。
- 运行时若 `LITTLE_ENDIAN_SYSTEM` 定义，`OutputEvent` 中对 `eventId` 和 `time` 做大端转换以写文件。

**关键约束：** 修改编码长度档位或 tag 布局会破坏已序列化事件的解析兼容性。

## 上报与释放

`HiEventReport(event)` 的行为：

1. 检查 `eventSwitch`、`event`、`payload` 非空。
2. 检查 `common.mark == 0`（所有参数已添加），恢复 `mark = EVENT_INFO_HEAD`。
3. 调用 `OutputEvent` 输出。
4. 释放 payload 内存，释放 event 内存。

**关键约束：** `HiEventReport` 之后 event 指针已释放，不可再访问。若 `mark != 0`（参数未添加完），事件不会输出但仍会被释放。

## 典型边界

| 流程 | 锚点 | 要点 |
| --- | --- | --- |
| 宏打点 | `HIEVENT_*_REPORT(id, k, v)` -> `HiEventPrintf` | 编译期裁剪，运行时开关检查 |
| 多参数创建 | `HiEventCreate` -> `HIVIEW_MemAlloc` | mark 临时存储剩余参数数 |
| 参数编码 | `HiEventPutInteger` -> `HiEventEncode` | 最后一个参数 last=1 |
| 上报输出 | `HiEventReport` -> `OutputEvent` -> 释放 | mark 恢复为 EVENT_INFO_HEAD |
| 编译期裁剪 | `HIEVENT_COMPILE_TYPE` / `IS_COMPILE_EVENT` | 类型未编译时宏展开为空 |

## 测试指引

- 事件创建和编码变更：验证 `HiEventCreate` 返回非空，`HiEventPutInteger` 后 `common.len` 递增，`HiEventReport` 后内存释放。
- 编码格式变更：验证不同值范围（0、255、256、65535、65536、负数）的编码长度和字节序正确。
- 宏裁剪变更：修改 `HIEVENT_COMPILE_TYPE` 后验证对应宏展开行为。
- 输出验证见 `event-output-modes.md` 和 `board-verification.md`。
