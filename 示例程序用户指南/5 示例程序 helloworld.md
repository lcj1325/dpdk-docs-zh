
# 5.示例应用程序 helloworld

Hello World 示例应用程序是可编写的最简单的 DPDK 应用程序的示例。该应用程序只是在每个启用的 lcore 上打印一条“helloworld”消息。

## 5.1.编译应用程序

要编译示例应用程序，请参阅 [Compiling the Sample Applications](https://doc.dpdk.org/guides/sample_app_ug/compiling.html)

该应用程序位于 `helloworld` 子目录中。

## 5.2.运行应用程序

在linux环境下运行示例：

```
$ ./<build_dir>/examples/dpdk-helloworld -l 0-3 -n 4
```

有关运行应用程序和环境抽象层 (EAL, Environment Abstraction Layer) 选项的一般信息，请参阅 DPDK *Getting Started Guide*。

## 5.3.解释

以下部分提供了一些代码解释。

### 5.3.1. EAL初始化

第一个任务是初始化环境抽象层（EAL, Environment Abstraction Layer ）。这是使用以下代码在 main() 函数中完成的：

```c
int
main(int argc, char **argv)
{
	int ret;
	unsigned lcore_id;

	ret = rte_eal_init(argc, argv);
	if (ret < 0)
		rte_panic("Cannot init EAL\n");
```

此调用完成在调用 main() 之前启动的初始化过程（在 Linux 环境中）。 argc 和 argv 参数提供给 rte_eal_init() 函数。返回的值是已解析参数的数量。

### 5.3.2.启动应用程序单元 lcore

一旦 EAL 初始化，应用程序就准备好在 lcore 上启动函数。在此示例中，在每个可用的 lcore 上调用 lcore_hello()。以下是该函数的定义：

```c
static int
lcore_hello(__rte_unused void *arg)
{
	unsigned lcore_id;
	lcore_id = rte_lcore_id();
	printf("hello from core %u\n", lcore_id);
	return 0;
}
```

在每个 lcore 上启动该函数的代码如下：
```c
RTE_LCORE_FOREACH_WORKER(lcore_id) {
	/* Simpler equivalent. 8< */
	rte_eal_remote_launch(lcore_hello, NULL, lcore_id);
	/* >8 End of simpler equivalent. */
}

/* call it on main lcore too */
lcore_hello(NULL);
```

下面的代码是等效的并且更简单：
```c
rte_eal_remote_launch(lcore_hello, NULL, lcore_id);
```

有关 rte_eal_mp_remote_launch() 函数的详细信息，请参阅 *DPDK API Reference*。
