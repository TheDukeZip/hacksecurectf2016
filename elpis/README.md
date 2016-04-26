# Elpis: 192.168.131.6

## by Brent Dukes (@TheDukeZip)

An initial scan run from the team's Armitage host showed only port 80 open - Apache on Windows with PHP.

![alt text](https://github.com/thedukezip/hacksecurectf2016/master/elpis/images/nmap6.png "nmap scan")

Wyatt made the observation that the server was hosting an old ZPanel instance on port 80 that had at least one known vulnerability. He was working another challenge so I gave it a look.

There is a Metasploit module for exploiting this vuln (exploit/multi/http/zpanel_information_disclosure_rce)

I ran the exploit against the server and it seemed to be able to extract some useful information but didn't result in a useful shell. I tried a number of payload options but none seemed to work in the end, so I looked into how the vulnerability actually worked.

![alt text](https://github.com/thedukezip/hacksecurectf2016/master/elpis/images/msfconsole.png "msfconsole")

It seems ZPanel shipped with a plugin called pChart2 that allows reading files from the host filesystem. So it uses directory traversal by calling:

`http://*/etc/lib/pChart2/examples/index.php?Action=View&Script=../../../../cnf/db.php`

This allows you to read the MySQL credentials from the ZPanel config file. Using those credentials, the module logs into phpMyAdmin (also included with ZPanel) and runs a SQL command to write arbitrary files to disk, such as a PHP file that will execute parameterized commands over HTTP. The module then leverages this for RCE on the host. It seems that the machines in the CTF were all set up with very strict incoming and outgoing firewall rules, preventing us from binding a shell to a port on the target machine, and also preventing us from running a reverse shell back to our own team. Both myself and a few team members think we verified this using a few different techniques although we were not exhaustive since time was ticking in the game.

Initially I modified the Metasploit module to drop a PHP file running shell_exec() into the root of the webserver. As I wanted to tweak the payload a few times I found it easier to utilize the exploit written in Perl by 0xlabs from 

`http://pastebin.com/y5Pf4Yms`

This exploit utilizes the same steps but then automatically attempts to change the root password (using another known ZPanel exploit, zsudo) so you can SSH into the machine. Since the host was running Windows, that part of the exploit didn't work, but I found it faster than Metasploit at dropping arbitrary files onto the host as I was tinkering. Line 93 is where to change the payload, you can comment out all the exploitation that it attempts after it drops the file onto the host.

![alt text](https://github.com/thedukezip/hacksecurectf2016/master/elpis/images/0xlabs.png "0xlabs")

Once I was able to run shell commands on the host it was trivial to read the flag at c:\flag.txt. 

![alt text](https://github.com/thedukezip/hacksecurectf2016/master/elpis/images/userflag.png "user flag")

The web server didn't have permissions to access the Desktop folders of any users to retrieve the 'root' flag. Running psexec was also fruitless as it required Administrator credentials.

![alt text](https://github.com/thedukezip/hacksecurectf2016/master/elpis/images/diradam.png "dir adam")

Other observations at this point was there was a PuTTY executable located in c:\ and two primary users where the root flag may be stored, adam and Admin. FileZilla was running on the server as well. Server was running Windows 8.

![alt text](https://github.com/thedukezip/hacksecurectf2016/master/elpis/images/dirc.png "dir c")

![alt text](https://github.com/thedukezip/hacksecurectf2016/master/elpis/images/dirusers.png "dir users")

![alt text](https://github.com/thedukezip/hacksecurectf2016/master/elpis/images/tasklist.png "tasklist")

![alt text](https://github.com/thedukezip/hacksecurectf2016/master/elpis/images/tasklist.png "ver")

Using the MySQL root password provided by the Metasploit module and 0xlabs I logged into phpMyAdmin at http://192.168.131.6/etc/apps/phpmyadmin/ and verified I could also read the user flag with the SQL query 

`SELECT '1' UNION ALL SELECT LOAD_FILE("c:\\flag.txt")`

I realized this probably would have been quicker than using the RCE to read the flag - but I also verified from MySQL I could also not read the 'root' flag from c:\Users\*\Desktop. I was testing this just as 523 was looking for a query to run in phpMyAdmin to access the flags in another machine so the query came in handy for us in the end.

523 noticed there were ZPanel credentials stored in c:\zpanel\login_details.txt, which gave us the ability to log into the tool and discover two potential paths for further exploitation - the ability to add cron jobs, and the ability to add users to the FTP server.

![alt text](https://github.com/thedukezip/hacksecurectf2016/master/elpis/images/zpanelftp.png "zpanel")

I had seen the server was running FileZilla so I added a few accounts. The FTP server wasn't accessible remotely but perhaps it was available locally. The fact that a PuTTY executable had been planted also gave a hint that this was the route to pursue. We researched the ability to run a long string of FTP commands from the command line but ultimately I determined it would be best if we had an interactive web shell I could plant with the original exploit and run remote FTP commands over HTTP. I needed something that not only allowed me to run commands and display the output, but interact with a program during execution so I could cd around the directories and find the flag.

As I was trying to craft this, 523 was planting a cron job to copy files out of all the Windows user's Desktop folders into c:\ in hopes that maybe there was a Windows Service with the proper credentials running. Unfortunately as we were close on these two paths, time ran out in the CTF.

Other thoughts:

I really liked the format of getting points for obtaining a user accounts on the machine, and more points for root or Adminstrator. This allowed it to be a jeopardy style competition but provided a little more realism as you were finding a way into the machine and then escalating priviledges.

This was also my first time using an Armitage team server. Whenever I have used Metasploit in the past I typically stick to msfconsole but I could see how the ability to share scans, credentials, and shells on the hosts really could save a lot of work in a collaborative environment. I wasn't super familiar with how to best make use of the collaboration but I certainly am going to take some time to learn more about it.

We discovered we couldn't connect back to our own machines from the servers... but I discovered the firewall seemed to be letting DNS queries through by running a ping command (the pings didn't go through, but DNS resolved). We potentially could have planted a Windows EXE of Dig on Elpis and overridden the DNS server to get a reverse DNS shell running. This occurred to me much later on but something to keep in mind for the future.

![alt text](https://github.com/thedukezip/hacksecurectf2016/master/elpis/images/ping.png "ping")
