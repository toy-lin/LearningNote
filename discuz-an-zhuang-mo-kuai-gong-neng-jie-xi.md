# Discuz安装模块功能解析

## 技术点

1. error\_reporting\(\)。设置php要显示错误报告的错误等级，如果要显示parse\_error，需要在php.ini配置文件中修改display\_errors = on。
2. set\_magic\_quotes\_runtime\(boolean\)。从这个链接\([http://php.net/manual/zh/function.set-magic-quotes-runtime.php](http://php.net/manual/zh/function.set-magic-quotes-runtime.php)\)的例子可知，此函数是控制php读取字符串时是否对斜杠\进行解析。参数为false则进行解析。
3. set\_time\_limit\(\)。设置php最大运行时间，以秒为单位。
4. getgpc\(key\)。这个函数在discuz中有多处实现，其功能是获取GET/POST/COOKIE/REQUEST请求中的参数。
5. 以下代码可对用户浏览的网址进行重定向，但是在调用下面的代码前不应该对用户输出任何内容，否则重定向将失效： header\("Location: index.php?step=3&install\_ucenter=yes"\); die; 如果要进行永久重定向，则应写为： header\("Location: index.php?step=3&install\_ucenter=yes", true , 301\);  die;
6. 变量有值则输出变量，php里很巧妙的写法：$comment && print\(''.$comment\);

## 功能

discuz安装功能在install文件夹内，整个安装过程都在install.php中，并且通过判断ROOT/data/install.lock文件是否来判断安装是否已完成。install.lock文件由下面的ext\_info步骤创建，如果install.lock文件不存在，用户访问install/index.php时，会执行安装操作，如果install.lock文件已存在，用户访问install/index.php则会报错：install\_locked。

discuz通过method或step字段的值判断当前的安装步骤，分别有以下几个步骤：

### show\_license

用户直接访问install/index.php时，由于不带参数，step字段值默认为0，会进入show\_license显示协议的页面。discuz以表格的形式显示许可协议，用户点击同意后，表单提交的step字段值为1，进入env\_check步骤。

### env\_check

discuz在这一步中会执行以下检查：

* php执行环境，检查是否存在以下函数：'mysqli\_connect', 'gethostbyname', 'file\_get\_contents', 'xml\_parser\_create'。
* 操作系统运行环境：操作系统类型，PHP版本，附件大小限制，libgd库版本\(图形相关\)，磁盘空间。
* 各种目录文件读写权限\(Windows系统无读写权限问题\)。

最后通过step加1进入app\_reg步骤。

### app\_reg

设置运行环境。如果已有UCenter，则需输入已有的UCenter的相关信息及要建立的discuz站点的信息。否则discuz将进行完整的discuz + UCenter配置及数据库安装。如果你不知道UCenter是什么的话，选全新安装就对了。

如果选择已有UCenter Server并输入UCenter相关信息，discuz会进行验证。通过向配置的地址请求相关配置信息以验证UCenter Server是否正常运行，如果验证成功，则自动保存用户输入的UCenter信息到/config/config\_ucenter.php文件中。

下一步会进入到db\_init步骤。

比较无语的是这里居然把代码写到了install\_lang.php文件里，害我半天都没找到隐藏显示UCenter配置的代码在哪里。

### db\_init

此步骤配置数据库相关信息及UCenter站长管理员的信息。

discuz将会执行install/data文件夹中的install.sql文件中的sql语句创建相关数据库表，并且执行install\_data.sql文件中的sql语句创建基础数据。

如果用户在上一步选择的是全新安装，则discuz还会执行/uc\_server/install/uc.sql中的sql在同一个数据库中创建UCenter需要用到的数据库表并输入相关初始化数据。然后保存UCenter相关信息到/config/config\_ucenter.php文件中。

UCenter数据库表初始化数据包括：

1. 向ucenter\_applications表中插入站点相关数据--站点名、url、ip等。
2. 向ucenter\_members表中插入第一个用户\(创始人\)用户名、密码等信息。
3. 在ucenter\_admins表中插入数据，设置创始人的UCenter操作权限。

### ext\_info

到达这个步骤表示discuz已经安装完成，会显示安装完成的信息，并创建/data/install.lock文件表示安装已完成。



