# 轻量事件打点组件指引

## 项目定位

本仓库对应 OpenHarmony `base/hiviewdfx/hievent_lite`，为 LiteOS-M 内核（mini 系统）提供故障、用户行为、功耗统计三类事件的打点、序列化和输出能力。优先按这些目录定位问题：

- `frameworks/`：事件创建、编码、输出核心实现，包含完整版和 mini 版两套输出后端。
- `interfaces/native/innerkits/`：对外公共 C API 头文件和数据结构定义。
- `command/`：shell 命令 `hievent` 的命令行解析与开关控制。
- `BUILD.gn`：构建配置，含可调参数（文件大小、缓存大小、mini 开关等）。
- `bundle.json`：组件清单，声明子系统归属、依赖和适配系统类型。

### 按任务类型定位代码

| 任务类型 | 首选目录 | 关键文件 |
| --- | --- | --- |
| 新增/修改事件类型 | `interfaces/native/innerkits/` | `hiview_event.h`（`HIEVENT_FAULT`/`HIEVENT_UE`/`HIEVENT_STAT` 宏定义） |
| 修改事件创建与编码逻辑 | `frameworks/` | `hiview_event.c`（`HiEventCreate`、`HiEventEncode`、`HiEventPutInteger`） |
| 修改事件输出到文件/UART | `frameworks/` | `hiview_output_event.c`（完整版）、`hiview_output_event_mini.c`（mini 版） |
| 修改缓存与刷盘逻辑 | `frameworks/` | `hiview_output_event.c`（`OutputEvent`、`Output2Flash`、`FlushEvent`） |
| 修改文件监听与文件处理 | `frameworks/` | `hiview_output_event.c`（`HiviewRegisterHieventFileWatcher`、`HiEventFileProcImp`） |
| 修改公共 API 或宏 | `interfaces/native/innerkits/` | `hiview_event.h`（`HIEVENT_*_REPORT`、`HIEVENT_CREATE` 等宏） |
| 修改命令行交互 | `command/` | `hievent_lite_command.c`、`hievent_lite_command.h` |
| 修改构建参数或编译开关 | 根目录 | `BUILD.gn`（`declare_args`）、`bundle.json` |
| 适配新平台输出后端 | `frameworks/` | `hiview_output_event.c` 或新增输出实现文件 |

### 嵌套指引

本仓库无目录级别的嵌套指引。所有任务级指导均通过 `docs/knowledge/` 中的场景文档提供。

## 构建和验证

构建命令从 OpenHarmony 源码根目录执行，不在本子目录执行。

```sh
./build.sh --product-name <product-name> --build-target hievent_lite --ccache
```

mini 系统产品（如 hi3861）使用默认配置；如需启用 mini 输出后端，在产品配置中设置 `hievent_lite_mini = true`。

```sh
./build.sh --product-name wifiiot_hispark_pegasus --build-target hievent_lite --ccache
```

### 完成标准

任务被认为完成，当且仅当：

1. **代码改动已提交** - 使用 `git commit -s`，多代理协作时添加 `Co-Authored-By: Agent`
2. **本地构建通过** - 执行上述构建命令
3. **相关测试通过** - 对应单元测试通过（如适用）
4. **板侧验证（如适用）** - 涉及事件输出、文件写入、命令行交互的改动需提供验证证据
5. **文档更新（如适用）** - 公共 API 修改需更新注释和文档

### 如果无法运行验证

明确说明无法运行的原因，列出推荐的验证步骤供人工执行，标记需要人工验证的部分。

### 完成报告格式

报告应包含：改动摘要（文件列表、改动点）、验证结果（构建/测试输出）、风险评估（API 兼容性、性能风险）、未完成事项。

## 知识索引

稳定背景知识放在 `docs/knowledge/`。改动前按场景读取对应文件：

### 场景与路径路由

| 场景 | 修改目录 | 先读文档 |
| --- | --- | --- |
| 事件创建、T-L-V 编码、参数添加、上报释放、宏接口使用 | `frameworks/`、`interfaces/native/innerkits/` | `docs/knowledge/event-pipeline.md` |
| 完整版与 mini 版输出后端、缓存机制、文件写入、UART 输出、刷盘、重试初始化 | `frameworks/` | `docs/knowledge/event-output-modes.md` |
| 公共 API 约束、数据结构布局、事件类型码、宏定义兼容性 | `interfaces/native/innerkits/` | `docs/knowledge/public-api.md` |
| 构建参数、编译开关、定制实现、mini 适配、依赖关系 | 根目录、`BUILD.gn`、`bundle.json` | `docs/knowledge/build-and-config.md` |
| 构建、板侧测试、命令行验证、事件输出验证 | 任何构建/测试相关改动 | `docs/knowledge/board-verification.md` |

### 开始编辑前

在修改代码前，按以下顺序确认：
1. 确认任务类别
2. 根据上表确定需要阅读的文档
3. 根据"项目约束"确认不违反任何约束
4. 声明："我将修改 X，已阅读 Y 文档，遵循 Z 约束"

## 项目约束

### 性能约束

- 事件打点可能被高频调用（如统计事件），不要在 `HiEventPrintf`、`HiEventPutInteger`、`OutputEvent` 等热路径中增加不必要的字符串格式化或冗余日志。
- mini 系统资源受限（ROM 26KB、RAM ~10KB），新增代码需注意内存占用，避免动态分配过大缓冲区。
- 事件编码采用变长 T-L-V 格式以节省空间，不要改为定长编码。

### 架构约束

- 完整版（`hiview_output_event.c`）和 mini 版（`hiview_output_event_mini.c`）通过 `HIEVENT_LITE_MINI` 宏区分，两套实现必须保持 `hiview_output_event.h` 中声明的接口签名完全一致。
- mini 版不支持文件系统，文件相关接口（`HiEventFileProcImp`、`HiviewRegisterHieventFileWatcher`、`HiEventOutputFileLockImp` 等）在 mini 版中为空实现，不要在其中添加实际逻辑。
- 事件类型使用位掩码（`HIEVENT_FAULT=1`、`HIEVENT_UE=2`、`HIEVENT_STAT=4`），通过 `eventType & HIEVENT_FAULT` 判断，不要改为枚举或线性值。
- `HiEventCommon` 使用 `#pragma pack(1)` 紧凑布局，修改字段时必须保持二进制兼容。
- 自定义输出处理器（`HieventProc`）返回 `TRUE` 时跳过平台默认处理，不要在此回调中执行阻塞操作。

### 编码约定

- C 代码优先复用 `HIVIEW_MemAlloc`/`HIVIEW_MemFree`（`MEM_POOL_HIVIEW_ID`）进行内存管理，不要直接使用 `malloc`/`free`。
- 字符串操作优先使用 `securec` 库的 `memcpy_s`、`snprintf_s`、`strnlen` 等安全函数。
- 日志输出使用 `HIVIEW_UartPrint`（轻量 UART 打印）或 `HILOG_ERROR`/`HILOG_INFO`（hilog），不要混用 `printf`（仅初始化阶段保留的 `printf` 除外）。
- 字节序转换使用 `Change16Endian`/`Change32Endian`，仅在 `LITTLE_ENDIAN_SYSTEM` 宏启用时生效。

### 公共 API 约束

**Do not（禁止）：**
- 修改 `hiview_event.h` 中已发布函数的签名、参数类型、返回值类型
- 修改 `HiEventCommon` 和 `HiEvent` 结构体的字段顺序、字段类型、字段名称
- 修改 `HIEVENT_NONE`、`HIEVENT_FAULT`、`HIEVENT_UE`、`HIEVENT_STAT` 的值
- 修改 `HIEVENT_*_REPORT`、`HIEVENT_CREATE`、`HIEVENT_PUT_INT_VALUE`、`HIEVENT_REPORT` 宏的参数个数和行为语义
- 删除或重命名已有公共 API 函数或宏
- 修改 `event.h` 的兼容性包含关系

**Ask before（修改前必须确认）：**
- 新增公共 API：确认是否需要在 mini 版中提供对应实现
- 新增事件类型：确认位掩码值不与现有类型冲突，且编译开关 `HIEVENT_COMPILE_TYPE` 需同步更新
- 修改 `HiEventCommon` 的 `mark`/`len`/`eventId`/`time` 字段语义：确认不破坏二进制序列化兼容性
- 修改 `EVENT_VALUE_MAX_NUM`（16）或 `SINGLE_VALUE_MAX_LEN`（5）：确认不影响已序列化事件的解析

### 安全与权限边界

**Do not（禁止）：**
- 在事件 payload 中写入敏感明文数据（如密码、密钥、用户隐私信息）
- 绕过 `g_hiviewConfig.eventSwitch` 开关检查直接输出事件
- 在未加锁的情况下并发操作事件缓存或文件句柄

**Ask before（修改前必须确认）：**
- 涉及事件 payload 内容格式的改动：确认不影响上传组件的事件解析
- 涉及文件路径（`HIVIEW_FILE_PATH_*`）的改动：确认目标平台文件系统支持
- 涉及跨组件回调（`HieventProc`、`FileProc`）的改动：确认回调时序和重入安全

### 协议与数据格式兼容性

**Do not（禁止）：**
- 修改 `HiEventCommon` 的 `#pragma pack(1)` 紧凑布局
- 修改 T-L-V 编码的 tag 结构（`HiEventTag` 的 `last`/`id`/`len` 字段布局）
- 修改 `EVENT_INFO_HEAD` 标记值
- 修改事件文件中公共头信息的格式和写入时机

**Ask before（修改前必须确认）：**
- 新增编码长度档位：确认解析端（上传组件）能正确处理
- 修改字节序转换逻辑：确认大小端系统的事件文件可互换

### 定制实现边界

**Do not（禁止）：**
- 在 `hievent_lite_customize_implementation = true` 时直接修改 `hievent_lite_static` 的编译产物
- 混用定制实现和默认实现

**正确做法：**
- 启用 `hievent_lite_customize_implementation` 后，由外部提供 `hiview_lite_config` 配置
- 定制实现需完整实现 `hiview_output_event.h` 中所有接口
- mini 版适配通过 `hievent_lite_mini = true` 启用，编译 `hiview_output_event_mini.c`

### 设备操作约束

**涉及真实设备时的注意事项：**
- lite 系统事件文件存储在 FLASH 上，频繁写入会影响 FLASH 寿命，不要降低 `HIVIEW_HIEVENT_FILE_BUF_SIZE` 缓冲阈值
- 需要在真实设备上验证的改动，必须提供板侧证据（UART 日志、事件文件内容）
- 系统重启前需调用 `HiEventFlush(TRUE)` 同步刷盘，验证时确认事件未丢失
- 命令行 `hievent -c` 开关事件功能后，需确认事件不再输出或恢复输出
