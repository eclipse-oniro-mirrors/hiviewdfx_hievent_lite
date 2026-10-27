# 板侧验证知识

本文记录轻量事件打点变更需要板侧证据时的稳定做法。构建配置见 `build-and-config.md`，输出后端见 `event-output-modes.md`。

## 构建位置

OpenHarmony 构建命令从源码根目录执行：

```sh
cd <openharmony-source-root>
./build.sh --product-name <product-name> --build-target hievent_lite --ccache
```

mini 系统产品需在产品配置中设置 `hievent_lite_mini = true`。

## 何时需要板侧证据

| 变更类型 | 最低证据 |
| --- | --- |
| 事件输出、文件写入、刷盘逻辑 | 构建通过；在板侧验证事件文件内容或 UART 输出 |
| 缓存机制、缓存阈值 | 构建通过；在板侧验证事件不丢失、不重复 |
| 命令行交互（`hievent -c`/`-h`） | 构建通过；在板侧验证开关行为和帮助输出 |
| mini 版输出后端 | 构建通过；在板侧验证 UART printf 输出 |
| 公共 API 兼容性 | 构建通过；调用方组件（如 hiview_lite）构建通过 |
| 仅构建参数调整 | 构建通过；如运行时行为变化需板侧验证 |
| 重试初始化逻辑 | 构建通过；在板侧模拟文件系统未就绪场景 |

## 板侧运行方式

### UART 输出验证

lite 系统通常通过 UART 串口查看输出：

1. 连接设备 UART 串口。
2. 触发事件打点（调用 `HIEVENT_*_REPORT` 宏或命令行）。
3. 在串口终端确认 `EVENT: time=... id=... type=... data=...` 格式输出。

### 事件文件验证

完整版输出后端的事件文件路径由 `HIVIEW_FILE_PATH_*` 宏定义（来自 hiview_lite）：

1. 触发事件打点，等待缓存刷盘（达到 `HIVIEW_HIEVENT_FILE_BUF_SIZE` 或调用 `HiEventFlush`）。
2. 通过文件系统工具或 shell 读取事件文件。
3. 验证文件头信息和二进制事件数据格式。

### 命令行验证

通过 shell 执行 `hievent` 命令：

```sh
# 查看帮助
hievent -h
# 预期输出：
# hievent [-h] [-c]
#  -h            Help
#  -c            Enable or disable event function

# 开关事件功能
hievent -c
# 预期输出（开启时）：
# Close event function success.
# 预期输出（关闭时）：
# Open event function success.
```

验证开关后事件输出行为是否正确变化。

## 场景化验证

### 事件打点验证

```c
// 在测试代码或业务组件中调用
HIEVENT_FAULT_REPORT(1, 0, 100);     // 故障事件，key=0, value=100
HIEVENT_UE_REPORT(2, 1, 200);        // 用户行为事件
HIEVENT_STAT_REPORT(3, 2, 300);     // 统计事件

// 多参数事件
HiEvent *e = HIEVENT_CREATE(HIEVENT_FAULT, 10, 3);
HIEVENT_PUT_INT_VALUE(e, 0, 100);
HIEVENT_PUT_INT_VALUE(e, 1, 200);
HIEVENT_PUT_INT_VALUE(e, 2, 300);
HIEVENT_REPORT(e);
```

验证 UART 输出或事件文件中包含对应事件。

### 刷盘验证

1. 打点若干事件（不超过缓存大小）。
2. 调用 `HiEventFlush(TRUE)` 同步刷盘。
3. 验证事件文件中包含已打点事件。
4. 重启设备后验证事件未丢失。

### 缓存满验证

1. 打点超过 `EVENT_CACHE_SIZE` 的事件。
2. 验证缓存满时触发刷盘，事件不丢失。
3. 验证空间不足时丢弃事件并打印 `HiEvent have no sufficient space to write` 警告。

### mini 版验证

1. 确认 `hievent_lite_mini = true` 编译。
2. 打点事件后验证 UART printf 输出 `EVENT: time=...` 格式。
3. 验证文件相关接口为空实现，不产生文件操作。

## PR 证据模板

```text
构建：
- 产品：<product-name>
- 目标：hievent_lite
- 结果：<通过/失败，已检查命令输出>
- mini 开关：<true/false>

板侧：
- 设备/镜像：<相关时填写标识>
- 验证方式：<UART 输出 / 事件文件内容 / 命令行输出>
- 命令：<精确的 shell 命令或测试调用>
- 结果：<事件输出正确/错误，含具体内容>
- 环境说明：<无，或已知设置/环境问题>
```
