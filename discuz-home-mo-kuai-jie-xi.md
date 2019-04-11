# Discuz home模块解析

## 功能解析

home模块很庞大，包含了discuz论坛和用户信息相关的几乎所有功能，比如动态、消息、勋章、道具等等，有的功能还分为很多小功能，下面列出home模块基本的组成：

| 子模块 | 说明 |
| :--- | :--- |
| space | 用户个人空间相关的功能，如活动、相册、日志、收藏、好友、分享、悬赏等等 |
| spacecp | 用户个人空间相关功能的改动，如添加好友、发日志、收藏帖子等等 |
| misc |  |
| magic | 道具相关 |
| editor | 编辑器 |
| invite | 邀请相关 |
| task | 任务相关，如任务列表、申请任务、获取奖励、放弃任务等功能 |
| medal | 勋章相关，如勋章列表、申请勋章等功能 |
| rss | 网站信息聚合页 |
| follow | 广播相关 |

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

### task模块

discuz自带任务模块，可在discuz管理中心-&gt;运营-&gt;站点任务中开启任务模块并管理论坛任务。

任务模块主要有以下几个功能，请求的字段do的值代表功能：

| do值 | 功能 |
| :--- | :--- |
|  | 当do字段值为空时，表示浏览任务列表 |
| view | 查看任务详情 |
| apply | 申请任务 |
| delete | 放弃任务 |
| draw | 领取奖励 |
| giveup | 放弃任务，功能与delete一样，暂时未知被哪里调用 |
| parter | 参与某任务的用户列表 |

任务模块的业务逻辑实现主要在/source/class/class\_task.php中，其数据存储主要跟表common\_task、common\_mytask和common\_taskvar表相关。

| 数据库表 | 描述 |
| :--- | :--- |
| common\_task | 存储所有论坛任务 |
| common\_mytask | 存储所有用户和任务的关系 |
| common\_taskvar | 存储任务的详细信息 |

#### 展示任务列表

获取任务列表的功能代码如下，我将在注释中解析功能：

```php
function tasklist($item) {
		...
		//从数据库中获取与用户和item相关的任务列表，item有4个值new/doing/done/failed分别对应新任务、进行中任务、已完成任务和失败任务
		foreach(C::t('common_task')->fetch_all_by_status($_G['uid'], $item) as $task) {
			//从fetch_all_by_status方法中看，new和canapply没有区别
			if($item == 'new' || $item == 'canapply') {
				//检查任务的时间限制
				list($task['allowapply'], $task['t']) = $this->checknextperiod($task);
				if($task['allowapply'] < 0) {//不符合时间限制则不显示任务
					continue;
				}
				$task['noperm'] = $task['applyperm'] && $task['applyperm'] != 'all' && !(($task['applyperm'] == 'member'&& $_G['adminid'] == '0') || ($task['applyperm'] == 'admin' && $_G['adminid'] > '0') || forumperm($task['applyperm']));
				$task['appliesfull'] = $task['tasklimits'] && $task['achievers'] >= $task['tasklimits'];
				//不符合权限限制和已申请满员则不显示任务
				if($item == 'canapply' && ($task['noperm'] || $task['appliesfull'])) {
					continue;
				}
			}
			$num++;
			//任务的奖励类型
			if($task['reward'] == 'magic') {
				$magicids[] = $task['prize'];
			} elseif($task['reward'] == 'medal') {
				$medalids[] = $task['prize'];
			} elseif($task['reward'] == 'invite') {
				$invitenum = $task['prize'];
			} elseif($task['reward'] == 'group') {
				$groupids[] = $task['prize'];
			}
			//任务已启用且在可申请时间内
			if($task['available'] == '2' && ($task['starttime'] > TIMESTAMP || ($task['endtime'] && $task['endtime'] <= TIMESTAMP))) {
				$endtaskids[] = $task['taskid'];
			}
			//csc字段包含了任务进度百分比和最后更新时间，用\t隔开
			$csc = explode("\t", $task['csc']);
			$task['csc'] = floatval($csc[0]);//获取任务进度
			$task['lastupdate'] = intval($csc[1]);//获取任务进度更新时间
			//更新任务进度
			if(!$updated && $item == 'doing' && $task['csc'] < 100) {
				$updated = TRUE;
				$escript = explode(':', $task['scriptname']);
				//discuz支持任务插件，插件任务存在/source/plugin/中，原生任务在/source/class/task/中
				if(count($escript) > 1) {
					include_once DISCUZ_ROOT.'./source/plugin/'.$escript[0].'/task/task_'.$escript[1].'.php';
					$taskclassname = 'task_'.$escript[1];
				} else {
					require_once libfile('task/'.$task['scriptname'], 'class');
					$taskclassname = 'task_'.$task['scriptname'];
				}
				$taskclass = new $taskclassname;
				$task['applytime'] = $task['dateline'];
				if(method_exists($taskclass, 'csc')) {
					$result = $taskclass->csc($task);//计算任务进度
				} else {
					showmessage('task_not_found', '', array('taskclassname' => $taskclassname));
				}
				//更新用户和任务的关系
				if($result === TRUE) {
					$task['csc'] = '100';
					C::t('common_mytask')->update($_G['uid'], $task['taskid'], array('csc' => $task['csc']));
				} elseif($result === FALSE) {
					C::t('common_mytask')->update($_G['uid'], $task['taskid'], array('status' => -1));
				} else {
					$task['csc'] = floatval($result['csc']);
					C::t('common_mytask')->update($_G['uid'], $task['taskid'], array('csc' => $task['csc']."\t".$_G['timestamp']));
				}
			}
			//如果该任务已完成或者已失败，检查是否可重新申请
			if(in_array($item, array('done', 'failed')) && $task['period']) {
				list($task['allowapply'], $task['t']) = $this->checknextperiod($task);
				$task['allowapply'] = $task['allowapply'] > 0 ? 1 : 0;
			}
			//获取任务图标
			$task['icon'] = $task['icon'] ? $task['icon'] : 'task.gif';
			if(strtolower(substr($task['icon'], 0, 7)) != 'http://') {
				$escript = explode(':', $task['scriptname']);
				if(count($escript) > 1 && file_exists(DISCUZ_ROOT.'./source/plugin/'.$escript[0].'/task/task_'.$escript[1].'.gif')) {
					$task['icon'] = 'source/plugin/'.$escript[0].'/task/task_'.$escript[1].'.gif';
				} else {
					$task['icon'] = 'static/image/task/'.$task['icon'];
				}
			}
			$task['dateline'] = $task['dateline'] ? dgmdate($task['dateline'], 'u') : '';
			$tasklist[] = $task;
		}
		//根据任务奖励类型获取奖励相关的信息
		//获取奖励物品名
		if($magicids) {
			foreach(C::t('common_magic')->fetch_all($magicids) as $magic) {
				$this->listdata[$magic['magicid']] = $magic['name'];
			}
		}
		//获取奖励奖章名
		if($medalids) {
			foreach(C::t('forum_medal')->fetch_all($medalids) as $medal) {
				$this->listdata[$medal['medalid']] = $medal['name'];
			}
		}
		//获取奖励用户组名
		if($groupids) {
			foreach(C::t('common_usergroup')->fetch_all($groupids) as $group) {
				$this->listdata[$group['groupid']] = $group['grouptitle'];
			}
		}
		//获取奖励邀请码
		if($invitenum) {
			$this->listdata[$invitenum] = $_G['lang']['invite_code'];
		}
		//无奖励
		if($endtaskids) {
		}
		return $tasklist;
	}
```

#### view--任务详情

任务详情的代码如下：

```php
function view($id) {
	//获取任务信息
	$this->task = C::t('common_task')->fetch_by_uid($_G['uid'], $id);
	...
	//获取任务奖励信息
	switch($this->task['reward']) {
		case 'magic':
			$this->task['rewardtext'] = C::t('common_magic')->fetch($this->task['prize']);
			$this->task['rewardtext'] = $this->task['rewardtext']['name'];
			break;
		case 'medal':
			$this->task['rewardtext'] = C::t('forum_medal')->fetch($this->task['prize']);
			$this->task['rewardtext'] = $this->task['rewardtext']['name'];
			break;
		case 'group':
			$group = C::t('common_usergroup')->fetch($this->task['prize']);
			$this->task['rewardtext'] = $group['grouptitle'];
			break;
	}
	//获取任务图标链接
	$this->task['icon'] = $this->task['icon'] ? $this->task['icon'] : 'task.gif';
	if(strtolower(substr($this->task['icon'], 0, 7)) != 'http://') {
		$escript = explode(':', $this->task['scriptname']);
		if(count($escript) > 1 && file_exists(DISCUZ_ROOT.'./source/plugin/'.$escript[0].'/task/task_'.$escript[1].'.gif')) {
			$this->task['icon'] = 'source/plugin/'.$escript[0].'/task/task_'.$escript[1].'.gif';
		} else {
			$this->task['icon'] = 'static/image/task/'.$this->task['icon'];
		}
	}
	//获取任务结束时间和任务描述
	$this->task['endtime'] = $this->task['endtime'] ? dgmdate($this->task['endtime'], 'u') : '';
	$this->task['description'] = nl2br($this->task['description']);

	$this->taskvars = array();
	foreach(C::t('common_taskvar')->fetch_all_by_taskid($id) as $taskvar) {
		if(!$taskvar['variable'] || $taskvar['value']) {
			if(!$taskvar['variable']) {
				$taskvar['value'] = $taskvar['description'];
			}
			//用户相对该任务的状态
			if($taskvar['sort'] == 'apply') {
				$this->taskvars['apply'][] = $taskvar;
			} elseif($taskvar['sort'] == 'complete') {
				$this->taskvars['complete'][$taskvar['variable']] = $taskvar;
			} elseif($taskvar['sort'] == 'setting') {
				$this->taskvars['setting'][$taskvar['variable']] = $taskvar;
			}
		}
	}
	//获取允许申请任务的用户组信息
	$this->task['grouprequired'] = $comma = '';
	$this->task['applyperm'] = $this->task['applyperm'] == 'all' ? '' : $this->task['applyperm'];
	if(!in_array($this->task['applyperm'], array('', 'member', 'admin'))) {
		$query = C::t('common_usergroup')->fetch_all(explode(',', str_replace("\t", ',', $this->task['applyperm'])));
		foreach($query as $group) {
			$this->task['grouprequired'] .= $comma.$group[grouptitle];
			$comma = ', ';
		}
	}
	//申请此任务前需要完成任务的任务信息
	if($this->task['relatedtaskid']) {
		$task = C::t('common_task')->fetch($this->task['relatedtaskid']);
		$_G['taskrequired'] = $task['name'];
	}
	//根据scriptname判断任务类型是内置任务还是插件类型的任务，并且获取任务对应的实现类
	$escript = explode(':', $this->task['scriptname']);
	if(count($escript) > 1) {
		include_once DISCUZ_ROOT.'./source/plugin/'.$escript[0].'/task/task_'.$escript[1].'.php';
		$taskclassname = 'task_'.$escript[1];
	} else {
		require_once libfile('task/'.$this->task['scriptname'], 'class');
		$taskclassname = 'task_'.$this->task['scriptname'];
	}
	$taskclass = new $taskclassname;
	if($this->task['status'] == '-1') {//失败的任务
		if($this->task['period']) {//检查申请时间限制
			list($allowapply, $this->task['t']) = $this->checknextperiod($this->task);
		} else {
			$allowapply = -4;//小于0则为不可申请
		}
	} elseif($this->task['status'] == '0') {//正在执行的任务
		...
		if($this->task['csc'] < 100) {//如果任务进度不是已完成，则检查任务进度
			if(method_exists($taskclass, 'csc')) {
				$result = $taskclass->csc($this->task);//更新任务进度，返回值为任务是否已完成或者进度
			}
			...
		}
	} elseif($this->task['status'] == '1') {//已完成的任务
		if($this->task['period']) {//检查是否可重新申请
			list($allowapply, $this->task['t']) = $this->checknextperiod($this->task);
		} else {
			$allowapply = -5;
		}
	} else {
		$allowapply = 1;
	}
	if(method_exists($taskclass, 'view')) {//获取勋章展示信息
		$this->task['viewmessage'] = $taskclass->view($this->task, $this->taskvars);
	} else {
		$this->task['viewmessage'] = '';
	}
	if($allowapply > 0) {//检查是否有其他申请限制
		if($this->task['applyperm'] && $this->task['applyperm'] != 'all' && !(($this->task['applyperm'] == 'member' && $_G['adminid'] == '0') || ($this->task['applyperm'] == 'admin' && $_G['adminid'] > '0') || preg_match("/(^|\t)(".$_G['groupid'].")(\t|$)/", $this->task['applyperm']))) {
			$allowapply = -2;
		} elseif($this->task['tasklimits'] && $this->task['achievers'] >= $this->task['tasklimits']) {
			$allowapply = -3;
		}
	}
	
	$this->task['dateline'] = dgmdate($this->task['dateline'], 'u');
	return $allowapply;

}
```





