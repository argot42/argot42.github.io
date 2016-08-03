---
layout: post
title:  "Instalar rTorrent y ruTorrent con Nginx (en debian jessie)"
ref: rtorrent-install
date:   2016-05-19 02:43:00 -0300
categories: bittorrent
tags: webserver, bittorrent, rutorrent, libtorrent, rtorrent, nginx
lang: es
--- 

## Preparar Instalación

### 1. Crear un nuevo usuario

```
sudo useradd -d /opt/rtorrent -m rtorrent
```

### 2. Instalar algunas herramientas

```
sudo apt-get install subversion git build-essential automake libtool pkg-config
```

Vas a necesitar estas para descargar los repositorios y compilarlos.

## Libtorrent+rTorrent

### 1. Instalar dependencias

```
sudo apt-get install libsigc++-2.0-dev libncurses5-dev curl libcurl4-openssl-dev libcppunit-dev
```

### 2. Instalar XMLRPC

Descarga el [repositorio][xmlrpc-c-repo] y compilalo.

```
svn checkout http://svn.code.sf.net/p/xmlrpc-c/code/stable xmlrpc-c
cd xmlrpc-c
./configure --disable-cplusplus
make
sudo make install
```

### 3. Instalar Libtorrent

Clona el [repositorio][libtorrent-repo] de Libtorrent y compilalo.

```
git clone https://github.com/rakshasa/libtorrent
cd libtorrent
./autogen.sh
./configure
make
sudo make install
```

### 4. Instalar rTorrent

Clona el [repositorio][rtorrent-repo] de rTorrent y compilalo.

```
git clone https://github.com/rakshasa/rtorrent
cd rtorrent
./autogen.sh
./configure --with-xmlrpc-c
make
sudo make install
sudo ldconfig
```

### 5. Set up rTorrent

Crea los directorios necesarios para que funcione rTorrent.

```
sudo mkdir -p /opt/rtorrent/{session,watch}
sudo chown -R rtorrent: /opt/rtorrent
```

Copia el archivo de configuración que viene con el repositorio en el home directory del usuario rTorrent.

```
sudo cp doc/rtorrent.rc /opt/rtorrent/.rtorrent.rc
```

Y modificalo como prefieras. [Acá][rtorrent-wiki-repo] vas a encontrar información que te va a ayudar a configurarlo. También vas a necesitar hacer un port forwarding de algunos puertos (revisa en el archivo de configuración para ver en que puertos esta escuchando rTorrent).

Acá un ejemplo de un archivo de configuración de rtorrent

{% highlight python %}

# SCGI
scgi_port = localhost:5000

# Maximum and minimum number of peers to connect to per torrent.
min_peers = 1
max_peers = 50

# Same as above but for seeding completed torrents (-1 = same as downloading)
min_peers_seed = 1
max_peers_seed = 50

# Maximum number of simultanious uploads per torrent.
max_uploads = 5

# Global upload and download rate in KiB. "0" for unlimited.
download_rate = 500
upload_rate = 30

# Default directory to save the downloaded torrents.
directory = /mnt/r/torrent

# Default session directory. Make sure you don't run multiple instance
# of rtorrent using the same session directory. Perhaps using a
# relative path?
session = /opt/rtorrent/session

# Watch a directory for new torrents, and stop those that have been
# deleted.
schedule = watch_directory, 5, 5, load.start=/opt/rtorrent/watch/*.torrent
schedule = untied_directory, 5, 5, stop_untied=
schedule = tied_directory, 5, 5, start_tied=

# Close torrents when diskspace is low. 
schedule = low_diskspace,5,60,close_low_diskspace=100M

# throttle schedule
# - night - #
schedule = throttle_d_off,03:00:00,24:00:00,download_rate=0
schedule = throttle_u_off,03:00:00,24:00:00,upload_rate=0

# - day - #
schedule = throttle_d_on,06:30:00,24:00:00,download_rate=500
schedule = throttle_u_on,06:30:00,24:00:00,upload_rate=30

# The ip address reported to the tracker.
#ip = 127.0.0.1
#ip = rakshasa.no

# The ip address the listening socket and outgoing connections is
# bound to.
#bind = 127.0.0.1
#bind = rakshasa.no

# Port range to use for listening.
port_range = 6881-6889

# Start opening ports at a random position within the port range.
port_random = yes

# Check hash for finished torrents. Might be usefull until the bug is
# fixed that causes lack of diskspace not to be properly reported.
check_hash = yes

# Set whether the client should try to connect to UDP trackers.
trackers.use_udp.set = yes

# Alternative calls to bind and ip that should handle dynamic ip's.
#schedule = ip_tick,0,1800,ip=rakshasa
#schedule = bind_tick,0,1800,bind=rakshasa

# Encryption options, set to none (default) or any combination of the following:
# allow_incoming, try_outgoing, require, require_RC4, enable_retry, prefer_plaintext
#
# The example value allows incoming encrypted connections, starts unencrypted
# outgoing connections but retries with encryption if they fail, preferring
# plaintext to RC4 encryption after the encrypted handshake
#
encryption = allow_incoming,enable_retry,prefer_plaintext

# Enable DHT support for trackerless torrents or when all trackers are down.
# May be set to "disable" (completely disable DHT), "off" (do not start DHT),
# "auto" (start and stop DHT as needed), or "on" (start DHT immediately).
# The default is "off". For DHT to work, a session directory must be defined.
# 
dht = auto

# UDP port to use for DHT. 
# 
dht_port = 63425

{% endhighlight %}

Ahora rTorrent debería estar funcionando. Para testearlo logeate con el usuario rTorrent (generale una contraseña con passwd si no lo hiciste) y ejecutalo.

### 6. Arrancar automáticamente rTorrent con las systemd-units

Primero instala [tmux][the-tao-of-tmux] para correr rTorrent sin necesidad de tener una terminal abierta (y poder reanudar la sesión cuando quieras).

```
sudo apt-get install tmux
```

Crea un systemd unit.

```
/etc/systemd/system/rtorrent@.service
```

{% highlight INI %}

[Unit]
Description=rTorrent
Requires=network.target local-fs.target

[Service]
Type=oneshot
RemainAfterExit=yes
KillMode=none
User=%i
ExecStart=/usr/bin/tmux new-session -s rtorrent -n rtorrent -d rtorrent
ExecStop=/usr/bin/tmux send-keys -t rtorrent:rtorrent C-q

[Install]
WantedBy=multi-user.target

{% endhighlight %}

El nombre del archivo debe ser nombre-del-servicio@.service, systemd trata a archivos con esa extensión como unidades que describe ***servicios*** y el @ que es un ***instanced service unit***. Eso, y las directivas, están explicadas [acá][systemd-service] pero básicamente podes hacer esto

```
systemctl start rtorrent@<user>
```

y correr instancias múltiples del servicio con diferentes usuarios (%i es reemplazado por el usuario después de @).

Ahora para que rTorrent arranque al inicio del sistema tenes que habilitar la unidad.

```
systemctl enable rtorrent@<user>
```

## ruTorrent

### 1. Instalar dependencias

Con rTorrent funcionando ahora instala el software necesario para rutorrent, algunos de sus plugins y Nginx

```
sudo apt-get install nginx php5 php5-cli php5-fpm php5-xmlrpc mediainfo unrar-free 
```

Clona el [repositorio][rutorrent-repo] de ruTorrent.

```
sudo git clone https://github.com/Novik/ruTorrent
```

### 2. Nginx

Mueve el repositorio clonado a donde quisieras tener tu pagina funcionando (Yo voy a usar /var/www) y cambia el propietario y el grupo de algunos directorios.

```
sudo mv rTorrent /var/www
sudo chown -R www-data: /var/www/ruTorrent/share/{setting,rtorrents}
sudo chown -R rtorrent: /var/www/ruTorrent/share/users
```

### 2.1. Crear archivo de configuración

Crea el siguiente archivo pero tené en cuenta que esta configuración no protege a rTorrent con un login por lo que cualquiera que tenga acceso a tu puerto 80 podrá usar rTorrent libremente ~~y borrar tu colección de anime~~. Si el puerto no esta abierto a internet esto no debería ser un gran problema pero sería mejor que solo lo usaras para probar tu install y luego actualices tu configuración para que coincida con la de la sección 2.2.

```
/etc/nginx/sites-available/rutorrent
```

{% highlight nginx %}

server {
    listen 80;
    root /var/www/ruTorrent;

    error_log /var/log/nginx/rutorrent_error.log;
    access_log /var/log/nginx/rutorrent_access.log;

    location /RPC2 {
        include scgi_params;
        scgi_pass 127.0.0.1:5000;
        scgi_params SCRIPT_NAME /RPC2;
    }
    
    location ~ \.php$ {
        try_files $uri =404;
        fastcgi_split_path_info ^(.+\.php)(/.+)$;
        fastcgi_pass unix:///var/run/php5-fpm.sock;
        include fastcgi_params;
        fastcgi_param SCRIPT_FILENAME $request_filename;
        fastcgi_read_timeout 300;
    }
}

{% endhighlight %}

Elimina el symlink 'default' en /etc/nginx/sites-enabled y crea uno para rutorrent

```
sudo rm /etc/nginx/sites-enabled/default
sudo ln -s /etc/nginx/sites-available/rutorrent /etc/nginx/sites-enabled/rutorrent
```

Reinicia Nginx y ya debería estar funcionando.

### 2.2. Configuración con soporte para https y autenticación

Para que tu sitio soporte una conexión segura vas a tener que generar tus propios certificados, comprar unos, o usar [Let's Encrypt][lets-encrypt-page]. 
Acá voy a generar un self-signed cert, pero vos podes elegir el que quieras.

```
sudo mkdir /etc/nginx/certs && cd /etc/nginx/certs
openssl req -x509 -newkey rsa:2048 -keyout key.pem -out cert.pem -days 365 -nodes
```

Crea un user file.

```
sudo sh -c "echo -n 'your-user-name:' >> /etc/nginx/.htpasswd"
sudo sh -c "openssl passwd -apr1 >> /etc/nginx/.htpasswd"
sudo chmod 640 /etc/nginx/.htpasswd
```

Config file de tu sitio.

```
/etc/nginx/sites-available/rutorrent
```

{% highlight nginx %}

server {
    listen 443 ssl;
    root /var/www/ruTorrent;

    error_log /var/log/nginx/rutorrent_error.log;
    access_log /var/log/nginx/rutorrent_access.log;

    ################################################
    ssl_certificate /etc/nginx/certs/cert.pem;
    ssl_certificate_key /etc/nginx/certs/key.pem; 

    location / {
        try_files $uri $uri/ =404;
        
        auth_basic "Restricted Content";
        auth_basic_user_file /etc/nginx/.htpasswd;
    }
    ################################################

    location /RPC2 {
        include scgi_params;
        scgi_pass 127.0.0.1:5000;
        scgi_params SCRIPT_NAME /RPC2;
    }
    
    location ~ \.php$ {
        try_files $uri =404;
        fastcgi_split_path_info ^(.+\.php)(/.+)$;
        fastcgi_pass unix:///var/run/php5-fpm.sock;
        include fastcgi_params;
        fastcgi_param SCRIPT_FILENAME $request_filename;
        fastcgi_read_timeout 300;
    }
}

{% endhighlight %}

Si tu servidor esta abierto a internet ["fortalecer" tu configuración][mozilla-tls-config] es una buena idea.

### 3. ruTorrent config.php

Si ruTorrent se queja de no poder acceder a un programa externo, como php o curl, tenes que agregar su path a pathToExternals en el archivo config.php.

Este es un ejemplo con el path de curl.

```
/var/www/ruTorrent/conf/config.php
```

{% highlight PHP %}

...

$pathToExternals = array(
                "php"   => '',                  // Something like /usr/bin/php. If empty, will be found in PATH.
                "curl"  => '/usr/bin/curl',                     // Something like /usr/bin/curl. If empty, will be found in PATH.
                "gzip"  => '',                  // Something like /usr/bin/gzip. If empty, will be found in PATH.
                "id"    => '',                  // Something like /usr/bin/id. If empty, will be found in PATH.
                "stat"  => '',                  // Something like /usr/bin/stat. If empty, will be found in PATH.
        );

...

{% endhighlight %}

## Referencias

* [Terminal28][terminal28-tuto] fue de gran ayuda cuando instale ruTorrent por primera vez.

* [Nginx docs][nginx-docs] 

* Y otros recursos que andan dando vuelta por Internet

Y eso es todo, ya todo debería estar funcionando.

Have fun ( ´・‿-) ~ ♥


[xmlrpc-c-repo]: http://svn.code.sf.net/p/xmlrpc-c/code/stable 
[libtorrent-repo]: https://github.com/rakshasa/libtorrent 
[rtorrent-repo]: https://github.com/rakshasa/rtorrent/wiki
[rtorrent-wiki-repo]: https://github.com/rakshasa/rtorrent/wiki
[the-tao-of-tmux]: http://tmuxp.readthedocs.io/en/latest/about_tmux.html
[systemd-service]: https://www.freedesktop.org/software/systemd/man/systemd.service.html
[rutorrent-repo]: https://github.com/Novik/ruTorrent
[lets-encrypt-page]: https://letsencrypt.org/getting-started/
[mozilla-tls-config]: https://wiki.mozilla.org/Security/Server_Side_TLS
[terminal28-tuto]: https://terminal28.com/how-to-install-and-configure-rutorrent-rtorrent-libtorrent-xmlrpc-screen-debian-7-wheezy/
[nginx-docs]: http://nginx.org/en/docs/
