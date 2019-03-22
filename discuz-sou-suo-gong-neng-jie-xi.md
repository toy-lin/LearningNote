# Discuz搜索模块解析

## 功能解析

搜索模块的子模块有8个，相关信息如下：

| 子模块 | 功能 |
| :--- | :--- |
| album | 搜索相册。 |
| blog | 搜索日志。 |
| collection | 搜索收藏的帖子。 |
| forum | 搜索帖子。 |
| group | 搜索群组或群组帖子。 |
| my | 貌似已废弃。 |
| portal | 搜索门户帖子。 |
| user | 搜索用户。 |

discuz的论坛搜索入口可以在/admin.php?action=setting&operation=search页面中配置。搜索项目为搜索论坛的项是直接控制论坛主页的搜索入口的，如果关闭了，论坛就没有搜索入口。各个子模块的功能实现也大同小异，下面我们分析一下主要的实现：

### forum--搜索帖子

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

### user--搜索用户

如果进行用户搜索，最终会跳转到home.php?mod=spacecp&ac=search页面。我想discuz这么做的主要原因是用户信息是存于UCenter的，所以理论上要通过UCenter进行搜索，但是UCenter又是home模块的范围，所以搜索用户的操作就交给home模块了。搜索的主要逻辑在source/include/spacecp/spacecp\_search.php中。搜索用户的过程主要是将各种搜索条件组合成sql语句，需要注意的是用户名有可能进行模糊搜索，需要注意性能，相关代码如下：

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

### my--未知作用

这个子模块最终会跳转到网址search.discuz.qq.com，由于这个网址现在已经无法访问了，不知道它的作用是什么，推测是用第三方引擎对本站进行全站的搜索。我当前用的discuz版本是x3.2。

### 其他模块

其他模块的实现和forum模块的实现差不多，但是都没有用到sphinx，都是对数据库表进行模糊搜索。而且每一次搜索都会记录到common\_searchindex表中。

## end

到这里搜索模块的功能就大致分析完毕了，由于有些功能找不到入口，比如相册、日志、门户和群组等，并没有实际使用过，都只是对代码做了分析，如果有说错的地方，望指正。



