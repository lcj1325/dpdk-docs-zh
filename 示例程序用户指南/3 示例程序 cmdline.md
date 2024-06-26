
# 3. 命令行示例程序

本章介绍数据平面开发套件 (DPDK) 部分命令行示例程序。

## 3.1.概述

命令行示例应用程序是一个简单的应用程序，演示了 DPDK 中命令行界面的使用。该应用程序是一个类似 readline 的接口，可用于在 Linux* 应用程序环境中调试 DPDK 应用程序。

> **note:** 
> rte_cmdline 库不应在生产代码中使用，因为它未按照与其他 DPDK 库相同的标准进行验证。另请参阅发行说明的“Known Issues”部分中的“rte_cmdline library should not be used in production code due to limited testing”项。

命令行示例应用程序支持 GNU readline 库的一些功能，例如完成、剪切/粘贴和其他一些特殊绑定，使配置和调试更快、更容易。

该应用程序展示了如何扩展 rte_cmdline 应用程序以处理对象列表。有三个简单的命令：
- add obj_name IP：添加一个与其关联的 IP/IPv6 地址的新对象。
- del obj_name：删除指定对象。
- show obj_name：显示与指定对象关联的 IP。

> **note:**
> 要终止应用程序，请使用 **Ctrl+d**。

## 3.2.编译应用程序

要编译示例应用程序，请参阅 [Compiling the Sample Applications](https://doc.dpdk.org/guides/sample_app_ug/compiling.html)

该应用程序位于 `cmd_line` 子目录中。

## 3.3.运行应用程序

要在 Linux 环境中运行该应用程序，请发出以下命令：
```
./<build_dir>/examples/dpdk-cmdline -l 0-3 -n 4
```

有关运行应用程序和环境抽象层 (EAL, Environment Abstraction Layer) 选项的一般信息，请参阅 *DPDK Getting Started Guide*。

## 3.4.解释

以下部分提供了代码的一些解释。

### 3.4.1. EAL 初始化和 cmdline 启动

第一个任务是环境抽象层（EAL, Environment Abstraction Layer）的初始化。这是通过以下方式实现的：

```
int main(int argc, char **argv)
{
	int ret;
	struct cmdline *cl;

	ret = rte_eal_init(argc, argv);
	if (ret < 0)
		rte_panic("Cannot init EAL\n");
```

然后，创建一个新的命令行对象并开始通过控制台与用户交互：
```
cl = cmdline_stdin_new(main_ctx, "example> ");
if (cl == NULL)
	rte_panic("Cannot create cmdline instance\n");
cmdline_interact(cl);
cmdline_stdin_exit(cl);
```

当用户键入 **Ctrl+d** 时，cmd line_interact() 函数返回，在这种情况下，应用程序退出。

### 3.4.2.定义命令行上下文

命令行上下文是在 NULL 终止表中列出的命令列表，例如：
```
cmdline_parse_ctx_t main_ctx[] = {
	(cmdline_parse_inst_t *)&cmd_obj_del_show,
	(cmdline_parse_inst_t *)&cmd_obj_add,
	(cmdline_parse_inst_t *)&cmd_help,
	NULL,
};
```

每个命令（cmdline_parse_inst_t 类型）都是静态定义的。它包含一个指向解析命令时执行的回调函数的指针、一个不透明指针、一个帮助字符串和一个以 NULL 结尾的表中的标记列表。

rte_cmdline 应用程序提供了预定义令牌类型的列表：
- String Token：匹配静态字符串、静态字符串列表或任何字符串。
- Number Token：匹配可以有符号或无符号的数字，从 8 位到 32 位。
- IP Address Token：匹配 IPv4 或 IPv6 地址或网络。
- Ethernet* Address Token：匹配 MAC 地址。

在此示例中，在 parse_obj_list.c 和 parse_obj_list.h 文件中定义并实现了新的标记类型 obj_list。

例如，cmd_obj_del_show命令定义如下：

```
struct cmd_obj_del_show_result {
	cmdline_fixed_string_t action;
	struct object *obj;
};

static void cmd_obj_del_show_parsed(void *parsed_result,
				    struct cmdline *cl,
				    __rte_unused void *data)
{
	struct cmd_obj_del_show_result *res = parsed_result;
	char ip_str[INET6_ADDRSTRLEN];

	if (res->obj->ip.family == AF_INET)
		snprintf(ip_str, sizeof(ip_str), NIPQUAD_FMT,
			 NIPQUAD(res->obj->ip.addr.ipv4));
	else
		snprintf(ip_str, sizeof(ip_str), NIP6_FMT,
			 NIP6(res->obj->ip.addr.ipv6));

	if (strcmp(res->action, "del") == 0) {
		SLIST_REMOVE(&global_obj_list, res->obj, object, next);
		cmdline_printf(cl, "Object %s removed, ip=%s\n",
			       res->obj->name, ip_str);
		free(res->obj);
	}
	else if (strcmp(res->action, "show") == 0) {
		cmdline_printf(cl, "Object %s, ip=%s\n",
			       res->obj->name, ip_str);
	}
}

cmdline_parse_token_string_t cmd_obj_action =
	TOKEN_STRING_INITIALIZER(struct cmd_obj_del_show_result,
				 action, "show#del");
parse_token_obj_list_t cmd_obj_obj =
	TOKEN_OBJ_LIST_INITIALIZER(struct cmd_obj_del_show_result, obj,
				   &global_obj_list);

cmdline_parse_inst_t cmd_obj_del_show = {
	.f = cmd_obj_del_show_parsed,  /* function to call */
	.data = NULL,      /* 2nd arg of func */
	.help_str = "Show/del an object",
	.tokens = {        /* token list, NULL terminated */
		(void *)&cmd_obj_action,
		(void *)&cmd_obj_obj,
		NULL,
	},
};
```

该命令由两个标记组成：
- 第一个标记是一个字符串标记，可以是 show 或 del。
- 第二个标记是之前使用 add 命令添加到 global_obj_list 变量中的对象。

解析命令后，rte_cmdline 应用程序将填充 cmd_obj_del_show_result 结构。指向该结构的指针作为回调函数的参数给出，并且可以在此函数的主体中使用。
