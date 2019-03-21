# Discuz搜索模块解析

## 技术点

### runhooks\(\)

discuz在多个主要的入口文件中都调用了这个函数，在search.php中也调用了，碰巧发现了就不能放过。主要是hook这个词引起了我的兴趣，想了解下函数里到底做了什么。该函数内的主要代码如下，可以看出该函数是在执行某些插件相关的功能。

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

## 功能解析

### 搜索帖子

discuz搜索的入口位于主目录的search.php文件，功能模块位于source/module/search中，由于其他类型的搜索非重点，所以此处只选择search\_forum.php和search\_user.php进行分析。forum搜索即帖子标题和内容的搜索，user即用户名的搜索。

搜索的相关设置可以在admin页面-&gt;全局-&gt;搜索设置中找到。如果要支持sphinx搜索，在该页面中配置sphinx服务器。搜索帖子的主要逻辑在search\_forum.php文件中，搜索有两种类型，一是标题搜索，二是全文搜索。在启用spinx的情况下，如果搜索类型为全文搜索，discuz才会用sphinx进行搜索，否则discuz会进行数据库的模糊匹配搜索，相关代码如下。模糊匹配搜索在帖子比较多的情况下执行会很耗时，需要多加注意。

```php
if($srchtype == 'fulltext' && $_G['setting']['sphinxon']) {
    ...
} else {
    ...
    $srcharr = $srchtype == 'fulltext' ? searchkey($keyword, "(p.message LIKE '%{text}%' OR p.subject LIKE '%{text}%')", true) : searchkey($keyword,"t.subject LIKE '%{text}%'", true);
}
```

检索完成后，discuz会将搜索结果保存到common\_searchindex表中作为一次搜索记录，ids那一列即为匹配帖子的tid列表：

```php
$searchid = C::t('common_searchindex')->insert(array(
				'srchmod' => $srchmod,
				'keywords' => $keywords,
				'searchstring' => $searchstring,
				'useip' => $_G['clientip'],
				'uid' => $_G['uid'],
				'dateline' => $_G['timestamp'],
				'expiration' => $expiration,
				'num' => $num,
				'ids' => $ids
			), true);
```

根据ids获取到帖子的信息后，对标题进行高亮操作，再返回给用户：

```php
foreach(C::t('forum_thread')->fetch_all_by_tid_fid_displayorder(explode(',',$index['ids']), null, 0, $orderby, $start_limit, $_G['tpp'], '>=', $ascdesc) as $thread) {
    $thread['subject'] = bat_highlight($thread['subject'], $keyword);
    $thread['realtid'] = $thread['isgroup'] == 1 ? $thread['closed'] : $thread['tid'];
    $threadlist[$thread['tid']] = procthread($thread, 'dt');
    $posttables[$thread['posttableid']][] = $thread['tid'];
}
```

### 搜索用户

如果进行用户搜索，最终会跳转到home.php?mod=spacecp&ac=search页面，搜索的主要逻辑在source/include/spacecp/spacecp\_search.php中。搜索用户的过程主要是将各种搜索条件组合成sql语句，需要注意的是用户名有可能进行模糊搜索，需要注意性能，相关代码如下：

```php
foreach (array('uid','username','videophotostatus','avatarstatus') as $value) {
	if($_GET[$value]) {
		if($value == 'username' && empty($_GET['precision'])) {
			$_GET[$value] = stripsearchkey($_GET[$value]);
			$wherearr[] = 's.'.DB::field($value, '%'.$_GET[$value].'%', 'like');
		} else {
			$wherearr[] = 's.'.DB::field($value, $_GET[$value]);
		}
	}
}
```

