---
layout: post
title:  "Configuring Matrix and Riot for private Chat"
date:   2019-07-16 19:38:21 -0700
categories: matrix synapse riot privacy chat
---

ROUGH outline of things to come

# Packages, we need some stinking pakcages

```
   add-apt-repository ppa:certbot/certbot
   add-apt-repository universe
   
   wget -O /usr/share/keyrings/matrix-org-archive-keyring.gpg https://packages.matrix.org/debian/matrix-org-archive-keyring.gpg
   sh -c 'echo "deb [signed-by=/usr/share/keyrings/matrix-org-archive-keyring.gpg] https://packages.matrix.org/debian/ $(lsb_release -cs) main" |     sudo tee /etc/apt/sources.list.d/matrix-org.list'
   wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add -
   sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt/ `lsb_release -cs`-pgdg main" >> /etc/apt/sources.list.d/pgdg.list'
   apt install matrix-synapse-py3 nginx certbot python-certbot-nginx matrix-synapse-py3 postgresql postgresql-contrib
   # common or contrib you decide: https://stackoverflow.com/questions/37128058/difference-between-postgresql-common-and-postgresql-contrib
```


## Set things up

### Create Database User

```bash
# Database
# Tuning? https://github.com/matrix-org/synapse/blob/master/docs/postgres.rst#tuning-postgres
sudo -u postgres createuser --pwprompt synapse_user # take pw too?
```

### Create Database

```SQL
sudo -u postrees psql -c '
  CREATE DATABASE synapse
    ENCODING 'UTF8'
    LC_COLLATE='C'
    LC_CTYPE='C'
    template=template0
    OWNER synapse_user;'
```

## Create Webserver

### Setup domains

```
# /etc/nginx/sites-enabled/chat
server {
  gzip on;

  server_name chat.qbrd.cc;
  location / {
    proxy_pass http://localhost:8008;
    proxy_set_header X-Forwarded-For $remote_addr;
  }

    listen [::]:443 ssl; # managed by Certbot
    listen 443 ssl; # managed by Certbot
    ssl_certificate /etc/letsencrypt/live/qbrd.cc-0001/fullchain.pem; # managed by Certbot
    ssl_certificate_key /etc/letsencrypt/live/qbrd.cc-0001/privkey.pem; # managed by Certbot
    include /etc/letsencrypt/options-ssl-nginx.conf; # managed by Certbot
    ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem; # managed by Certbot

}
server {
    if ($host = chat.qbrd.cc) {
        return 301 https://$host$request_uri;
    } # managed by Certbot



  listen 80;
  listen [::]:80;

  server_name chat.qbrd.cc;
    return 404; # managed by Certbot


}# /etc/nginx/sites-enabled/default
server {

    root /var/www/html;

    index index.html;

    server_name qbrd.cc
                www.qbrd.cc;

    location /.well-known/matrix/ {
        add_header 'Content-Type' 'application/json';
        add_header 'Access-Control-Allow-Origin' '*';
        add_header 'Access-Control-Allow-Methods' 'GET, POST, OPTIONS';
        add_header 'Access-Control-Allow-Headers' 'DNT,User-Agent,X-Requested-With,If-Modified-Since,Cache-Control,Content-Type,Range';
        add_header 'Access-Control-Expose-Headers' 'Content-Length,Content-Range';

        autoindex on;
        autoindex_exact_size on;
        autoindex_format json;
        autoindex_localtime on;
    }

    # location /.well-known/matrix/server {
    #     add_header 'Content-Type' 'application/json';
    #     autoindex on;
    # }

    # location /_matrix {
    #     return 301 https://matrix.qbrd.cc$request_uri;
    # }

    location / {
        # First attempt to serve request as file, then
        # as directory, then fall back to displaying a 404.
        try_files $uri $uri/ =404;
    }

    listen [::]:443 ssl ipv6only=on; # managed by Certbot
    listen 443 ssl; # managed by Certbot
    ssl_certificate /etc/letsencrypt/live/qbrd.cc-0001/fullchain.pem; # managed by Certbot
    ssl_certificate_key /etc/letsencrypt/live/qbrd.cc-0001/privkey.pem; # managed by Certbot
    include /etc/letsencrypt/options-ssl-nginx.conf; # managed by Certbot
    ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem; # managed by Certbot
}

server {
    if ($host = www.qbrd.cc) {
        return 301 https://$host$request_uri;
    } # managed by Certbot


    if ($host = qbrd.cc) {
        return 301 https://$host$request_uri;
    } # managed by Certbot


    listen 80 default_server;
    listen [::]:80 default_server;

    server_name qbrd.cc
                www.qbrd.cc
                _;
    return 404; # managed by Certbot
}

# /etc/nginx/sites-enabled/matrix
server {
  gzip off;

  server_name matrix.qbrd.cc;
  location / {
    proxy_pass http://localhost:8008;
    proxy_set_header X-Forwarded-For $remote_addr;
  }

    listen [::]:443 ssl; # managed by Certbot
    listen 443 ssl; # managed by Certbot
    ssl_certificate /etc/letsencrypt/live/qbrd.cc-0001/fullchain.pem; # managed by Certbot
    ssl_certificate_key /etc/letsencrypt/live/qbrd.cc-0001/privkey.pem; # managed by Certbot
    include /etc/letsencrypt/options-ssl-nginx.conf; # managed by Certbot
    ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem; # managed by Certbot

}
server {
    if ($host = matrix.qbrd.cc) {
        return 301 https://$host$request_uri;
    } # managed by Certbot



  listen 80;
  listen [::]:80;

  server_name matrix.qbrd.cc;
    return 404; # managed by Certbot


}
```



```
# /var/www/html/.well-known/matrix/server 
{
  "m.server": "matrix.qbrd.cc:443"
}
```

```
# cat /var/www/html/.well-known/matrix/client 
{
    "m.homeserver": {
        "base_url": "https://matrix.qbrd.cc"
    },
    "m.identity_server": {
        "base_url": "https://vector.im"
    }
}
```

    certbot --nginx
    
### Riot App

Download to `/var/www/riot-version`, link to `/var/www/riot`

```
{
    "default_server_config": {
        "m.homeserver": {
            "base_url": "https://matrix.qbrd.cc",
            "server_name": "qbrd.cc"
        },
        "m.identity_server": {
            "base_url": "https://vector.im"
        }
    },
    "disable_identity_server": true,
    "disable_custom_urls": false,
    "disable_guests": false,
    "disable_login_language_selector": false,
    "disable_3pid_login": true,
    "brand": "QBRD - Riot",
    "branding": {
      "welcomeBackgroundUrl": "img/custom/bg-stars.jpg"
    },
    "integrations_ui_url": "https://scalar.vector.im/",
    "integrations_rest_url": "https://scalar.vector.im/api",
    "integrations_jitsi_widget_url": "https://scalar.vector.im/api/widgets/jitsi.html",
    "bug_report_endpoint_url": "https://riot.im/bugreports/submit",
    "defaultCountryCode": "US",
    "showLabsSettings": true,
    "features": {
        "feature_groups": "labs",
        "feature_pinning": "labs"
    },
    "default_federate": true,
    "default_theme": "dark",
    "roomDirectory": {
        "servers": [
            "matrix.qbrd.cc"
        ]
    },
    "welcomeUserId": "@riot-bot:matrix.org",
    "piwik": {
        "url": "https://piwik.riot.im/",
        "whitelistedHSUrls": ["https://matrix.org"],
        "whitelistedISUrls": ["https://vector.im", "https://matrix.org"],
        "siteId": 1
    },
    "enable_presence_by_hs_url": {
        "https://matrix.org": false
    }
}
```

## Setup TURN server, coturn

```
# cat /etc/turnserver.conf 
listening-port=3478
tls-listening-port=5349
listening-ip=IPv6
listening-ip=IPv4
listening-ip=IPv4local-address
use-auth-secret
static-auth-secret=<generated password>
server-name=turn.qbrd.cc
realm=turn.qbrd.cc
user-quota=12
total-quota=1200
no-tcp-relay
cert=/etc/letsencrypt/live/qbrd.cc/fullchain.pem
pkey=/etc/letsencrypt/live/qbrd.cc/privkey.pem
dh-file=/etc/letsencrypt/ssl-dhparams.pem
denied-peer-ip=10.0.0.0-10.255.255.255
denied-peer-ip=192.168.0.0-192.168.255.255
denied-peer-ip=172.16.0.0-172.31.255.255
allowed-peer-ip=IPv4local-address
``` 


## Reference

* 1 - [Matrix README](https://github.com/matrix-org/synapse/blob/master/README.rst)
* x - [install docs](https://github.com/matrix-org/synapse/blob/master/INSTALL.md)
* 2 - [setup workshop](https://gist.github.com/attacus/cb5c8a53380ca755b10a5b37a636a0b9)
* 3 - [PgSQL Apt setup](https://wiki.postgresql.org/wiki/Apt)
* 3 - [PgSQL Setup](https://github.com/matrix-org/synapse/blob/master/docs/postgres.rst)
* x - [Federation](https://github.com/matrix-org/synapse/blob/master/docs/federate.md)
* x - [Ubuntu Certbot](https://certbot.eff.org/lets-encrypt/ubuntubionic-nginx.html)

