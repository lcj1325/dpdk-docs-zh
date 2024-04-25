# 5. 日志库

DPDK 日志库为其他 DPDK 库和驱动程序提供日志记录功能。默认情况下，在 Linux 应用程序中，日志会发送到 syslog 以及控制台。在 FreeBSD 和 Windows 应用程序上，日志仅发送到控制台。但是，用户可以覆盖日志功能以使用不同的日志记录机制。

## 5.1.日志级别

来自应用程序和库的日志消息以给定的严重性级别进行报告。 `rte_log.h` 中指定的这些级别是（从最重要到最不重要）：

1. Emergency
2. Alert
3. Critical
4. Error
5. Warning
6. Notice
7. Information
8. Debug

在运行时，应用程序只会将配置级别或更高级别（即更高重要性）的消息发送到日志输出。该级别可以通过应用程序从日志记录库调用相关 API 来配置，也可以由用户通过应用程序将 `--log-level` 参数传递给 EAL 来配置。

### 5.1.1.设置全局日志级别

要调整应用程序的全局日志级别，只需将数字级别或级别名称传递给 `--log-level EAL` 参数即可。例如：

```bash
/path/to/app --log-level=error

/path/to/app --log-level=debug

/path/to/app --log-level=5   # warning
```

在应用程序中，可以使用 rte_log_set_global_level API 类似地设置日志级别。

## 5.2.使用日志记录 API 生成日志消息

要输出日志消息，应使用 `rte_log()` API 函数。除了日志消息之外，`rte_log()` 还带有两个附加参数：

- 日志级别 
- 日志组件类型

日志级别是如上所述的数值。组件类型是一个唯一的 ID，用于向日志系统标识特定的 DPDK 组件。要获取此 id，每个组件都需要在启动时使用宏 `RTE_LOG_REGISTER_DEFAULT` 注册自身。该宏采用两个参数，第二个参数是组件的默认日志级别。第一个参数称为“type”，是组件中使用的“logtype”或“component type”变量的名称。该变量将由宏定义，并应作为调用 rte_log() 时的第二个参数传递。一般来说，大多数DPDK组件都会定义自己的日志宏来简化对日志API的调用。他们通过以下方式做到这一点：

- 将组件类型参数隐藏在宏内，因此永远不需要显式传递它。
- 使用 `rte_log.h` 中给出的日志级别定义来允许使用短文本名称代替数字日志级别。

以下代码取自 `rte_cfgfile.c`，显示日志注册以及快捷方式日志记录宏的后续定义。它可以用作任何使用 DPDK 日志记录的新组件的模板。

```c
RTE_LOG_REGISTER_DEFAULT(cfgfile_logtype, INFO);
#define RTE_LOGTYPE_CFGFILE cfgfile_logtype

#define CFG_LOG(level, ...) \
	RTE_LOG_LINE_PREFIX(level, CFGFILE, "%s(): ", __func__, __VA_ARGS__)
```

> note:
> 由于日志注册宏提供了 logtype 变量定义，因此应将其放置在使用它的 C 文件的顶部附近。如果不是，则 logtype 变量应定义为文件顶部附近的“extern int”。
>
> 同样，如果一个组件中的多个文件要进行日志记录，则只有一个文件应该通过宏注册日志类型，并且日志类型应该在公共头文件中定义为“extern int”。任何特定于组件的日志记录宏都应该类似地在该标头中定义。

因此，在整个 cfgfile 库中，所有日志记录调用的形式如下：

```c
CFG_LOG(ERR, "missing cfgfile parameters");

CFG_LOG(ERR, "invalid comment characters %c",
       params->comment_character);
```