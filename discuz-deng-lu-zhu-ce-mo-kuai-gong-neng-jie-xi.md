# Discuz member模块功能解析

## 知识点

1.ip转换成地址

discuzip装换成地址用的是离线数据，一个叫纯真IP地址数据库的数据库，存放了几乎所有ip到地址的映射，且经常有更新，可以在这个[git库](https://github.com/WisdomFusion/qqwry.dat)上下载，也可以网上搜qqwry.dat。

## 功能解析

discuz的member模块主管用户的登录、注册、账号激活、忘记密码和切换在线状态等功能，下面我们一一分析这些功能。排列顺序按简单到难吧。

### 切换在线状态

切换在线状态的实现在/source/module/member/member\_switchstatus.php中。

discuz用户有在线和隐身两种状态，用户点击更改状态时，后台会在数据库及session中（如果有开启session）更新用户状态：

```php
$_G['session']['invisible'] = $_G['session']['invisible'] ? 0 : 1;
C::app()->session->update_by_uid($_G['uid'], array('invisible' => $_G['session']['invisible']));
C::t('common_member_status')->update($_G['uid'], array('invisible' => $_G['session']['invisible']), 'UNBUFFERED');
```

### 账号激活

账号激活的实现在/source/module/member/member\_emailverify.php和member\_activate.php中。

discuz自带了邮件激活账号的功能，激活流程比较简单。首先用户注册完之后，系统会生成6位数的随机认证码，并且用这个码拼成一个链接指向member\_activate.php，通过邮件发送给用户：

```php
$idstring = random(6);
...
$verifyurl = "{$_G[siteurl]}member.php?mod=activate&amp;uid={$_G[uid]}&amp;id=$idstring";
$email_verify_message = lang('email', 'email_verify_message', array(
						'username' => $_G['member']['username'],
						'bbname' => $this->setting['bbname'],
						'siteurl' => $_G['siteurl'],
						'url' => $verifyurl
					));
sendmail("$username <$email>", lang('email', 'email_verify_subject'), $email_verify_message);
```

用户点击链接之后会进入激活流程，系统会验证请求的uid和认证码是否匹配，若匹配，则激活成功，系统将会更新该用户的用户组和激活状态：

```php
if($operation == 2 && $idstring == $_GET['id']) {
	$newgroup = C::t('common_usergroup')->fetch_by_credits($member['credits']);
	C::t('common_member')->update($member['uid'], array('groupid' => $newgroup['groupid'], 'emailstatus' => '1'));
	C::t('common_member_field_forum')->update($member['uid'], array('authstr' => ''));
	showmessage('activate_succeed', 'index.php', array('username' => $member['username']));
} else {
	showmessage('activate_illegal', 'index.php');
}
```

从上面的代码看，激活账号没有时间限制，但是在discuz源码的其他实现中，随机认证码是有时限的，超过了时限就会失效，如下面的忘记密码功能。

### 忘记密码

忘记密码取回密码的实现在/source/module/member/member\_lostpassword.php和member\_getpassword.php中。

忘记密码取回密码的流程和账号激活的流程基本类似，先是系统随机生成一个6位的随机验证码，然后组合成一个重置密码的链接指向member\_getpassword.php，通过邮件发送到用户的注册邮箱中：

```php
$idstring = $type == 2 && $idstring ? $idstring : random(6);
...
$email_verify_message = lang('email', 'email_verify_message', array(
	'username' => $_G['member']['username'],
	'bbname' => $_G['setting']['bbname'],
	'siteurl' => $_G['siteurl'],
	'url' => $verifyurl
));
...
sendmail("{$_G[member][username]} <$_GET[email]>", lang('email', 'email_verify_subject'), $email_verify_message);
```

用户在邮件中打开重置密码的链接时，系统会验证链接的有效性，并提供重新输入密码的界面，用户提交重置的密码后，系统会验证密码的非法字符、长度和强度，最后将密码保存到数据库中：

```php
if($_GET['uid'] && $_GET['id'] && $_GET['sign'] === make_getpws_sign($_GET['uid'], $_GET['id'])) {
    //验证密码非法字符
    if($_GET['newpasswd1'] != addslashes($_GET['newpasswd1'])) {
			showmessage('profile_passwd_illegal');
	}
	//验证密码长度
	if($_G['setting']['pwlength']) {
		if(strlen($_GET['newpasswd1']) < $_G['setting']['pwlength']) {
			showmessage('profile_password_tooshort', '', array('pwlength' => $_G['setting']['pwlength']));
		}
	}
	//验证密码强度
	if($_G['setting']['strongpw']) {
		$strongpw_str = array();
		if(in_array(1, $_G['setting']['strongpw']) && !preg_match("/\d+/", $_GET['newpasswd1'])) {
			$strongpw_str[] = lang('member/template', 'strongpw_1');
		}
		if(in_array(2, $_G['setting']['strongpw']) && !preg_match("/[a-z]+/", $_GET['newpasswd1'])) {
			$strongpw_str[] = lang('member/template', 'strongpw_2');
		}
		if(in_array(3, $_G['setting']['strongpw']) && !preg_match("/[A-Z]+/", $_GET['newpasswd1'])) {
			$strongpw_str[] = lang('member/template', 'strongpw_3');
		}
		if(in_array(4, $_G['setting']['strongpw']) && !preg_match("/[^a-zA-z0-9]+/", $_GET['newpasswd1'])) {
			$strongpw_str[] = lang('member/template', 'strongpw_4');
		}
		if($strongpw_str) {
			showmessage(lang('member/template', 'password_weak').implode(',', $strongpw_str));
		}
	}
	...
	C::t('common_member')->update($_GET['uid'], array('password' => $password));
}
```

### 注册

注册功能的实现在/source/module/member/member\_register.php和/source/class/class\_member.php和register\_ctl类中。

注册的流程比较复杂，整个on\_register函数有将近600行代码，接下来我们看看主要流程。首先系统会判断管理员是否设置了注册验证，分别验证注册用户所在地或者ip是否在白名单内：

```php
if($this->setting['regverify']) {
	if($this->setting['areaverifywhite']) {
		$location = $whitearea = '';
		$location = trim(convertip($_G['clientip'], "./"));
		if($location) {
			$whitearea = preg_quote(trim($this->setting['areaverifywhite']), '/');
			$whitearea = str_replace(array("\\*"), array('.*'), $whitearea);
			$whitearea = '.*'.$whitearea.'.*';
			$whitearea = '/^('.str_replace(array("\r\n", ' '), array('.*|.*', ''), $whitearea).')$/i';
			if(@preg_match($whitearea, $location)) {
				$this->setting['regverify'] = 0;
			}
		}
	}

	if($_G['cache']['ipctrl']['ipverifywhite']) {
		foreach(explode("\n", $_G['cache']['ipctrl']['ipverifywhite']) as $ctrlip) {
			if(preg_match("/^(".preg_quote(($ctrlip = trim($ctrlip)), '/').")/", $_G['clientip'])) {
				$this->setting['regverify'] = 0;
				break;
			}
		}
	}
}
```

如果注册用户填写了邀请码，还会进行邀请码验证，同样是验证用户所在地和ip是否在白名单内：

```php
if($this->setting['regstatus'] == 2) {
	if($this->setting['inviteconfig']['inviteareawhite']) {
		$location = $whitearea = '';
		$location = trim(convertip($_G['clientip'], "./"));
		if($location) {
			$whitearea = preg_quote(trim($this->setting['inviteconfig']['inviteareawhite']), '/');
			$whitearea = str_replace(array("\\*"), array('.*'), $whitearea);
			$whitearea = '.*'.$whitearea.'.*';
			$whitearea = '/^('.str_replace(array("\r\n", ' '), array('.*|.*', ''), $whitearea).')$/i';
			if(@preg_match($whitearea, $location)) {
				$invitestatus = true;
			}
		}
	}

	if($this->setting['inviteconfig']['inviteipwhite']) {
		foreach(explode("\n", $this->setting['inviteconfig']['inviteipwhite']) as $ctrlip) {
			if(preg_match("/^(".preg_quote(($ctrlip = trim($ctrlip)), '/').")/", $_G['clientip'])) {
				$invitestatus = true;
				break;
			}
		}
	}
}
```

然后校验验证码、验证用户名长度、验证用户名是否重复、密码长度强度、查找用户名屏蔽字符、ip限制等：

```php
if(!submitcheck('regsubmit', 0, $seccodecheck, $secqaacheck)) {
    ...
} else {
    ...
    //验证用户名长度
    if($usernamelen < 3) {
    	showmessage('profile_username_tooshort');
    } elseif($usernamelen > 15) {
    	showmessage('profile_username_toolong');
    }
    ...
    //验证用户名重复
    if(uc_get_user(addslashes($username)) && !C::t('common_member')->fetch_uid_by_username($username) && !C::t('common_member_archive')->fetch_uid_by_username($username)) {
		if($_G['inajax']) {
			showmessage('profile_username_duplicate');
		} else {
			showmessage('register_activation_message', 'member.php?mod=logging&action=login', array('username' => $username));
		}
	}
	//验证密码长度
	if($this->setting['pwlength']) {
		if(strlen($_GET['password']) < $this->setting['pwlength']) {
			showmessage('profile_password_tooshort', '', array('pwlength' => $this->setting['pwlength']));
		}
	}
	//验证密码强度
	if($this->setting['strongpw']) {
		$strongpw_str = array();
		if(in_array(1, $this->setting['strongpw']) && !preg_match("/\d+/", $_GET['password'])) {
			$strongpw_str[] = lang('member/template', 'strongpw_1');
		}
		if(in_array(2, $this->setting['strongpw']) && !preg_match("/[a-z]+/", $_GET['password'])) {
			$strongpw_str[] = lang('member/template', 'strongpw_2');
		}
		if(in_array(3, $this->setting['strongpw']) && !preg_match("/[A-Z]+/", $_GET['password'])) {
			$strongpw_str[] = lang('member/template', 'strongpw_3');
		}
		if(in_array(4, $this->setting['strongpw']) && !preg_match("/[^a-zA-z0-9]+/", $_GET['password'])) {
			$strongpw_str[] = lang('member/template', 'strongpw_4');
		}
		if($strongpw_str) {
			showmessage(lang('member/template', 'password_weak').implode(',', $strongpw_str));
		}
	}
	...
	//验证屏蔽字符
	if($this->setting['censoruser'] && @preg_match($censorexp, $username)) {
		showmessage('profile_username_protect');
	}
	...
	//验证ip注册时间间隔
	if($this->setting['regctrl']) {
		if(C::t('common_regip')->count_by_ip_dateline($ctrlip, $_G['timestamp']-$this->setting['regctrl']*3600)) {
			showmessage('register_ctrl', NULL, array('regctrl' => $this->setting['regctrl']));
		}
	}
	...
	//验证同一ip注册次数上限
	if($regip['count'] >= $this->setting['regfloodctrl']) {
		showmessage('register_flood_ctrl', NULL, array('regfloodctrl' => $this->setting['regfloodctrl']));
	} else {
		$setregip = 1;
	}
	...
	//向ucenter注册，如果有错误则打印错误内容
	$uid = uc_user_register(addslashes($username), $password, $email, $questionid, $answer, $_G['clientip']);
	if($uid <= 0) {
		if($uid == -1) {
			showmessage('profile_username_illegal');
		} elseif($uid == -2) {
			showmessage('profile_username_protect');
		} elseif($uid == -3) {
			showmessage('profile_username_duplicate');
		} elseif($uid == -4) {
			showmessage('profile_email_illegal');
		} elseif($uid == -5) {
			showmessage('profile_email_domain_illegal');
		} elseif($uid == -6) {
			showmessage('profile_email_duplicate');
		} else {
			showmessage('undefined_action');
		}
	}
	...
	//保存用户信息
	$init_arr = array('credits' => explode(',', $this->setting['initcredits']), 'profile'=>$profile, 'emailstatus' => $emailstatus);
	C::t('common_member')->insert($uid, $username, $password, $email, $_G['clientip'], $groupinfo['groupid'], $init_arr);
		
}
```

注册流程中还有不少地方不怎么理解，如果有需要再深入分析代码。

### 登录

登录功能相比注册功能简单一点，过程如下：

```php
...
//验证密码输错次数
if(!($_G['member_loginperm'] = logincheck($_GET['username']))) {
    captcha::report($_G['clientip']);
    showmessage('login_strike');
}
...
//验证非法字符
if(!$_GET['password'] || $_GET['password'] != addslashes($_GET['password'])) {
    showmessage('profile_passwd_illegal');
}
...
//检查用户名密码等信息是否匹配
$result = userlogin($_GET['username'], $_GET['password'], $_GET['questionid'], $_GET['answer'], $this->setting['autoidselect'] ? 'auto' : $_GET['loginfield'], $_G['clientip']);
...
//将用户信息从存档表中移动到用户主表中，猜测是更改用户状态为已上线
C::t('common_member_archive')->move_to_master($member['uid']);
```



