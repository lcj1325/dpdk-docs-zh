# 6. Argparse库

argparse 库提供参数解析功能，该库可以轻松编写用户友好的命令行程序。

## 6.1.特性和功能

- 支持解析可选参数（可以带无值、必需值和可选值）。
- 支持解析位置参数（必须带有必需值）。
- 支持自动生成使用信息。
- 当提供无效参数时支持问题错误。
- 支持两种方式解析参数:
    1. autosave：用于解析已知值类型；
    2. 回调：将调用用户回调来解析。

## 6.2.使用指南

下面的代码演示了如何使用：

```c
static int
argparse_user_callback(uint32_t index, const char *value, void *opaque)
{
   if (index == 1) {
      /* process "--ddd" argument, because it is configured as no-value,
       * the parameter 'value' is NULL.
       */
      ...
   } else if (index == 2) {
      /* process "--eee" argument, because it is configured as
       * required-value, the parameter 'value' must not be NULL.
       */
      ...
   } else if (index == 3) {
      /* process "--fff" argument, because it is configured as
       * optional-value, the parameter 'value' maybe NULL or not NULL,
       * depend on input.
       */
      ...
   } else if (index == 300) {
      /* process "ppp" argument, because it's a positional argument,
       * the parameter 'value' must not be NULL.
       */
      ...
   } else {
      return -EINVAL;
   }
}

static int aaa_val, bbb_val, ccc_val, ooo_val;

static struct rte_argparse obj = {
   .prog_name = "test-demo",
   .usage = "[EAL options] -- [optional parameters] [positional parameters]",
   .descriptor = NULL,
   .epilog = NULL,
   .exit_on_error = true,
   .callback = argparse_user_callback,
   .args = {
      { "--aaa", "-a", "aaa argument", &aaa_val, (void *)100, RTE_ARGPARSE_ARG_NO_VALUE       | RTE_ARGPARSE_ARG_VALUE_INT },
      { "--bbb", "-b", "bbb argument", &bbb_val, NULL,        RTE_ARGPARSE_ARG_REQUIRED_VALUE | RTE_ARGPARSE_ARG_VALUE_INT },
      { "--ccc", "-c", "ccc argument", &ccc_val, (void *)200, RTE_ARGPARSE_ARG_OPTIONAL_VALUE | RTE_ARGPARSE_ARG_VALUE_INT },
      { "--ddd", "-d", "ddd argument", NULL,     (void *)1,   RTE_ARGPARSE_ARG_NO_VALUE       },
      { "--eee", "-e", "eee argument", NULL,     (void *)2,   RTE_ARGPARSE_ARG_REQUIRED_VALUE },
      { "--fff", "-f", "fff argument", NULL,     (void *)3,   RTE_ARGPARSE_ARG_OPTIONAL_VALUE },
      { "ooo",   NULL, "ooo argument", &ooo_val, NULL,        RTE_ARGPARSE_ARG_REQUIRED_VALUE | RTE_ARGPARSE_ARG_VALUE_INT },
      { "ppp",   NULL, "ppp argument", NULL,     (void *)300, RTE_ARGPARSE_ARG_REQUIRED_VALUE },
   },
};

int
main(int argc, char **argv)
{
   ...
   ret = rte_argparse_parse(&obj, argc, argv);
   ...
}
```

在此示例中，以连字符 (-) 开头的参数是可选参数（它们是“–aaa”/“–bbb”/“–ccc”/“–ddd”/“–eee”/“–fff” ）；不以连字符 (-) 开头的参数是位置参数（它们是“ooo”/“ppp”）。

每个参数都必须设置是否带有值（`RTE_ARGPARSE_ARG_NO_VALUE`、`RTE_ARGPARSE_ARG_REQUIRED_VALUE` 和 `RTE_ARGPARSE_ARG_OPTIONAL_VALUE` 之一）。

> note:
> 位置参数必须设置 `RTE_ARGPARSE_ARG_REQUIRED_VALUE`。

### 6.2.1. 用户输入要求

对于不带值的可选参数，支持以下模式（以上面的“--aaa”为例）：

- 单一模式：“-aaa”或“-a”。

对于采用 required-value 的可选参数，支持以下两种模式（以上面的“-bbb”为例）：

- kv模式：“-bbb=1234”或“-b=1234”。
- 分割模式：“-bbb 1234”或“-b 1234”。

对于必须采用必需值的位置参数，它们的值按照定义的顺序进行解析。

> note:
> 不支持紧凑模式。以上面“-a”和“-d”为例，不支持“-ad”输入。

### 6.2.2.自动保存方式解析

已知值类型的参数（例如 `RTE_ARGPARSE_ARG_VALUE_INT`）可以使用这种自动保存方式进行解析，其结果将保存在 `val_saver` 字段中。

上面的例子中，参数“-aaa”/“-bbb”/“-ccc”和“ooo”都使用了这种方式，解析如下：

- 对于参数“–aaa”，它被配置为无值，因此 aaa_val 将被设置为 val_set 字段，在上例中为 100。
- 对于参数“–bbb”，它被配置为必需值，因此 bbb_val 将设置为用户输入的值（例如，输入“–bbb 1234”时将设置为 1234）。
- 对于参数“--ccc”，它被配置为可选值，如果用户只输入“--ccc”，则 ccc_val 将被设置为 val_set 字段，在上例中为 200；如果用户输入“–ccc=123”，则 ccc_val 将设置为 123。
- 对于参数“ooo”，它是位置参数，ooo_val 将被设置为用户输入的值。

### 6.2.3.通过回调方式解析

它还可以选择使用回调来解析，只需为参数定义一个唯一索引并将val_save字段设置为NULL也是零值类型。

在上面的例子中，参数“-ddd”/“-eee”/“-fff”和“ppp”都使用这种方式。

### 6.2.4.多次争论

如果想要支持多次输入相同参数的能力，则应在标志字段中标记 `RTE_ARGPARSE_ARG_SUPPORT_MULTI`。例如：

```c
{ "--xyz", "-x", "xyz argument", NULL, (void *)10, RTE_ARGPARSE_ARG_REQUIRED_VALUE | RTE_ARGPARSE_ARG_SUPPORT_MULTI },
```

那么用户输入可以包含多个“-xyz”参数。

> note:
> 多次参数仅支持可选参数，并且必须通过回调方式解析。