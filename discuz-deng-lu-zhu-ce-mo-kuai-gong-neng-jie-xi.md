# Discuz登录注册模块功能解析

## 知识点

## 功能解析

discuz的member模块主管用户的登录、注册、账号激活、忘记密码和切换在线状态等功能，下面我们一一分析这些功能。排列顺序按简单到难吧。

### 切换在线状态

切换在线状态的实现在/source/module/member\_switchstatus.php中。

discuz用户有在线和隐身两种状态，用户点击更改状态时，后台会在数据库及session中（如果有开启session）更新用户状态：

```php
$_G['session']['invisible'] = $_G['session']['invisible'] ? 0 : 1;
C::app()->session->update_by_uid($_G['uid'], array('invisible' => $_G['session']['invisible']));
C::t('common_member_status')->update($_G['uid'], array('invisible' => $_G['session']['invisible']), 'UNBUFFERED');
```

### 账号激活

账号激活的实现在/source/module/member\_activate.php中。

discuz自带了邮件激活账号的功能，激活流程比较简单。首先用户注册完之后，系统会生成6位数的随机认证码，并且用这个码拼成一个链接通过邮件发送给用户：

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
sendmail("$username <$email>", lang('email', 'email_verify_subject'), $email_verify_message)
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

忘记密码取回密码的实现在/source/module/member\_lostpassword.php和member\_getpassword.php中。



