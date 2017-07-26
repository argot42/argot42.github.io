---
layout: post
title:  "Installing rTorrent and ruTorrent with nginx (on debian jessie)"
ref: rtorrent-install
date:   2016-05-19 02:43:00 -0300
categories: bittorrent
tags: webserver, bittorrent, rutorrent, libtorrent, rtorrent, nginx
lang: en
--- 

## Preparing to Install

### 1. Set up new user

```
sudo useradd -d /opt/rtorrent -m rtorrent
```

### 2. Install some tools

```
sudo apt-get install subversion git build-essential automake libtool pkg-config
```

You are going to need these to download the repos and compile them.

## Libtorrent+rTorrent

### 1. Install dependencies

```
sudo apt-get install libsigc++-2.0-dev libncurses5-dev curl libcurl4-openssl-dev libcppunit-dev zlib1g-dev libssl-dev
```

### 2. Install XMLRPC

Download the [repository][xmlrpc-c-repo] and compile it.

```
svn checkout http://svn.code.sf.net/p/xmlrpc-c/code/stable xmlrpc-c
cd xmlrpc-c
./configure --disable-cplusplus
make
sudo make install
```

### 3. Install Libtorrent

Clone rakshasa Libtorrent [repo][libtorrent-repo] and compile.

```
git clone -b https://github.com/rakshasa/libtorrent
cd libtorrent
./autogen.sh
./configure
make
sudo make install
```

### 4. Install rTorrent

Clone rakshasa rTorrent [repo][rtorrent-repo] and compile it.

```
git clone -b https://github.com/rakshasa/rtorrent
cd rtorrent
./autogen.sh
./configure --with-xmlrpc-c
make
sudo make install
sudo ldconfig
```

### 5. Set up rTorrent

Create directories needed.

```
sudo mkdir -p /opt/rtorrent/{session,watch}
sudo chown -R rtorrent: /opt/rtorrent
```

Copy config file that came with the source code to rTorrent's home directory.

```
sudo cp doc/rtorrent.rc /opt/rtorrent/.rtorrent.rc
```

And modify it as you see fit. You can find resources to help you setting up your config file [here][rtorrent-wiki-repo]. You Also will need to port forward the necessary ports (check in your config file on what ports did you set rTorrent to listen).

Here is an example of a rtorrent config file

{% highlight python %}

# SCGI
network.scgi.open_port = 127.0.0.1:5000
encoding.add = UTF-8

# Maximum and minimum number of peers to connect to per torrent.
throttle.min_peers.normal.set = 1
throttle.max_peers.normal.set = 50

# Same as above but for seeding completed torrents (-1 = same as downloading)
throttle.min_peers.seed.set = -1
throttle.max_peers.seed.set = -1

# Maximum number of simultanious uploads per torrent.
throttle.max_uploads.set = 5

# Global upload and download rate in KiB. "0" for unlimited.
download_rate = 500
upload_rate = 30
throttle.global_down.max_rate.set_kb = 500
throttle.global_up.max_rate.set_kb = 30

# Default directory to save the downloaded torrents.
directory.default.set = /opt/rtorrent/downloads

# Default session directory. Don't run multiple instance 
# of rtorrent using the same session directory
session.path.set = /opt/rtorrent/session

# Watch a directory for new torrents, and stop those that have been
# deleted.
schedule2 = watch_directory, 5, 5, load.start=/opt/rtorrent/watch/*.torrent
schedule2 = untied_directory, 5, 5, stop_untied=
schedule2 = tied_directory, 5, 5, start_tied=

# Close torrents when diskspace is low.
schedule2 = low_diskspace, 5, 60, close_low_diskspace=100M

# throttle schedule
# - night - #
schedule2 = throttle_d_off, 03:00:00, 24:00:00, throttle.global_down.max_rate.set_kb=0
schedule2 = throttle_u_off, 03:00:00, 24:00:00, throttle.global_up.max_rate.set_kb=0

# - day - #
schedule2 = throttle_d_on, 06:30:00, 24:00:00, throttle.global_down.max_rate.set_kb=500
schedule2 = throttle_u_on, 06:30:00, 24:00:00, throttle.global_up.max_rate.set_kb=30

# Port range to use for listening.
network.port_range.set = 6881-6889
network.port_random.set = yes

# Check hash on finished torrents.
pieces.hash.on_completion.set = yes

# Set whether the client should try to connect to UDP trackers.
trackers.use_udp.set = yes

# Encryption options, set to none (default) or any combination of the following:
# allow_incoming, try_outgoing, require, require_RC4, enable_retry, prefer_plaintext
#
# The example value allows incoming encrypted connections, starts unencrypted
# outgoing connections but retries with encryption if they fail, preferring
# plaintext to RC4 encryption after the encrypted handshake
#
protocol.encryption.set = allow_incoming,enable_retry,prefer_plaintext

# Enable DHT support for trackerless torrents or when all trackers are down.
# May be set to "disable" (completely disable DHT), "off" (do not start DHT),
# "auto" (start and stop DHT as needed), or "on" (start DHT immediately).
# The default is "off". For DHT to work, a session directory must be defined.
#
dht.mode.set = auto

# UDP port to use for DHT.
#
dht.port.set = 63425

{% endhighlight %}

Now rTorrent should be working. To test it, log into the rTorrent user (set up it up a password if you didn't) and run rTorrent.

### 6. Auto starting rTorrent with systemd-units

First install [tmux][the-tao-of-tmux] to run rTorrent without needing a terminal open (and be able to attach it later if you want to).

```
sudo apt-get install tmux
```

Create systemd unit.

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

The name of the service must be name-of-service@.service, the extension tells systemd this unit describes a ***service*** and the @ that it is a ***instanced service unit***. That, and the directives, are explained [here][systemd-service], but basically you can do this

```
systemctl start rtorrent@<user>
```

and run multiples instances of the service with different users (%i is replaced by the user after the @).

Now for rTorrent to run automatically when the system starts just enable the unit.

```
systemctl enable rtorrent@<user>
```

## ruTorrent

### 1. Install dependencies

Having rTorrent working, you will now install the sofware needed for ruTorrent, some of its plugins and nginx.

```
sudo apt-get install nginx php5 php5-cli php5-fpm php5-xmlrpc mediainfo unrar-free 
```

Clone the ruTorrent [repo][rutorrent-repo].

```
sudo git clone https://github.com/Novik/ruTorrent
```

### 2. Set up Nginx

Move rTorrent repo where you like to have your website (I'm going to use /var/www) and change the ownership of some directories.

```
sudo mv rTorrent /var/www
sudo chown -R www-data: /var/www/ruTorrent/share/{setting,rtorrents}
sudo chown -R rtorrent: /var/www/ruTorrent/share/users
```

### 2.1. Create page config 

Create the next file but be careful this configuration is not secured with password so everybody with access to your port 80 will be able to mess around with rTorrent ~~and potentially delete all your anime collection~~. If the port is not facing the internet it shouldn't be a big problem but it'll be better if you use it just for testing and eventually update your configuration to match the one in the part 2.2.

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
        scgi_pass 127.0.0.1:5000;
        include scgi_params;
    }
    
    location ~ \.php$ {
        try_files $uri =404;
        fastcgi_pass unix:///var/run/php5-fpm.sock;
        include fastcgi_params;
        fastcgi_param SCRIPT_FILENAME $request_filename;
        fastcgi_read_timeout 300;
    }
}

{% endhighlight %}

Remove the symlink on /etc/nginx/sites-enabled to the default page and create one for rutorrent

```
sudo rm /etc/nginx/sites-enabled/default
sudo ln -s /etc/nginx/sites-available/rutorrent /etc/nginx/sites-enabled/rutorrent
```

Reload nginx and it should be working by now.

### 2.2. Page config with support for https and authentication

For your site to support a secure connection you'll need to generate your own certificates, buy one, or use [Let's Encrypt][lets-encrypt-page]. Here I'll generate a self-signed cert, but you can choose whatever you like.

```
sudo mkdir /etc/nginx/certs && cd /etc/nginx/certs
openssl req -x509 -newkey rsa:2048 -keyout key.pem -out cert.pem -days 365 -nodes
```

Create a user file.

```
sudo sh -c "echo -n 'your-user-name:' >> /etc/nginx/.htpasswd"
sudo sh -c "openssl passwd -apr1 >> /etc/nginx/.htpasswd"
sudo chmod 640 /etc/nginx/.htpasswd
```

Page config file.

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
        scgi_pass 127.0.0.1:5000;
        include scgi_params;
    }
    
    location ~ \.php$ {
        try_files $uri =404;
        fastcgi_pass unix:///var/run/php5-fpm.sock;
        include fastcgi_params;
        fastcgi_param SCRIPT_FILENAME $request_filename;
        fastcgi_read_timeout 300;
    }
}

{% endhighlight %}

If your server is facing the internet it's a good idea to [harden your ssl configuration][mozilla-tls-config].

### 3. ruTorrent config.php

If ruTorrent is complaining about not been able to access an external program like php or curl you have to add its path into pathToExternals in the config.php file. 

This is an example with the curl's path.

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

## References

* [Terminal28][terminal28-tuto] was of great help the first time I installed ruTorrent.

* [Nginx docs][nginx-docs] 

* And other resources around the web

And that's it, it should be functional by now. 

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
