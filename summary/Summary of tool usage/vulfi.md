#### 项目地址：

https://github.com/Accenture/VulFi

VulFi（漏洞查找器）工具是 IDA Pro 的一个插件，可用于协助在二进制文件中寻找错误。它的主要目标是提供一个单一视图，其中包含对最有趣的功能（例如`strcpy`、`sprintf`、`system`等）的所有交叉引用。对于可以使用 Hexrays 反编译器的情况，它会尝试排除对这些函数的调用，这些函数从漏洞研究的角度来看并不有趣（想想类似`strcpy(dst,"Hello World!")`）。如果没有反编译器，规则会简单得多（不依赖于架构），因此只排除最明显的情况。

#### 安装使用

将`vulfi.py`、`vulfi_prototypes.json`和`vulfi_rules.json`文件放在 IDA 插件文件夹（也可将自定义的规则文件放入）

顶部栏菜单中选择`Search`>选项。`VulFi`这将启动新的扫描，或者它将读取存储在`idb`/`i64`文件中的先前结果。每当保存数据库时，数据都会自动保存。

也可在某个函数中选择VulFi扫描

#### 设置自定义规则

支持我们加载包含了多种自定义规则的自定义文件，下面给出的是自定义规则文件的数据结构：

```
[   // An array of rules

    {

        "name": "RULE NAME", // The name of the rule

        "alt_names":[

            "function_name_to_look_for" // List of all function names that should be matched against the conditions defined in this rule

        ],

        "wrappers":true,    // Look for wrappers of the above functions as well (note that the wrapped function has to also match the rule)

        "mark_if":{

            "High":"True",  // If evaluates to True, mark with priority High (see Rules below)

            "Medium":"False", // If evaluates to True, mark with priority Medium (see Rules below)

            "Low": "False" // If evaluates to True, mark with priority Low (see Rules below)

        }

    }

]
```



#### 规则设置

##### 可用变量

> 1、param[<index>]：用访问函数调用的参数（index起始为0）；
>
> 2、function_call：用于访问函数调用事件；
>
> 3、param_count：获取传递给函数的参数数量；

##### 可用函数

> 1、判断参数是否为常量：param[<index>].is_constant()
>
> 2、获取参数的数字值：param[<index>].number_value()
>
> 3、获取参数的字符串值：param[<index>].string_value()
>
> 4、调用后参数是否被设置为NULL：param[<index>].set_to_null_after_call()
>
> 5、函数的返回值是否经过检测：function_call.return_value_checked(<constant_to_check>)
