# Discuz home模块解析

## 功能解析

home模块很庞大，包含了discuz论坛和用户信息相关的几乎所有功能，比如动态、消息、勋章、道具等等，有的功能还分为很多小功能，下面列出home模块基本的组成：

| 子模块 | 功能 | 说明 |
| :--- | :--- | :--- |
| space | 活动、相册、日志、收藏、好友、分享、悬赏等等等等 | 用户个人空间相关的功能 |
| spacecp |  | 用户个人空间相关功能的改动，如添加好友、发日志、收藏帖子等等 |
| misc |  |  |
| magic |  | 道具相关 |
| editor |  | 编辑器 |
| invite |  | 邀请相关 |
| task |  | 任务相关 |
| medal |  | 勋章相关 |
| rss |  | 网站信息 |
| follow |  | 广播相关 |

下面我们分别分析home模块的各个子模块：

### rss模块

访问连接:home.php?mod=rss

此处的rss应该是指Really Simple Syndication\(简易信息聚合\)，输出网站的信息和对应用户的个人动态信息（blog）：

```php
//输出网站信息
echo 	"<?xml version=\"1.0\" encoding=\"".$charset."\"?>\n".
	"<rss version=\"2.0\">\n".
	"  <channel>\n".
	"    <title>{$space[username]}</title>\n".
	"    <link>{$space[space_url]}</link>\n".
	"    <description>{$_G[setting][bbname]}</description>\n".
	"    <copyright>Copyright(C) {$_G[setting][bbname]}</copyright>\n".
	"    <generator>Discuz! Board by Comsenz Inc.</generator>\n".
	"    <lastBuildDate>".gmdate('r', TIMESTAMP)."</lastBuildDate>\n".
	"    <image>\n".
	"      <url>{$_G[siteurl]}static/image/common/logo_88_31.gif</url>\n".
	"      <title>{$_G[setting][bbname]}</title>\n".
	"      <link>{$_G[siteurl]}</link>\n".
	"    </image>\n";

//输出用户个人动态，$data_blog是根据uid在home_blog表中获取的
//$data_blog = C::t('home_blog')->range(0, $pagenum, 'DESC', 'dateline', 0, null, $uid);
foreach($data_blog as $curblogid => $value) {
	$value = array_merge($value, (array)$data_blogfield[$curblogid]);
	$value['message'] = getstr($value['message'], 300, 0, 0, 0, -1);
	if($value['pic']) {
		$value['pic'] = pic_cover_get($value['pic'], $value['picflag']);
		$value['message'] .= "<br /><img src=\"$value[pic]\">";
	}
	echo 	"    <item>\n".
			"      <title>".$value['subject']."</title>\n".
			"      <link>$_G[siteurl]home.php?mod=space&amp;uid=$value[uid]&amp;do=blog&amp;id=$value[blogid]</link>\n".
			"      <description><![CDATA[".dhtmlspecialchars($value['message'])."]]></description>\n".
			"      <author>".dhtmlspecialchars($value['username'])."</author>\n".
			"      <pubDate>".gmdate('r', $value['dateline'])."</pubDate>\n".
			"    </item>\n";
}

echo 	"  </channel>\n".
	"</rss>";
```

### medal模块

访问连接:home.php?mod=medal

medal模块即勋章模块，这个模块分别有以下几个功能（对应到action字段）：

| action | 功能 |
| :--- | :--- |
|  | action为空的情况下显示所有可用勋章列表 |
| log | 显示我的勋章列表 |
| confirm | 显示申请勋章窗口 |
| apply | 确认申请勋章 |

进入medal模块，就有一句缓存相关的调用，从common\_syscache表中取出medals的缓存并存入全局变量$\_G\['cache'\]\[$cachename\]中，以便取用。

```php
loadcache('medals');
```

#### 展示所有勋章

展示所有勋章的代码如下，我添加了注释说明：

```php
include libfile('function/forum');
$medalcredits = array();
//在forum_medal表中获取所有勋章并遍历
foreach(C::t('forum_medal')->fetch_all_data(1) as $medal) {
	//勋章获取权限，如金钱>100、威望>100等
	$medal['permission'] = medalformulaperm(serialize(array('medal' => dunserialize($medal['permission']))), 1);
	//勋章也可设置为可使用积分、威望等值购买
	if($medal['price']) {
		$medal['credit'] = $medal['credit'] ? $medal['credit'] : $_G['setting']['creditstransextra'][3];
		$medalcredits[$medal['credit']] = $medal['credit'];
	}
	$medallist[$medal['medalid']] = $medal;
}
//在common_member_field_forum表中获取当前用户已获得的勋章列表
$memberfieldforum = C::t('common_member_field_forum')->fetch($_G['uid']);
$membermedal = $memberfieldforum['medals'] ? explode("\t", $memberfieldforum['medals']) : array();
$membermedal = array_map('intval', $membermedal);

$lastmedals = $uids = array();
//在forum_medallog表中获取最后10条勋章发放/获取记录
foreach(C::t('forum_medallog')->fetch_all_lastmedal(10) as $id => $lastmedal) {
	$lastmedal['dateline'] = dgmdate($lastmedal['dateline'], 'u');
	$lastmedals[$id] = $lastmedal;
	$uids[] = $lastmedal['uid'];
}
//获取最后10条勋章发放记录相关用户的信息
$lastmedalusers = C::t('common_member')->fetch_all($uids);
//获取当前登录用户已获得的所有勋章列表，此处的勋章列表和前面的变量$membermedal重复了，暂时不知道二者的区别
$mymedals = C::t('common_member_medal')->fetch_all_by_uid($_G['uid']);
```

#### log--我的勋章

我的勋章列表的实现代码与展示所有勋章的实现代码基本一致，唯一不同的就是勋章获取记录显示的是个人的记录，所以此处就不展开分析了。

#### comfirm--申请勋章窗口

```php
include libfile('function/forum');
//获取所有勋章列表
$medal = C::t('forum_medal')->fetch($_GET['medalid']);
//获取勋章权限和价格
$medal['permission'] = medalformulaperm(serialize(array('medal' => dunserialize($medal['permission']))), 1);
if($medal['price']) {
	$medal['credit'] = $medal['credit'] ? $medal['credit'] : $_G['setting']['creditstransextra'][3];
	$medalcredits[$medal['credit']] = $medal['credit'];
}
include template('home/space_medal_float');
```

confirm功能通过上面的代码获取勋章的详细信息，并通过下面的js代码以窗口的形式展示：

```javascript
onclick="showWindow('medal', 'home.php?mod=medal&action=confirm&medalid=7')"
```

#### apply--提交申请

提交申请的实现代码大致如下，在注释里分析其实现：

```php
$medalid = intval($_GET['medalid']);
$_G['forum_formulamessage'] = $_G['forum_usermsg'] = $medalnew = '';
//获取所有可用的勋章列表
$medal = C::t('forum_medal')->fetch($medalid);
//检查勋章是否可申请
if(!$medal['type']) {
	showmessage('medal_apply_invalid');
}
//检查是否已持有勋章
if(C::t('common_member_medal')->count_by_uid_medalid($_G['uid'], $medalid)) {
	showmessage('medal_apply_existence', 'home.php?mod=medal');
}

$applysucceed = FALSE;//是否符合申请条件
$medalpermission = $medal['permission'] ? dunserialize($medal['permission']) : '';
//检查勋章是否有用户组限制
if($medalpermission[0] || $medalpermission['usergroupallow']) {
	include libfile('function/forum');
	medalformulaperm(serialize(array('medal' => $medalpermission)), 1);

	if($_G['forum_formulamessage']) {
		showmessage('medal_permforum_nopermission', 'home.php?mod=medal', array('formulamessage' => $_G['forum_formulamessage'], 'usermsg' => $_G['forum_usermsg']));
	} else {
		$applysucceed = TRUE;
	}
} else {
	$applysucceed = TRUE;
}

if($applysucceed) {
	$expiration = empty($medal['expiration'])? 0 : TIMESTAMP + $medal['expiration'] * 86400;
	if($medal['type'] == 1) {//勋章类型为可购买
		if($medal['price']) {//勋章是否需要花积分或威望等值购买
			//积分或威望是否足够
			$medal['credit'] = $medal['credit'] ? $medal['credit'] : $_G['setting']['creditstransextra'][3];
			if($medal['price'] > getuserprofile('extcredits'.$medal['credit'])) {
				showmessage('medal_not_get_credit', '', array('credit' => $_G['setting']['extcredits'][$medal[credit]][title]));
			}
			//扣除购买积分或威望等
			updatemembercount($_G['uid'], array($medal['credit'] => -$medal['price']), true, 'BME', $medal['medalid']);
		}

		$memberfieldforum = C::t('common_member_field_forum')->fetch($_G['uid']);
		$usermedal = $memberfieldforum;
		unset($memberfieldforum);
		$medal['medalid'] = $medal['medalid'].(empty($expiration) ? '' : '|'.$expiration);
		$medalnew = $usermedal['medals'] ? $usermedal['medals']."\t".$medal['medalid'] : $medal['medalid'];
		//给当前用户添加购买的勋章
		C::t('common_member_field_forum')->update($_G['uid'], array('medals' => $medalnew));
		C::t('common_member_medal')->insert(array('uid' => $_G['uid'], 'medalid' => $medal['medalid']), 0, 1);
		$medalmessage = 'medal_get_succeed';
	} else {//勋章类型为其他可申请的类型
		//检查是否已经申请过了
		if(C::t('forum_medallog')->count_by_verify_medalid($_G['uid'], $medal['medalid'])) {
			showmessage('medal_apply_existence', 'home.php?mod=medal');
		}
		$medalmessage = 'medal_apply_succeed';
		//通知管理员审核勋章
		manage_addnotify('verifymedal');
	}
	//记录勋章申请请求
	C::t('forum_medallog')->insert(array(
	    'uid' => $_G['uid'],
	    'medalid' => $medalid,
	    'type' => $medal['type'],
	    'dateline' => TIMESTAMP,
	    'expiration' => $expiration,
	    'status' => ($expiration ? 1 : 0),
	));
	showmessage($medalmessage, 'home.php?mod=medal', array('medalname' => $medal['name']));
}
```

