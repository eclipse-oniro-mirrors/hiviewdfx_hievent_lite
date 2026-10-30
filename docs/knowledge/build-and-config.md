# 构建与配置知识

本文记录构建参数、编译开关、定制实现和 mini 适配的相关知识。公共 API 约束见 `public-api.md`，板侧验证见 `board-verification.md`。

## 构建系统

本仓库使用 GN + Ninja 构建系统，通过 `BUILD.gn` 定义构建目标：

| 目标 | 类型 | 说明 |
| --- | --- | --- |
| `hievent_lite_static` | static_library | 核心静态库，含事件创建和输出实现 |
| `hievent_lite` | group | 顶层目标，根据定制开关选择实现 |

`hievent_lite_command` 在 `command/BUILD.gn` 中单独定义为静态库。

## 可配置参数

`BUILD.gn` 的 `declare_args()` 定义以下可配置参数：

| 参数 | 默认值 | 编译宏 | 说明 |
| --- | --- | --- | --- |
| `hievent_lite_fault_file_size` | 1024 | `FAULT_EVENT_FILE_SIZE` | 故障事件文件大小（字节） |
| `hievent_lite_ue_file_size` | 1024 | `UE_EVENT_FILE_SIZE` | 用户行为事件文件大小 |
| `hievent_lite_stat_file_size` | 1024 | `STAT_EVENT_FILE_SIZE` | 统计事件文件大小 |
| `hievent_lite_cache_size` | 256 | `EVENT_CACHE_SIZE` | 每类事件缓存大小 |
| `hievent_lite_file_buffer_size` | 128 | `HIVIEW_HIEVENT_FILE_BUF_SIZE` | 触发刷盘的缓存阈值 |
| `hievent_lite_customize_implementation` | false | - | 启用定制实现 |
| `hievent_lite_mini` | false | `HIEVENT_LITE_MINI` | 启用 mini 输出后端 |

这些参数在产品配置文件（如 `vendor/<vendor>/<product>/config.json`）中覆盖。

## mini 适配

`hievent_lite_mini = true` 时的行为：

1. 编译 `frameworks/hiview_output_event_mini.c` 替代 `frameworks/hiview_output_event.c`。
2. 定义 `HIEVENT_LITE_MINI` 宏。
3. `HiEventInit` 中跳过 `InitCoreEventOutput` 和 `InitEventOutput` 调用。
4. `HiEventFlush` 调用 `FlushEvent` 被条件编译移除（`#ifndef HIEVENT_LITE_MINI`）。
5. 文件相关接口为空实现。

**关键约束：** mini 版不含文件系统依赖，适合无 FLASH 或 FLASH 极小的 mini 设备。切换 mini 开关后需重新编译整个组件。

## 定制实现

`hievent_lite_customize_implementation = true` 时的行为：

1. `hievent_lite` group 不再依赖 `hievent_lite_static`。
2. 仅暴露 `hiview_lite_config` 公共配置。
3. 实现方需自行提供 `hiview_output_event.h` 中所有接口的实现。

**关键约束：** 定制实现和默认实现不可混用。定制实现需完整实现所有接口，包括 mini 版中的空实现接口。

## 依赖关系

`bundle.json` 声明的依赖：

| 依赖类型 | 组件 | 用途 |
| --- | --- | --- |
| component | `hiview_lite` | hiview 基础设施（缓存、文件、配置、互斥锁、工具函数） |
| component | `samgr_lite` | 系统服务管理框架 |

包含路径依赖（`include_dirs`）：

| 路径 | 用途 |
| --- | --- |
| `//base/hiviewdfx/hievent_lite/interfaces/native/innerkits` | 本组件公共 API |
| `//base/hiviewdfx/hilog_lite/interfaces/native/kits/hilog_lite` | 轻量日志 |
| `//base/hiviewdfx/hiview_lite` | hiview 核心（`hiview_def.h`、`hiview_file.h` 等） |
| `//commonlibrary/utils_lite/include` | 通用工具（`ohos_types.h`） |
| `//third_party/bounds_checking_function/include` | 安全函数库（`securec.h`） |
| `//foundation/systemabilitymgr/samgr_lite/interfaces/kits/samgr` | 服务管理 |

## 组件清单

`bundle.json` 关键字段：

| 字段 | 值 | 说明 |
| --- | --- | --- |
| `name` | `@ohos/hievent_lite` | 组件全名 |
| `component.name` | `hievent_lite` | 组件短名 |
| `component.subsystem` | `hiviewdfx` | 所属子系统 |
| `component.adapted_system_type` | `["mini"]` | 适配系统类型 |
| `component.rom` | `26KB` | ROM 占用 |
| `component.ram` | `~10KB` | RAM 占用 |
| `component.version` | `3.1` | 组件版本 |

## 构建命令

从 OpenHarmony 源码根目录执行：

```sh
# 标准构建（完整版输出后端）
./build.sh --product-name <product-name> --build-target hievent_lite --ccache

# mini 系统构建（需产品配置中设置 hievent_lite_mini = true）
./build.sh --product-name wifiiot_hispark_pegasus --build-target hievent_lite --ccache

# 单独构建命令行库
prebuilts/build-tools/linux-x86/bin/ninja -C out/<product> hievent_lite_command
```

## 修改构建参数的检查清单

1. 修改文件大小参数是否影响 FLASH 占用和事件容量？
2. 修改缓存大小是否影响内存占用（RAM ~10KB 预算）？
3. 修改 `file_buffer_size` 阈值是否影响刷盘频率和 FLASH 寿命？
4. 启用 mini 开关后是否移除了文件系统依赖？
5. 启用定制实现后是否提供了完整的接口实现？
6. 新增编译宏是否在 `defines` 中正确声明？
7. 新增源文件是否在 `sources` 中正确添加（含 mini 条件分支）？
