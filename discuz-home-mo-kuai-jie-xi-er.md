# Discuz home模块解析（二）

这篇文章将继续分析home模块其余的功能。

### invite模块

invite模块是邀请注册或者成为好友功能，邀请功能可以在管理后台的全局-&gt;注册与访问控制中开启。普通用户邀请的链接如下：

```text
http://127.0.0.1/home.php?mod=invite&id=8&c=zakbhb
```

管理员批量邀请的链接如下：

```text
http://127.0.0.1/home.php?mod=invite&u=1&c=7219c3069259f602
```

用户通过该链接访问论坛，论坛调用invite模块进行认证等操作，invite模块的主要功能如下：

```php
$id = intval($_GET['id']);//普通用户邀请链接的邀请码id
$uid = intval($_GET['u']);//管理员批量邀请链接的管理员id
$appid = intval($_GET['app']);
$acceptconfirm = false;//用户是否接受邀请
if($_G['setting']['regstatus'] < 2) {//论坛未开启邀请注册功能
	showmessage('not_open_invite', '', array(), array('return' => true));
}
if($_G['uid']) {//如果用户已登录
	//用户已登录状态下打开邀请链接，有接受邀请和忽略邀请两个选项，分别对应accept=yes和accept=no
	if($_GET['accept'] == 'yes') {
		$cookies = empty($_G['cookie']['invite_auth'])?array():explode(',', $_G['cookie']['invite_auth']);

		if(empty($cookies)) {
			showmessage('invite_code_error', '', array(), array('return' => true));
		}
		if(count($cookies) == 3) {
			$uid = intval($cookies[0]);
			$_GET['c'] = $cookies[1];
			$appid = intval($cookies[2]);
		} else {
			$id = intval($cookies[0]);
			$_GET['c'] = $cookies[1];
		}
		$acceptconfirm = true;

	} elseif($_GET['accept'] == 'no') {
		dsetcookie('invite_auth', '');
		showmessage('invite_accept_no', 'home.php');
	}
}

if($id) {//如果是普通邀请

	$invite = C::t('common_invite')->fetch($id);
	//检查邀请码是否一致
	if(empty($invite) || $invite['code'] != $_GET['c']) {
		showmessage('invite_code_error', '', array(), array('return' => true));
	}
	//邀请码已经使用过了
	if($invite['fuid'] && $invite['fuid'] != $_G['uid']) {
		showmessage('invite_code_fuid', '', array(), array('return' => true));
	}
	//邀请码失效检查
	if($invite['endtime'] && $_G['timestamp'] > $invite['endtime']) {
		C::t('common_invite')->delete($id);
		showmessage('invite_code_endtime_error', '', array(), array('return' => true));
	}

	$appid = $invite['appid'];
	$uid = $invite['uid'];

	$cookievar = "$id,$invite[code]";

} elseif ($uid) {//如果是批量邀请
	$id = 0;
	//检查邀请码
	$invite_code = space_key($uid, $appid);
	if($_GET['c'] != $invite_code) {//
		showmessage('invite_code_error', '', array(), array('return' => true));
	}
	//邀请者用户组检查
	$inviteuser = getuserbyuid($uid);
	loadcache('usergroup_'.$inviteuser['groupid']);
	if(!empty($_G['cache']['usergroup_'.$inviteuser['groupid']]) && $_G['cache']['usergroup_'.$inviteuser['groupid']]['inviteprice']) {
		showmessage('invite_code_error', '', array(), array('return' => true));
	}
	$cookievar = "$uid,$invite_code,$appid";
} else {
	showmessage('invite_code_error', '', array(), array('return' => true));
}
$userapp = array();
if($appid) {
	$userapp = C::t('common_myapp')->fetch($appid);
}
//获取管理员邀请者用户信息
$space = getuserbyuid($uid);
if(empty($space)) {
	showmessage('space_does_not_exist', '', array(), array('return' => true));
}
$jumpurl = $appid ? "userapp.php?mod=app&id=$appid&my_extra=invitedby_bi_{$uid}_$_GET[c]&my_suffix=Lw%3D%3D" : 'home.php?mod=space&uid='.$uid;
if($acceptconfirm) {//用户接受好友申请
	...
	//加为好友
	friend_make($space['uid'], $space['username']);

	if($id) {
		//邀请码设置为已使用
		C::t('common_invite')->update($id, array('fuid'=>$_G['uid'], 'fusername'=>$_G['username'], 'regdateline' => $_G['timestamp'], 'status' => 2));
		//加好友通知
		notification_add($uid, 'friend', 'invite_friend', array('actor' => '<a href="home.php?mod=space&uid='.$_G['uid'].'" target="_blank">'.$_G['username'].'</a>'), 1);
	}
	//将用户信息重表field_home中取出，合并到space变量中
	space_merge($space, 'field_home');
	if(!empty($space['privacy']['feed']['invite'])) {
		require_once libfile('function/feed');
		$tite_data = array('username' => '<a href="home.php?mod=space&uid='.$_G['uid'].'">'.$_G['username'].'</a>');
		feed_add('friend', 'feed_invite', $tite_data, '', array(), '', array(), array(), '', '', '', 0, 0, '', $space['uid'], $space['username']);
	}
	//受邀请方接受邀请的积分奖励
	if($_G['setting']['inviteconfig']['inviteaddcredit']) {
		updatemembercount($_G['uid'],
			array($_G['setting']['inviteconfig']['inviterewardcredit'] => $_G['setting']['inviteconfig']['inviteaddcredit']));
	}
	//邀请方邀请用户的积分奖励
	if($_G['setting']['inviteconfig']['invitedaddcredit']) {
		updatemembercount($uid,
			array($_G['setting']['inviteconfig']['inviterewardcredit'] => $_G['setting']['inviteconfig']['invitedaddcredit']));
	}
	//
	include_once libfile('function/stat');
	updatestat($appid ? 'appinvite' : 'invite');

	showmessage('invite_friend_ok', $jumpurl);
} else {
	dsetcookie('invite_auth', $cookievar, 604800);
}
//获取邀请者用户信息
space_merge($space, 'count');
space_merge($space, 'field_home');
space_merge($space, 'profile');
//获取邀请者好友列表
$flist = array();
$query = C::t('home_friend')->fetch_all_by_uid($uid, 0, 12, true);
foreach($query as $value) {
	$value['uid'] = $value['fuid'];
	$value['username'] = $value['fusername'];
	$flist[] = $value;
}
$jumpurl = urlencode($jumpurl);
include_once template('home/invite');
```

### magic模块

magic模块在discuz中是道具模块。道具模块的功能结构大致如下：

| action | operation | 说明 |
| :--- | :--- | :--- |
| shop | index | 道具商店默认页面 |
|  | hot | 热销道具页面 |
|  | buy | 购买页面及购买功能 |
|  | give | 赠送页面及赠送功能 |
| mybox | use | 使用道具 |
|  | drop | 丢弃道具 |
|  | give | 赠送道具 |
| log | uselog | 道具使用日志 |
|  | buylog | 道具购买日志 |
|  | givelog | 道具赠送日志 |
|  | receivelog | 道具获赠日志 |

接下来我们一一分析上面的功能：

#### shop--道具商店相关功能

道具商店index和hot页面和功能实现如下：

```php
//在common_magic表中获取可用道具的总个数。
$magiccount = C::t('common_magic')->count_page($operation);
//生成翻页布局
$multipage = multi($magiccount, $_G['tpp'], $page, "home.php?mod=magic&action=shop&operation=$operation");
//在common_magic表中获取当前页面的道具列表。$_G['tpp']为每页道具数量，默认12。
foreach(C::t('common_magic')->fetch_all_page($operation, $start_limit, $_G['tpp']) as $magic) {
	//获取道具出售的折扣并计算出售价格，跟用户所在的用户组有关
	$magic['discountprice'] = $_G['group']['magicsdiscount'] ? intval($magic['price'] * ($_G['group']['magicsdiscount'] / 10)) : intval($magic['price']);
	$eidentifier = explode(':', $magic['identifier']);
	//获取道具图标
	if(count($eidentifier) > 1) {
		//这里可以看出道具也支持插件
		$magic['pic'] = 'source/plugin/'.$eidentifier[0].'/magic/magic_'.$eidentifier[1].'.gif';
	} else {
		$magic['pic'] = STATICURL.'image/magic/'.strtolower($magic['identifier']).'.gif';
	}
	$magiclist[] = $magic;
}
...
//输出最终页面
include template('home/space_magic');
```

道具商店购买功能实现如下：

```php
...
if(!submitcheck('operatesubmit')) {//显示确认购买窗口
	...
	//允许被使用道具的用户组
	if($magicperm['targetgroups']) {
		loadcache('usergroups');
		foreach(explode("\t", $magicperm['targetgroups']) as $_G['groupid']) {
			if(isset($_G['cache']['usergroups'][$_G['groupid']])) {
				$targetgroupperm .= $comma.$_G['cache']['usergroups'][$_G['groupid']]['grouptitle'];
				$comma = '&nbsp;';
			}
		}
	}
	//允许使用道具的版块
	if($magicperm['forum']) {
		loadcache('forums');
		foreach(explode("\t", $magicperm['forum']) as $fid) {
			if(isset($_G['cache']['forums'][$fid])) {
				$forumperm .= $comma.'<a href="forum.php?mod=forumdisplay&fid='.$fid.'" target="_blank">'.$_G['cache']['forums'][$fid]['name'].'</a>';
				$comma = '&nbsp;';
			}
		}
	}
	//页面展示
	include template('home/space_magic_shop_opreation');
	dexit();
} else {//提交购买请求
	$magicnum = intval($_GET['magicnum']);
	$magic['weight'] = $magic['weight'] * $magicnum;
	$totalprice = $magic['discountprice'] * $magicnum;
	//价差价格及剩余数量
	if(getuserprofile('extcredits'.$magic['credit']) < $totalprice) {
		if($_G['setting']['ec_ratio'] && $_G['setting']['creditstrans'][0] == $magic['credit']) {
			showmessage('magics_credits_no_enough_and_charge', '', array('credit' => $_G['setting']['extcredits'][$magic['credit']]['title']));
		} else {
			showmessage('magics_credits_no_enough', '', array('credit' => $_G['setting']['extcredits'][$magic['credit']]['title']));
		}
	} elseif($magic['num'] < $magicnum) {
		showmessage('magics_num_no_enough');
	} elseif(!$magicnum || $magicnum < 0) {
		showmessage('magics_num_invalid');
	}
	//在common_member_magic表中为用户添加道具
	getmagic($magic['magicid'], $magicnum, $magic['weight'], $totalweight, $_G['uid'], $_G['group']['maxmagicsweight']);
	//在common_magiclog表中添加道具购买记录
	updatemagiclog($magic['magicid'], '1', $magicnum, $magic['price'].'|'.$magic['credit'], $_G['uid']);
	//更新道具存量
	C::t('common_magic')->update_salevolume($magic['magicid'], $magicnum);
	//在common_member_count表中更新用户积分信息
	updatemembercount($_G['uid'], array($magic['credit'] => -$totalprice), true, 'BMC', $magic['magicid']);
	//购买成功
	showmessage('magics_buy_succeed', 'home.php?mod=magic&action=mybox', array('magicname' => $magic['name'], 'num' => $magicnum, 'credit' => $totalprice.' '.$_G['setting']['extcredits'][$magic['credit']]['unit'].$_G['setting']['extcredits'][$magic['credit']]['title']));
}
```

赠送道具：

```php
//该用户组不允许使用道具，包括赠送
if($_G['group']['allowmagics'] < 2) {
	showmessage('magics_nopermission');
}
...
if(!submitcheck('operatesubmit')) {//赠送道具确认窗口

	include libfile('function/friend');
	$buddyarray = friend_list($_G['uid'], 20);//可选的朋友列表
	include template('home/space_magic_shop_opreation');
	dexit();

} else {//确认赠送道具
	...
	//检查金币、存量等是否足够
	if(getuserprofile('extcredits'.$magic['credit']) < $totalprice) {
		if($_G['setting']['ec_ratio'] && $_G['setting']['creditstrans'][0] == $magic['credit']) {
			showmessage('magics_credits_no_enough_and_charge', '', array('credit' => $_G['setting']['extcredits'][$magic['credit']]['title']));
		} else {
			showmessage('magics_credits_no_enough', '', array('credit' => $_G['setting']['extcredits'][$magic['credit']]['title']));
		}
	} elseif($magic['num'] < $magicnum) {
		showmessage('magics_num_no_enough');
	} elseif(!$magicnum || $magicnum < 0) {
		showmessage('magics_num_invalid');
	}
	...
	//在common_member_magic表中给用户添加道具
	givemagic($toname, $magic['magicid'], $magicnum, $magic['num'], $totalprice, $givemessage, $magicarray);
	//更新道具存量
	C::t('common_magic')->update_salevolume($magic['magicid'], $magicnum);
	//在common_member_count表中更新用户积分信息
	updatemembercount($_G['uid'], array($magic['credit'] => -$totalprice), true, 'BMC', $magicid);
	showmessage('magics_buygive_succeed', 'home.php?mod=magic&action=shop', array('magicname' => $magic['name'], 'toname' => $toname, 'num' => $magicnum, 'credit' => $_G['setting']['extcredits'][$magic['credit']]['title'].' '.$totalprice.' '.$_G['setting']['extcredits'][$magic['credit']]['unit']), array('locationtime' => true));

}
```



