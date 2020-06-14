# Logs In ! Part 1

## Description

Data printed on one of our dev application has been altered, after questioning the team responsible of its development, it doesn't seem to be their doing. The H4XX0R that changed the front seems to be a meme fan and enjoyed setting the front.

We've already closed the RCE he used, there is a sensitive database running behind it. If anyone could access it we'll be in trouble. Can you please audit this version of the application and tell us if you find anything compromising, you shouldn't be able to find the admin session.

The application is hosted at logs_in

Creator : Remsio

## Solution

http://logs_in.sharkyctf.xyz/_profiler/open?file=src/Controller/MainController.php&line=23#line23

```

    /**
     * @Route("/e48e13207341b6bffb7fb1622282247b/debug")
     * @return Response
     */
    public function debug(Profiler $profiler)
    {
        $profiler->enable();
        return $this->render('main/debug.html.twig');
    }
}

```

http://logs_in.sharkyctf.xyz/e48e13207341b6bffb7fb1622282247b/debug