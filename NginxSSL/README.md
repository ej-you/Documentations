# How to install Nginx on server and up certbot for automatic SSL release

#### (use sudo if you are not root)

<hr>

## Initial Nginx setup

### 1. Update packages and install nginx package:
```shell
apt update
apt install nginx
```

### 2. Adjusting the Firewall
```shell
ufw app list

# output:

#  Available applications:
#  Nginx Full
#  Nginx HTTP
#  Nginx HTTPS
#  OpenSSH
```

### 3. Use needed setting from list and check status after updates:
```shell
ufw allow 'Nginx Full'
ufw status
```

### 4. If output of `sudo ufw status` is `inactive` enter the next command:
```shell
ufw enable

# output:

# Firewall is active and enabled on system startup
```

### 5. Checking your Web Server
```shell
systemctl status nginx
```

<hr>

## Initial Nginx configs setup

### 1. Add or uncomment this in `/etc/nginx/nginx.conf` in `http` directive:
```nginx configuration
#...

http {
    # ...
    include /etc/nginx/sites-enabled/*.conf;
}
```

### 2. Add `no_ssl.conf` file (filename may be another) to `/etc/nginx/sites-available/`:
```shell
touch /etc/nginx/sites-available/no_ssl.conf
```

### 3. Add symlink for `/etc/nginx/sites-available/no_ssl.conf` file to `/etc/nginx/sites-enabled/`:
```shell
ln -s /etc/nginx/sites-available/no_ssl.conf /etc/nginx/sites-enabled/no_ssl.conf
```

### Add nginx config for http (80 port) to release SSL and in other cases to redirect to https
```nginx configuration
server {
    # external incoming port
    listen 80;
    # the domain name(s) of this server
    server_name domain.ru www.domain.ru;

    # disabling display of the NGINX version in HTTP responses from the server like "Server: nginx1.20.2"
    server_tokens off;

    # location for getting SSL certificate from Let's Encrypt
    location /.well-known/ {
        root /var/www/certbot;
        try_files $uri =404;
    }

    # redirect from http to https
    location / {
        return 301 https://$host$request_uri;
    }
}
```


## Certbot set up

### 1. Install certbot:
```shell
apt update
apt install certbot
```

### 2. Make directory for certbot configs:
```shell
mkdir /var/www/certbot
chown www-data:www-data /var/www/certbot/
chmod 755 /var/www/certbot/
```

### 3. Register account (if you need)
```shell
certbot register -m your.email@gmail.com
```

### 4. Release first certificate:
> _!! Hint !!_ <br>
> You may add flag `--dry-run` to create **test request** for SSL certificate release

```shell
certbot certonly --webroot --webroot-path=/var/www/certbot/ -d domain.ru -d www.domain.ru
```


## Automatic update SSL with cron

### 1. Add cron task:
> _!! Hint !!_ <br>
> You may add flag `--dry-run` to create **test request** for SSL certificate update
> <br><br>
> (touching file in cron task use to make min log)

```shell
# update every two months
0 0 1 */2 * certbot renew && systemctl reload nginx && touch /root/certbot-logs/cert_log_$(date +"\%Y-\%m-\%d_\%H-\%M-\%S").log
```


## Https Nginx conf sample:
```nginx configuration
# backend group
upstream back {
    server 172.17.0.17:8080;
}


server {
    # external incoming port
    listen 443 ssl;
    #http2 on;
    # the domain name(s) of this server
    server_name domain.ru;

    ssl_certificate     /etc/letsencrypt/live/domain.ru/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/domain.ru/privkey.pem;

    # redirect to backend
    location / {
        proxy_pass http://back;
    }
}
```