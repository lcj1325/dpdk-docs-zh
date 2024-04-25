# 7. 命令行库

从最早的版本开始，DPDK 就包含了一个命令行库 - 主要供 `dpdk-testpmd` 和 `dpdk-test` 二进制文件等内部使用，但该库也会在安装时导出，可供任何最终应用程序使用。本章介绍命令行库的基础知识以及如何在应用程序中使用它。

## 7.1. 库功能

DPDK命令行库支持以下功能：

- 交互式终端会话下可用 Tab 补全
- 能够读取和处理从输入文件中获取的命令，例如启动脚本
- 参数化命令能够采用不同数据类型的多个参数：
    - 字符串
    - 有符号/无符号 16/32/64 位整数 
    - IP地址 
    - 以太网地址

## 7.2.将命令行添加到应用程序

将命令行实例添加到应用程序涉及许多编码步骤。

1. 定义命令的结果结构，指定命令参数
2. 为结果中的每个字段提供一个初始值设定项
3. 定义命令的回调函数
4. 为命令提供解析结果结构实例，将回调链接到命令
5. 将解析结果结构添加到命令行上下文
6. 在主应用程序代码中，创建一个在上下文中传递的新命令行实例。

其中许多步骤可以使用 DPDK 安装的脚本 `dpdk-cmdline-gen.py` 自动执行，该脚本可在源树的 `buildtools` 文件夹中找到。本节介绍使用此脚本添加命令行来生成样板，而以下部分“[向应用程序添加命令行的工作示例](https://doc.dpdk.org/guides/prog_guide/cmdline.html#worked-example-of-adding-command-line-to-an-application)”介绍了手动执行此操作的步骤。

### 7.2.1.创建命令列表文件

dpdk-cmdline-gen.py 脚本将应用程序使用的命令列表作为输入。虽然这些可以通过标准输入通过管道传输到它，但使用列表文件可能是最好的。

列表文件的格式必须是：

- 注释行以“#”作为第一个非空白字符开头
- 每行一个命令
- 变量字段以尖括号中的类型名称为前缀，例如：
    - <STRING>message
    - <UINT16>port_id
    - <IP>src_ip
    - <IPv4>dst_ip4
    - <IPv6>dst_ip6
- 变量字段从选项列表中获取值，将逗号分隔的选项列表放在大括号中，而不是类型名称。例如:
    - <(rx,tx,rxtx)>mode
- 命令的帮助文本以注释的形式给出，与命令位于同一行

包含各种（不相关）命令的示例列表文件如下所示：

```bash
# example list file
list                     # show all entries
add <UINT16>x <UINT16>y  # add x and y
echo <STRING>message     # print message to screen
add socket <STRING>path  # add unix socket with the given path
set mode <(rx,tx)>rxtx   # set Rx-only or Tx-only mode
quit                     # close the application
```

### 7.2.2.运行生成器脚本

要生成命令行所需的定义，请运行 `dpdk-cmdline-gen.py`，并将列表文件作为参数传递。该脚本会将生成的C代码输出到标准输出，其内容采用C头文件的形式。或者，可以通过 `-o/--output-file` 参数指定输出文件名。

生成的内容包括：

- 每个命令的结果结构定义
- 每个结构体字段的标记初始值设定项
- 每个命令的回调的“外部”函数原型
- 每个命令的解析上下文，包括每个命令的注释作为帮助字符串
- 命令行上下文数组定义，适合传递给 `cmdline_new`

如果需要，脚本还可以输出每个命令的回调函数的函数存根。此行为是通过将 `--stubs` 标志传递给脚本来触发的。在这种情况下，输出文件必须提供以“.h”结尾的文件名，并且回调存根将被写入等效的“.c”文件。

> note:
> 存根被写入一个单独的文件，以允许连续使用脚本重新生成命令行头，而不会覆盖用户添加到回调函数中的任何代码。这使得向现有应用程序增量添加新命令变得容易。

### 7.2.3.提供函数回调

如上所述，脚本输出是一个头文件，包含结构定义，但回调函数本身显然必须由用户提供。这些回调函数必须作为 C 文件中的非静态函数提供，并命名为 `cmd_<cmdname>_parsed`。函数原型可以在生成的输出标头中看到。

函数名称的“cmdname”部分是通过组合命令中的非可变初始标记构建的。因此，考虑到我们下面的工作示例中的命令：`quit` 和 `show port stats <n>`, 回调函数将是：

```c
void
cmd_quit_parsed(void *parsed_result, struct cmdline *cl, void *data)
{
     ...
}

void
cmd_show_port_stats_parsed(void *parsed_result, struct cmdline *cl, void *data)
{
     ...
}
```
这些函数必须由开发人员提供，但是，如上所述，存根函数可以由脚本使用 `--stubs` 参数自动生成。

相同的“cmdname”词干也用于生成的结构的命名。要获取上述每个命令的结果结构，`parsed_result` 参数应分别转换为 `struct cmd_quit_result` 或 `struct cmd_show_port_stats_result`。

### 7.2.4.与应用程序集成

要将脚本输出与应用程序集成，我们必须将生成的标头 `#include` 到我们的应用程序 C 文件中，然后通过 `cmdline_new` 或 `cmdline_stdin_new` 创建命令行。函数调用的第一个参数应该是生成的头文件中的上下文数组，默认为 `ctx`。 （可通过脚本参数修改）。

回调函数可以位于同一个文件中，也可以位于单独的文件中 - 它们只需要在构建时可供链接器使用。

### 7.2.5.脚本方法的局限性

脚本方法适用于用户可能希望添加到应用程序的大多数命令。但是，它不支持 DPDK 命令行库可能提供的全部功能。例如，不可能使用脚本将多个命令复用到单个回调函数中。要使用此功能，用户应按照下一节“[向应用程序添加命令行的工作示例](https://doc.dpdk.org/guides/prog_guide/cmdline.html#worked-example-of-adding-command-line-to-an-application)”中的说明来手动配置命令行实例。

## 7.3.向应用程序添加命令行的工作示例

接下来的几个小节将更详细地介绍[向应用程序添加命令行](https://doc.dpdk.org/guides/prog_guide/cmdline.html#adding-command-line-to-an-application)中列出的每个步骤，并通过示例向命令行实例添加两个命令。

这两个命令将是：

1. `quit` - 顾名思义，关闭应用程序
2. `show port stats <n>` - 在屏幕上显示给定以太网端口的统计信息

> note:
> 有关使用命令行的更多示例，请参阅 [cmdline 示例应用程序](https://doc.dpdk.org/guides/sample_app_ug/cmd_line.html)

### 7.3.1.定义命令结果结构

要定义的第一个结构是将在成功解析命令时创建的结构。该结构为命令中的每个标记或单词包含一个成员字段。最简单的情况是单字命令，例如 `quit`。为此，我们只需要定义一个具有单个字符串参数的结构来包含该单词。

```c
struct cmd_quit_result {
   cmdline_fixed_string_t quit;
};
```

为了便于阅读，结构成员的名称应与命令中标记的名称相匹配。

对于我们的第二个命令，我们需要一个包含四个成员字段的结构，因为我们的命令中有四个单词/标记。前三个是字符串，最后一个是 16 位数值。生成的结构如下所示：

```c
struct cmd_show_port_stats_result {
   cmdline_fixed_string_t show;
   cmdline_fixed_string_t port;
   cmdline_fixed_string_t stats;
   uint16_t n;
};
```

和以上一样，我们选择名称来匹配命令中的标记。由于我们的数字参数是 16 位值，因此我们使用 uint16_t 类型。任何标准大小的整数类型都可以用作参数，具体取决于所需的结果。

除了标准整数类型之外，该库还允许变量参数为许多其他类型，如上面的功能列表中所述。

- 对于可变字符串参数，类型应为 `cmdline_fixed_string_t` - 与固定标记相同，但这些参数的初始化方式不同（如下所述）。
- 对于以太网地址，请使用 struct rte_ether_addr 类型
- 对于 IP 地址，请使用类型 cmdline_ipaddr_t

### 7.3.2.提供字段初始化器

结果结构的每个字段都需要一个初始值设定项。对于固定字符串标记，如“quit”、“show”和“port”，初始化器将是字符串本身。

```c
static cmdline_parse_token_string_t cmd_quit_quit_tok =
   TOKEN_STRING_INITIALIZER(struct cmd_quit_result, quit, "quit");
```

这里使用的命名约定是包括整个结果结构的基本名称 - 在本例中为 `cmd_quit`，以及该结构中字段的名称 - 在本例中为 `quit`，后跟 `_tok`。（这就是为什么上面的名字中有两个`quit`的原因）。

在我们的第二个示例中可以看到这种命名约定，该示例还演示了如何定义数字初始值设定项。

```c
static cmdline_parse_token_string_t cmd_show_port_stats_show_tok =
   TOKEN_STRING_INITIALIZER(struct cmd_show_port_stats_result, show, "show");
static cmdline_parse_token_string_t cmd_show_port_stats_port_tok =
   TOKEN_STRING_INITIALIZER(struct cmd_show_port_stats_result, port, "port");
static cmdline_parse_token_string_t cmd_show_port_stats_stats_tok =
   TOKEN_STRING_INITIALIZER(struct cmd_show_port_stats_result, stats, "stats");
static cmdline_parse_token_num_t cmd_show_port_stats_n_tok =
   TOKEN_NUM_INITIALIZER(struct cmd_show_port_stats_result, n, RTE_UINT16);
```

对于可变字符串标记，应使用相同的 TOKEN_STRING_INITIALIZER 宏。但是，最后一个参数应该是 NULL，而不是硬编码的标记字符串。

对于数字参数，TOKEN_NUM_INITIALIZER 宏的最终参数应该是与结果结构中定义的变量类型匹配的命令行类型，例如RTE_UINT8、RTE_UINT32 等。

对于 IP 地址，应使用宏 TOKEN_IPADDR_INITIALIZER。

对于以太网地址，应使用宏 TOKEN_ETHERADDR_INITIALIZER。

### 7.3.3.定义回调函数

对于每个命令，我们需要定义一个在命令被识别后调用的函数。回调函数的类型应为：

```c
void (*f)(void *, struct cmdline *, void *)
```

其中第一个参数是指向上面定义的结果结构的指针，第二个参数是命令行实例，最后一个参数是当我们将回调与命令关联时提供的用户定义的指针。大多数回调函数仅使用第一个参数，或者根本不使用第一个参数，但附加的两个参数提供了一些额外的灵活性，以允许回调在应用程序中处理非全局状态。

对于我们的两个示例命令，相关回调函数的定义看起来非常相似。但是，在函数体内，我们假设用户需要引用结果结构来提取第二种情况下的端口号。

```c
void
cmd_quit_parsed(void *parsed_result, struct cmdline *cl, void *data)
{
   quit = 1;
}
void
cmd_show_port_stats_parsed(void *parsed_result, struct cmdline *cl, void *data)
{
   struct cmd_show_port_stats_result *res = parsed_result;
   uint16_t port_id = res->n;
   ...
}
```

### 7.3.4.关联回调和命令

`cmdline_parse_inst_t` 类型定义了一个“解析实例”，即要匹配的标记序列，然后是要调用的关联函数。实例类型中还包括命令的帮助文本字段，以及要传递给上面引用的回调函数的任何其他用户定义参数。例如，对于我们简单的“退出”命令：

```c
static cmdline_parse_inst_t cmd_quit = {
    .f = cmd_quit_parsed,
    .data = NULL,
    .help_str = "Close the application",
    .tokens = {
        (void *)&cmd_quit_quit_tok,
        NULL
    }
};
```

在这种情况下，我们首先确定要调用的回调函数，然后将用户自定义参数设置为NULL，在最终列出要与该命令实例匹配的单个标记之前，根据请求向用户提供一条帮助消息来解释该命令。

对于我们的第二个端口统计示例，以及通过匹配多个标记使事情变得更加复杂，我们还可以演示向函数传递参数。假设我们的应用程序并不总是使用所有可用的端口，而是仅使用端口的子集，存储在名为 `active_ports` 的数组中。因此，我们的 `stats` 命令应该只显示当前正在使用的端口的统计信息，因此我们传递此 `active_ports` 数组。（为了简单起见，我们假设数组使用终止标记，例如 -1 表示端口列表的末尾，因此我们也不需要传入长度参数。）

```c
extern int16_t active_ports[];
...
static cmdline_parse_inst_t cmd_show_port_stats = {
    .f = cmd_show_port_stats_parsed,
    .data = active_ports,
    .help_str = "Show statistics for active network ports",
    .tokens = {
        (void *)&cmd_show_port_stats_show_tok,
        (void *)&cmd_show_port_stats_port_tok,
        (void *)&cmd_show_port_stats_stats_tok,
        (void *)&cmd_show_port_stats_n_tok,
        NULL
    }
};
```

### 7.3.5.将命令添加到命令行上下文

现在我们已经配置了每个单独的命令和回调，我们需要将它们合并到单个命令行“上下文”数组中。该上下文数组将用于在应用程序中创建实际的命令行实例。值得庆幸的是，每个上下文条目与每个解析实例相同，因此我们的数组是通过简单地列出先前定义的命令解析实例来定义的。

```c
static cmdline_parse_ctx_t ctx[] = {
    &cmd_quit,
    &cmd_show_port_stats,
    NULL
};
```

上下文列表必须以 NULL 条目终止。

### 7.3.6.创建命令行实例

定义了 `ctx` 变量后，我们现在只需调用 API 即可在应用程序中创建新的命令行实例。基本 API 是 `cmdline_new` ，它将创建一个包含所有可用命令的交互式命令行。但是，如果需要用于交互式使用的其他功能（例如制表符不全），建议改用 `cmdline_new_stdin`。

可以在应用程序中使用的一种模式是使用 `cmdline_new` 处理来自文件或环境的任何启动命令（如“dpdk-test”应用程序中所做的那样），然后使用 `cmdline_stdin_new` 此后处理交互部分。例如，要处理启动文件，然后提供交互式提示：

```c
struct cmdline *cl;
int fd = open(startup_file, O_RDONLY);

if (fd >= 0) {
    cl = cmdline_new(ctx, "", fd, STDOUT_FILENO);
    if (cl == NULL) {
        /* error handling */
    }
    cmdline_interact(cl);
    cmdline_quit(cl);
    close(fd);
}

cl = cmdline_stdin_new(ctx, "Proxy>> ");
if (cl == NULL) {
    /* error handling */
}
cmdline_interact(cl);
cmdline_stdin_exit(cl);
```

### 7.3.7.将多个命令复用到单个函数

为了减少为应用程序创建命令行时所需的样板代码量，可以将多个命令合并在一起，让它们调用单独的函数。这可以通过多种不同的方式来完成：

- 回调函数可以用作许多不同命令的目标。可以通过检查第一个参数（上面示例中的 `parsed_result`）来确定使用哪个命令进入该函数。

对于简单的字符串命令，可以使用“#”字符连接多个选项。例如：`exit#quit`，指定为令牌初始值设定项，将匹配字符串“exit”或字符串“quit”。

作为一个具体示例，这两种技术用于 DPDK 单元测试应用程序 `dpdk-test`，其中单个命令 `cmdline_parse_t` 实例用于所有“dump_<item>”测试用例。

```c
static void cmd_dump_parsed(void *parsed_result,
			    __rte_unused struct cmdline *cl,
			    __rte_unused void *data)
{
	struct cmd_dump_result *res = parsed_result;

	if (!strcmp(res->dump, "dump_physmem"))
		rte_dump_physmem_layout(stdout);
	else if (!strcmp(res->dump, "dump_memzone"))
		rte_memzone_dump(stdout);
	else if (!strcmp(res->dump, "dump_struct_sizes"))
		dump_struct_sizes();
	else if (!strcmp(res->dump, "dump_ring"))
		rte_ring_list_dump(stdout);
	else if (!strcmp(res->dump, "dump_mempool"))
		rte_mempool_list_dump(stdout);
	else if (!strcmp(res->dump, "dump_devargs"))
		rte_devargs_dump(stdout);
	else if (!strcmp(res->dump, "dump_log_types"))
		rte_log_dump(stdout);
	else if (!strcmp(res->dump, "dump_malloc_stats"))
		rte_malloc_dump_stats(stdout, NULL);
	else if (!strcmp(res->dump, "dump_malloc_heaps"))
		rte_malloc_dump_heaps(stdout);
}

cmdline_parse_token_string_t cmd_dump_dump =
	TOKEN_STRING_INITIALIZER(struct cmd_dump_result, dump,
				 "dump_physmem#"
				 "dump_memzone#"
				 "dump_struct_sizes#"
				 "dump_ring#"
				 "dump_mempool#"
				 "dump_malloc_stats#"
				 "dump_malloc_heaps#"
				 "dump_devargs#"
				 "dump_log_types");

cmdline_parse_inst_t cmd_dump = {
	.f = cmd_dump_parsed,  /* function to call */
	.data = NULL,      /* 2nd arg of func */
	.help_str = "dump status",
	.tokens = {        /* token list, NULL terminated */
		(void *)&cmd_dump_dump,
		NULL,
	},
};
```

## 7.4. DPDK 中的命令行使用示例

为了帮助用户遵循上面提供的步骤，可以查阅以下 DPDK 文件以获取命令行使用示例。

> note:
> 这并不是 DPDK 中命令行使用示例的详尽列表。它只是应用程序开发人员可能有用的一些文件的列表。其中一些引用的文件包含比其他文件更复杂的使用示例。

- `commands.c/.h` in `examples/cmdline`
- `mp_commands.c/.h` in `examples/multi_process/simple_mp`
- `commands.c` in `app/test`



