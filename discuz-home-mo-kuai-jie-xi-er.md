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

