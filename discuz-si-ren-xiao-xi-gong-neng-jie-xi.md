# Discuz论坛架构功能解析

Discuz结构庞大，功能繁杂，细看发现它整体大概分为以下几个模块：

| 模块 | 功能 |
| :--- | :--- |
| home | 查看用户信息、私信、关注、道具、rss等功能 |
| forum | 跟论坛、版块、帖子相关的功能 |
| member | 用户登录相关的功能 |
| plugin | 插件相关的功能 |
| group | 用户群组相关的功能 |
| portal | 模块相关的功能 |
| search | 搜索相关的功能 |
| admin | 论坛管理模块 |
| misc | 其他功能 |

模块和模块之间有比较明显的分割界线，每个模块都和代码主目录下的代码文件一一对应。这些代码文件都是入口（后面简称入口），主要功能逻辑的代码实现位于source/module/目录下对应的目录中，如图：

![](.gitbook/assets/image%20%281%29.png)

![](.gitbook/assets/image%20%282%29.png)

各大模块的入口主要做的是一些初始化工作以及将请求转发到对应的功能模块中。下面我们分析下入口的功能。

所有的入口基本上都首先会执行下面的语句：

```php
require './source/class/class_core.php';
...
$discuz = C::app();
...
$discuz->init();
```

class\_core.php文件中首先设置了错误报告等级和异常处理机制，并且实现了自动加载函数（关于自动加载函数，我觉得[这篇文章](https://www.cnblogs.com/CpNice/p/4119925.html)写得不错），代码如下：

```php
error_reporting(E_ALL);
...
set_exception_handler(array('core', 'handleException'));

if(DISCUZ_CORE_DEBUG) {
	set_error_handler(array('core', 'handleError'));
	register_shutdown_function(array('core', 'handleShutdown'));
}

if(function_exists('spl_autoload_register')) {
	spl_autoload_register(array('core', 'autoload'));
} else {
	function __autoload($class) {
		return core::autoload($class);
	}
}
```







