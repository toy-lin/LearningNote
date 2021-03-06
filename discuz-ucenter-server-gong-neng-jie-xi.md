# Discuz验证码功能解析

discuz验证码功能主要在\uc\_server\lib\seccode.class.php或/source/class/class\_seccode.php文件中实现，这两个文件的代码基本一致，区别是一个运行于ucenter server，一个运行于discuz。下面的分析是针对ucenter server上的验证码功能。

discuz支持的验证码类型有图形、flash、音频和位图。在服务器安装了ming库（动态生成flash动画）的情况下，discuz默认使用flash验证码。如果未安装ming，已安装GD库的情况下，discuz会使用图片验证码，否则使用位图验证码。音频验证码则需要在代码中手动配置。

### 位图验证码

discuz代码中已事先保存了各种字母和数字的位图编码：

```php
$numbers = array
			(
			'B' => array('00','fc','66','66','66','7c','66','66','fc','00'),
			'C' => array('00','38','64','c0','c0','c0','c4','64','3c','00'),
			'E' => array('00','fe','62','62','68','78','6a','62','fe','00'),
			'F' => array('00','f8','60','60','68','78','6a','62','fe','00'),
			'G' => array('00','78','cc','cc','de','c0','c4','c4','7c','00'),
			'H' => array('00','e7','66','66','66','7e','66','66','e7','00'),
			'J' => array('00','f8','cc','cc','cc','0c','0c','0c','7f','00'),
			'K' => array('00','f3','66','66','7c','78','6c','66','f7','00'),
			'M' => array('00','f7','63','6b','6b','77','77','77','e3','00'),
			'P' => array('00','f8','60','60','7c','66','66','66','fc','00'),
			'Q' => array('00','78','cc','cc','cc','cc','cc','cc','78','00'),
			'R' => array('00','f3','66','6c','7c','66','66','66','fc','00'),
			'T' => array('00','78','30','30','30','30','b4','b4','fc','00'),
			'V' => array('00','1c','1c','36','36','36','63','63','f7','00'),
			'W' => array('00','36','36','36','77','7f','6b','63','f7','00'),
			'X' => array('00','f7','66','3c','18','18','3c','66','ef','00'),
			'Y' => array('00','7e','18','18','18','3c','24','66','ef','00'),
			'2' => array('fc','c0','60','30','18','0c','cc','cc','78','00'),
			'3' => array('78','8c','0c','0c','38','0c','0c','8c','78','00'),
			'4' => array('00','3e','0c','fe','4c','6c','2c','3c','1c','1c'),
			'6' => array('78','cc','cc','cc','ec','d8','c0','60','3c','00'),
			'7' => array('30','30','38','18','18','18','1c','8c','fc','00'),
			'8' => array('78','cc','cc','cc','78','cc','cc','cc','78','00'),
			'9' => array('f0','18','0c','6c','dc','cc','cc','cc','78','00')
			);
```

其中，每个字符对应的列表的每一个元素都可以转化为一个8位的二进制数字，其中1表示白点，0表示黑点。上述代码中每个字符都对应10个元素，故discuz用一个10x8的位图表示一个字符，例如B对应的列表，转化为二进制后得到：

00000000  
11111100  
01100110  
01100110  
01100110  
01111100  
01100110  
01100110  
11111100  
00000000

把0看成黑点，1看成白点，你就会得到一个黑底白字的字母B。discuz随机生成的4个字符进行组合，转换成位图，插入一些噪点，再输出给前端，验证码功能就完成了。

### 音频验证码

音频验证码的实现比较简单，如下面的代码，discuz将事先录制好的对应到每个字符的mp3文件进行连续播放：

```php
header('Content-type: audio/mpeg');
for($i = 0;$i <= 3; $i++) {
	readfile($this->datapath.'sound/'.strtolower($this->code[$i]).'.mp3');
}
```

### flash验证码

flash验证码的实现和位图验证码的实现类似，都是利用已有的编码，对随机生成的字符进行对应组合，再输出给前端：

flash验证码编码片段：

```php
case 'B':$str .= 'moveTo('.$x_1.', '.$y_0.');lineTo('.$x_0.', '.$y_0.');lineTo('.$x_0.', '.$y_2.');lineTo('.$x_1.', '.$y_2.');lineTo('.$x_2.', '.$y_1_5.');lineTo('.$x_1.', '.$y_1.');lineTo('.$x_2.', '.$y_0_5.');lineTo('.$x_1.', '.$y_0.');moveTo('.$x_0.', '.$y_1.');lineTo('.$x_1.', '.$y_1.');';break;
case 'C':$str .= 'moveTo('.$x_2.', '.$y_0.');lineTo('.$x_0.', '.$y_0.');lineTo('.$x_0.', '.$y_2.');lineTo('.$x_2.', '.$y_2.');';break;
case 'E':$str .= 'moveTo('.$x_2.', '.$y_0.');lineTo('.$x_0.', '.$y_0.');lineTo('.$x_0.', '.$y_2.');lineTo('.$x_2.', '.$y_2.');moveTo('.$x_0.', '.$y_1.');lineTo('.$x_1.', '.$y_1.');';break;
case 'F':$str .= 'moveTo('.$x_2.', '.$y_0.');lineTo('.$x_0.', '.$y_0.');lineTo('.$x_0.', '.$y_2.');moveTo('.$x_0.', '.$y_1.');lineTo('.$x_1.', '.$y_1.');';break;
case 'G':$str .= 'moveTo('.$x_2.', '.$y_0.');lineTo('.$x_0.', '.$y_0.');lineTo('.$x_0.', '.$y_2.');lineTo('.$x_2.', '.$y_2.');lineTo('.$x_2.', '.$y_1.');lineTo('.$x_1.', '.$y_1.');';break;
case 'H':$str .= 'moveTo('.$x_0.', '.$y_0.');lineTo('.$x_0.', '.$y_2.');moveTo('.$x_2.', '.$y_0.');lineTo('.$x_2.', '.$y_2.');moveTo('.$x_0.', '.$y_1.');lineTo('.$x_2.', '.$y_1.');';break;
```

### 图形验证码

图形验证码支持gif动图、jpg和png。主要是利用GD库的功能进行图片合成，字体取自ttf文件，背景取自static/image/seccode/background/文件夹中的图片，由于实现过程有点繁杂，这里就不深入了。





