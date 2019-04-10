# PHP各类型变量与布尔值之间的转换

经常看到PHP代码中拿各种类型的变量来做布尔值用，故写此文章理顺各种类型变量和布尔值之间的转化规律，总结如下：

1. 对于数值类型，值为0等于布尔值false，其他值等于true。
2. 对于字符串类型，字符串为空，即长度为0时，等于布尔值false，其他情况则等于true。
3. 对于引用类型，值为null则等于布尔值false，否则等于true。
4. 对于数组类型，当数组为空，即大小为0时，等于布尔值false，其他情况则等于true。
5. 数组类型用下标或者key取值时，若对应的下标或者key不存在，都会取得值null，其布尔值参考第3条。

实验过程如下：

```php
$i = 0;
if ($i) {
    echo "i is not 0</br>";
} else {
    echo "i is 0</br>";
}

$i = -1;
if ($i) {
    echo "i is not 0</br>";
} else {
    echo "i is 0</br>";
}

$i = 1;
if ($i) {
    echo "i is not 0</br>";
} else {
    echo "i is 0</br>";
}

$v = null;
if ($v) {
    echo "v is not null</br>";
} else {
    echo "v is null</br>";
}

$v = new Data_Live_Configure();
if ($v) {
    echo "v is not null</br>";
} else {
    echo "v is null</br>";
}

$v = "";
if ($v) {
    echo "v is not empty string</br>";
} else {
    echo "v is empty string</br>";
}

$v = " ";
if ($v) {
    echo "v is not empty string</br>";
} else {
    echo "v is empty string</br>";
}

$v = "false";
if ($v) {
    echo "v is not empty string</br>";
} else {
    echo "v is empty string</br>";
}

$arr = array();
if ($arr) {
    echo "arr is not empty</br>";
} else {
    echo "arr is empty</br>";
}

$arr[1] = "false";
if ($arr) {
    echo "arr is not empty</br>";
} else {
    echo "arr is empty</br>";
}

if ($arr[0]) {
    echo "arr has element at index 0</br>";
} else {
    echo "arr do not has element at index 0</br>";
}

if ($arr['hello']) {
    echo "arr has value of key 'hello'</br>";
} else {
    echo "arr do not has value of key 'hello'</br>";
}
```

输出结果为：

```markup
i is 0
i is not 0
i is not 0
v is null
v is not null
v is empty string
v is not empty string
v is not empty string
arr is empty
arr is not empty
arr do not has element at index 0
arr do not has value of key 'hello'
```

