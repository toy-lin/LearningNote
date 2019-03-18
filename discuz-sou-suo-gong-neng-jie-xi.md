# Discuz搜索功能解析

## 知识点

## 功能解析

discuz的搜索功能模块位于source/module/search中，可搜索的类型比较多，我也不太清楚各个类型是什么意思，但是代码实现都差不多，所以此处只选择search\_forum.php和search\_user.php进行分析。forum搜索即帖子标题和内容的搜索，user即用户名的搜索。

搜索的相关设置可以在admin页面-&gt;全局-&gt;搜索设置中找到。如果要支持sphinx搜索，在该页面中配置sphinx服务器。搜索帖子的主要逻辑在search\_forum.php文件中，搜索有两种类型，一是标题搜索，二是全文搜索。在启用spinx的情况下，如果搜索类型为全文搜索，discuz才会用sphinx进行搜索，否则discuz会进行数据库的模糊匹配搜索，相关代码如下。模糊匹配搜索在帖子比较多的情况下执行会很耗时，需要多加注意。

```php
if($srchtype == 'fulltext' && $_G['setting']['sphinxon']) {
    ...
} else {
    ...
    $srcharr = $srchtype == 'fulltext' ? searchkey($keyword, "(p.message LIKE '%{text}%' OR p.subject LIKE '%{text}%')", true) : searchkey($keyword,"t.subject LIKE '%{text}%'", true);
}
```



