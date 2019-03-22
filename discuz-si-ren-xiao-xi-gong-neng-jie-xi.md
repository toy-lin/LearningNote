# Discuz论坛架构功能解析

## 知识点

1.init\_get和init\_set函数：获取和修改php运行环境的变量值，即存于php.ini中的值。

2.自动加载函数：discuz的class\_core.php中有下面的代码，其作用是当我们在代码中引用不存在的类时，自动加载对应的类。这时\_\_autoload就会被调用，并且类名会被作为参数传送过去。discuz的自动加载函数实现了自动加载source/class/目录下的类，避免了引用各种类文件的繁杂操作。

```php
if(function_exists('spl_autoload_register')) {
	spl_autoload_register(array('core', 'autoload'));
} else {
	function __autoload($class) {
		return core::autoload($class);
	}
}
```

## 结构解析

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

### class\_core

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

然后定义了discuz代码中经常用到的C类，其父类core实现了所有的功能，存放着一些静态变量和提供一些静态方法，扩展出一个子类C应该只是方便调用。

```php
class C extends core {}
```

core类主要管理4个静态变量：

1. $\_tables：此变量存放了一些discuz\_model和discuz\_table类对象，用于数据库操作。获取discuz\_model/discuz\_table类对象的方法是m和t，即C::m\($name\)或C::t\($name\)。已经生成的对象会保存在$\_tables中，避免多次创建对象。
2. $\_imports：此变量存放着已导入类的列表，用于前面提到的自动加载函数中，避免类被多次导入。
3. $\_memory：这个是缓存器，缓存类型可能是redis、memcache或者其他。提供了一些基础的缓存管理功能。
4. $\_app：discuz核心应用类对象，完成了各种初始化操作，保证discuz正确运行。

core.php中还创建了discuz\_application类对象赋值给$\_app，discuz\_application类做了一些核心的初始化操作：

1. \_init\_env：初始化环境变量，检查内存限制、检测爬虫和初始化全局变量$\_G等。
2. \_init\_config：初始化论坛设置，检测论坛是否安装完整、初始化authkey和初始化cookie目录等。
3. \_init\_input：检查全局变量、过滤请求参数、合并GET/POST请求参数和设置盐值等。盐值用于生成authkey，用于验证用户身份。
4. \_init\_output：初始化输出缓存和输出header等。

$discuz-&gt;init\(\)的调用又做了以下一些操作：

1. \_init\_db：初始化数据库连接。
2. \_init\_setting：初始化设置，并加载设置、风格等缓存。
3. \_init\_user：验证存于cookie的auth字段中的用户名密码或初始化访客身份信息及设置、检查购买的用户组是否过期和更新新消息数量等和用户相关的操作。
4. \_init\_session：初始化用户session。
5. \_init\_mobile：检查用户是否从移动端访问并决定是否跳转到移动端页面。
6. \_init\_cron：检查并执行定时任务。
7. \_init\_misc：执行xss检查、访问权限检查和登录超时检查等。

### mod

字段mod确定了用户访问各大模块的哪个子功能。各个入口都会检查mod的合理性，如home.php中：

```php
$mod = getgpc('mod');
if(!in_array($mod, array('space', 'spacecp', 'misc', 'magic', 'editor', 'invite', 'task', 'medal', 'rss', 'follow'))) {
	$mod = 'space';
	$_GET['do'] = 'home';
}
```

### runhooks

每个入口都执行了一次runhooks函数，该函数内的主要代码如下，可以看出该函数是在执行某些插件相关的功能。

```php
...
if($_G['setting']['plugins']['func'][HOOKTYPE]['common']) {
	hookscript('common', 'global', 'funcs', array(), 'common');
}
...
```

再看看hookscript函数的主要代码就比较明朗了，hookscript主要执行了在全局变量$\_G中已设置的插件，并将返回值记录在全局变量$\_G中。

```php
	...
	foreach((array)$_G['setting'][HOOKTYPE][$hscript][$script]['module'] as $identifier => $include) {
		if($_G['pluginrunlist'] && !in_array($identifier, $_G['pluginrunlist'])) {
			continue;
		}
		$hooksadminid[$identifier] = !$_G['setting'][HOOKTYPE][$hscript][$script]['adminid'][$identifier] || ($_G['setting'][HOOKTYPE][$hscript][$script]['adminid'][$identifier] && $_G['adminid'] > 0 && $_G['setting']['hookscript'][$hscript][$script]['adminid'][$identifier] >= $_G['adminid']);
		if($hooksadminid[$identifier]) {
			//引用插件类
			@include_once DISCUZ_ROOT.'./source/plugin/'.$include.'.class.php';
		}
	}
	...
	//执行插件类中的方法
	$return = $pluginclasses[$classkey]->$hookfunc[1]($param);
	...
	//记录执行方法的返回值
	foreach($return as $k => $v) {
		$_G['setting']['pluginhooks'][$hookkey][$k] .= $v;
	}
```

不过哪些插件功能会在程序入口处被执行呢，我追溯了一下$\_G\['setting'\]\['plugins'\]\['func'\]\[HOOKTYPE\]\['common'\]的来源，确实是来自插件类，但是由于整个过程比较繁杂，我就不展开分析了。下面的代码是我在cache\_setting.php中看到的，配合上下文代码看，大概可以证明runhooks中hook方法的来源，以后有空再专门分析discuz插件：

```php
...
$data[$k][$hscript][$curscript]['funcs'][$funcname][] = array('displayorder' => $module['displayorder'], 'func' => array($plugin['identifier'], $funcname));
...
```

### end

discuz模块入口的功能大概就前面写的这些，后面我们继续分析各大功能模块的功能。

