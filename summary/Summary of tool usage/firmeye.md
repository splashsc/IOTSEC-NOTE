#### 项目地址：

https://github.com/VulnTotal-Team/firmeye

Firmeye 是一个 IDA 的一个重要功能参数回溯辅助功能。调用敏感敏感的地方非常多，人工分析可以帮助提高功能的位置，通过该调用的安全性，提高效率。

- 漏洞类型支持规范：溢出、命令执行、格式化字符串

#### 使用：

将项目中`firmeye` 和 `firmeye.py` 复制到 IDA Pro 插件目录下，安装依赖：pip install -r requirements.txt



Ctrl+F1 查看插件使用帮助。热键：

- `Ctrl+Shift+s`：主菜单
- `Ctrl+Shift+d`：启动/禁用调试钩子
- `Ctrl+Shift+c`：扫描代码模式（TODO）
- `Ctrl+Shift+x`：逆向辅助工具
- `Ctrl+Shift+q`：功能测试

敏感函数被分为 5 类：printf、strcpy、memcpy、scanf、system。分别对应各自的漏洞类型和检测规则。



可以批量对可以地址下断点



#### 总结：

我个人用的比较多的就是静态匹配危险函数的功能，其他下断点的功能没怎么用过，总的来说很方便去寻找危险函数，省去了很多时间，一目了然。

