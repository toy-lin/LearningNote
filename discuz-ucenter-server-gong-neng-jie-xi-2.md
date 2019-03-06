# Discuz UCenter Server功能解析

## 技术点

#### 1.extract函数

将列表中的存在映射关系的key创建为变量，value作为变量值。具体可看：[https://secure.php.net/manual/en/function.extract.php](https://secure.php.net/manual/en/function.extract.php)。

#### 2.explode函数

分割字符串，相当于Java或者Python中的split。

#### 3.sid包含用户名信息

登录UCenter之后，页面的链接显示为：/uc\_server/admin.php?sid=c935ichRGUiM9%2BuTj3o%2BE4h9towLwM9iSFMM0%2BV0GB7hDtoQCCOilLv99TMU7IoI5yfrccjEZ54C0w，其中sid包含了用户名和check码，check码与浏览器UserAgent相关，解密代码为：

```php
	function sid_decode($sid) {
		$ip = $this->onlineip;
		$agent = $_SERVER['HTTP_USER_AGENT'];
		$authkey = md5($ip.$agent.UC_KEY);
		$s = $this->authcode(rawurldecode($sid), 'DECODE', $authkey, 1800);
		if(empty($s)) {
			return FALSE;
		}
		@list($username, $check) = explode("\t", $s);
		if($check == substr(md5($ip.$agent), 0, 8)) {
			return $username;
		} else {
			return FALSE;
		}
	}
```

#### 4.eval函数

对字符串内的变量进行解析，例如：

```php
<?php
$string = 'cup';
$name = 'coffee';
$str = 'This is a $string with my $name in it.';
echo $str. "\n";
eval("\$str = \"$str\";");
echo $str. "\n";
?>
```

输出为：

This is a $string with my $name in it. 

This is a cup with my coffee in it.

#### 5.register\_shutdown\_function函数

```php
<?php
function shutdown()
{
    // This is our shutdown function, in 
    // here we can do any last operations
    // before the script is complete.

    echo 'Script executed with success', PHP_EOL;
}

register_shutdown_function('shutdown');
?>
```



## 功能

discuz ucenter server采用MVC\(即Model、View、Control\)的代码架构，



