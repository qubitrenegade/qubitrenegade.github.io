---
layout: post
title:  "Configuring Matrix and Riot for private Chat"
date:   2019-07-331 19:38:21 -0700
categories: matrix synapse riot privacy chat
---

# Configuring Matrix and Riot for Private Chat

It's time to replace Slack and Discord.  While they provided chat to make the transition to a "Virtual Workplace" easier, they are the very antithesis of the decentralized web.  IRC was, [and still is](https://m.xkcd.com/1782/), an amazing technology.  However, running an IRC server, while not hard, isn't exactly easy, IRC suffers from the [Network Effect](https://en.wikipedia.org/wiki/Network_effect) making stand alone servers a bit pointless, and it's not encrypted (yes you can do encryption over the wire, but the data is unencrypted at rest).  Not to mention, you're reliant on external tools for keeping history.

Enter [Matrix](https://matrix.org/).  Matrix is "An open network for secure, decentralized communication".  Practically what this means, is Matrix attempts to solve the centralization problem by making federation easy, it solves the security problem by implementing an easy to use end-to-end encryption system, and attempts to solve the "Network Effect problem" by providing easy to use [bridging](https://matrix.org/bridges/) with existing chat platforms (including but not limited to Slack, Discord, and IRC).  It's a complete, secure, chat, VoIP, and Video calling platform.

Mattermost is another "Slack alternative" that deserves a special mention and received serious consideration before Matrix was ultimately chosen.  [Mattermost vs Matrix](https://www.slant.co/versus/12763/12764/~mattermost_vs_matrix) is really out of the scope of this conversation.  (Though I will say you absolutely _can_ edit posts with Matrix)

## Introduction

Matrix is a protocol. Which is defined by the [Matrix Specification](https://matrix.org/docs/spec/).  Matrix defines client-to-server, server-to-server, and client-to-client interactions.  What this means to the administrator/user is there is [potentially] a variety of server and client software to chose from.  The [reference implementation](https://en.wikipedia.org/wiki/Reference_implementation) known as [Synapse](https://github.com/matrix-org/synapse) and [Riot.im](https://about.riot.im/) appear to be the defacto standard server at client, at the time of this writing.

Synapse has some [great docs](https://github.com/matrix-org/synapse/blob/master/README.rst) which cover nearly every scenario.  There's also some [great guides](https://gist.github.com/attacus/cb5c8a53380ca755b10a5b37a636a0b9) to help get started.

In short we need to install and configure:

* [Synapse](https://github.com/matrix-org/synapse)
* [Riot Web](https://github.com/vector-im/riot-web)
* [NGINX](https://nginx.org/en/)
* [PostgreSQL](https://www.postgresql.org/)
* [CertBot](https://certbot.eff.org/)
* [TURN](https://en.wikipedia.org/wiki/Traversal_Using_Relays_around_NAT) Server (we'll use [coturn](https://github.com/coturn/coturn))

This is an attempt to collate all the information into a single procedure based on the information learned before it is forgotten.

This guide is written as if all applications are running on the same server, however, each service can be run in its own VM/container/etc.

## Prerequisites

* Minimum Virtual Machine
  * 1 CPU
  * 1 GB Ram
  * 25 GB root `/` 
  * 10 GB `/var/lib/postgresql` `chown postgres:postgres /var/lib/postgresql` XFS, mounted `noatime`
* Free CloudFlare account
  * DNS Subdomain Entries, recommend both A and AAAA entries if IPv6 is supported:
    * `chat` - Riot Web address
    * `matrix` - Matrix Server Address
    * `turn` - TURN Server address

To make it easier on ourselves since we're using CloudFlare, we'll leverage the [certbot-dns-cloudflare](https://certbot-dns-cloudflare.readthedocs.io/en/stable/) CertBot plugin for CertBot authentication.  

## Install Packages

### Add Matrix.org Repo

```bash
wget -O /usr/share/keyrings/matrix-org-archive-keyring.gpg https://packages.matrix.org/debian/matrix-org-archive-keyring.gpg
sh -c 'echo "deb [signed-by=/usr/share/keyrings/matrix-org-archive-keyring.gpg] https://packages.matrix.org/debian/ $(lsb_release -cs) main" |  sudo tee /etc/apt/sources.list.d/matrix-org.list'
```

### Add CertBot Repo

```bash
add-apt-repository ppa:certbot/certbot
add-apt-repository universe
```

### Add PostgreSQL Repo

```bash
wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add -
sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt/ `lsb_release -cs`-pgdg main" >> /etc/apt/sources.list.d/pgdg.list'
```

### Package List

#### Synapse

```bash
apt install matrix-synapse-py3 nginx certbot python-certbot-nginx python3-certbot-dns-cloudflare
```

#### PostgreSQL

[Common and/or Contrib](https://stackoverflow.com/questions/37128058/difference-between-postgresql-common-and-postgresql-contrib) you be the judge.

```bash
apt install postgresql postgresql-contrib postgresql-common
```

#### Riot Web

```bash
apt install nginx certbot python-certbot-nginx python3-certbot-dns-cloudflare
```

#### TURN Server

```bash
apt install coturn certbot python3-certbot-dns-cloudflare
```

#### All in one

```bash
apt install matrix-synapse-py3 coturn \
    postgresql postgresql-contrib postgresql-common \
    nginx certbot python-certbot-nginx python3-certbot-dns-cloudflare \
    libjemalloc
```

[`libjemalloc`](http://jemalloc.net/) is optional, results are unclear if it actually helps memory usage in Python3.

## Setup CertBot

### CloudFlare Credentials

Create an `/etc/certbot-cloudflare.ini`.  N.b.: you currently must use the "[CloudFlare Global API Key](https://support.cloudflare.com/hc/en-us/articles/200167836-Where-do-I-find-my-Cloudflare-API-key-)" for `certbot` to be able to authenticate.

```bash
dns_cloudflare_email = "your-cloudflare-email@protonmail.com"
dns_cloudflare_api_key = "<GLOBAL API KEY>"
```

### Run Certbot

CertBot will use the first domain listed when generating the certificate path.

```bash
certbot certonly -a dns-cloudflare --dns-cloudflare-credentials /etc/certbot-cloudflare.ini \
  --expand -d example.com,www.example.com,matrix.example.com,chat.example.com,turn.example.com
```

This will create a timer which can be validated with `systemctl list-timers`

```
# systemctl list-timers -all | grep certbot
Thu 2019-08-01 04:41:16 UTC  14min left    Wed 2019-07-31 21:19:33 UTC  7h ago      certbot.timer                certbot.service
```

## Configure PostgreSQL

See [Tuning](https://github.com/matrix-org/synapse/blob/master/docs/postgres.rst#tuning-postgres) for info on tuning PgSQL.

### Create Database User

```bash
sudo -u postgres createuser --pwprompt synapse_user
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

## Configure Coturn

### Coturn Config

Create a `/etc/turnserver.conf`.  This is the minimal options required.

{% gist a16780803bb9662c7825ce9e5becd8fa turnserver.conf %}

### Enable Coturn

```
sh -c 'echo TURNSERVER_ENABLED=1 >> /etc/default/coturn'
```

### Start Coturn

```
systemctl enable coturn # this might actually not be required because it still uses sysv init...
systemctl start coturn
```

## Install Riot Web

### Download latest Release

The latest release can be found [here](https://github.com/vector-im/riot-web/releases)

```
wget -O- https://github.com/vector-im/riot-web/releases/download/v1.3.0/riot-v1.3.0.tar.gz | tar vzxf - -C /var/www/html
```

### Link to Riot root dir

We'll use `/var/www/html/riot` as the DocumentRoot of our `chat.example.com` domain.  This makes upgrading (or downgrading) in the future easier.  e.g.: we can upgrade without restarting NGINX.

```
ln -s /var/www/html/riot-v1.3.0 /var/www/html/riot
```

## Configure Synapse

### Server Name Config

We don't want our name to show up as `@foo:matrix.example.com`, so we'll update `/etc/matrix-synapse/conf.d/server_name.yaml` to point our server name to `example.com`.

```
server_name: example.com
```

### Homeserver Config File

This is an opinionated `/etc/matrix-synapse/homeserver.yaml` file and by no means covers every config option.  (We'll look at how we can break this into multiple smaller files in a future post)

{% gist a16780803bb9662c7825ce9e5becd8fa homeserver.yaml %}

### Optional Tuning

`jemalloc` purportedly reduced memory usage in Python2 up to 40% in some cases.  However, it is unclear how `jemalloc` affects Python3 memory usage.  If any instability is encountered, this would be the first thing to undo.

```
sh -c 'echo "LD_PRELOAD=/usr/lib/x86_64-linux-gnu/libjemalloc.so.1" | tee -a /etc/default/matrix-synapse'
```

### Start Synapse

```
systemctl start matrix-synapse
```

### Verify our Setup by registering a new user

Execute the `register_new_matrix_user` script to register a new user and validate our Synapse configuration is operational.

```
register_new_matrix_user -c /etc/matrix-synapse/homeserver.yaml http://localhost:8008
```

### Troubleshooting

If any errors are encounterd with the registration of a new user, refer to `/var/log/matrix-synapse/homeserver.log` and `systemctl status matrix-synapse` and `journalctl -u matrix-synapse`

## Configure NGINX

We need to configure three sites, the "default" page, or the `example.com` main site, `chat.example.com` where we'll host the Riot Web app, and `matrix.example.com` which is where Synapse is running.

We'll mimic the Apache method of creating our site configs in `/etc/nginx/sites-available` and linking them to `/etc/nginx/sites-enabled/`.  This makes enabling/disabling sites easy for purposes of scaling.

### Setup Sites

#### Default - `example.com`

In many cases, this will already be configured and running a site.  We need to be able to serve two files from `example.com/.well-known/matrix`, the below config identifies a SIMPLE configuration.  The most important part for the operation of Synapse/Riot is the location 

{% gist a16780803bb9662c7825ce9e5becd8fa nginx-default %}

#### Riot Web - `chat.example.com`

Nothing too fancy here, we're just serving the static files, all the magic happens in the client.

{% gist a16780803bb9662c7825ce9e5becd8fa nginx-chat %}

#### Synapse - `matrix.example.com`

{% gist a16780803bb9662c7825ce9e5becd8fa nginx-gist %}

### Create "well known" files

#### `/var/www/html/.well-known/matrix/client`

```
{
    "m.homeserver": {
        "base_url": "https://matrix.example.com"
    },
    "m.identity_server": {
        "base_url": "https://vector.im"
    }
}
```

#### `/var/www/html/.well-known/matrix/server`

```
{ "m.server": "matrix.example.com:443" }
```

### Restart Nginx

```
systemctrl restart nginx
```

## Firewall Rules

### UFW Setup

Update `/etc/services`

```
# grep turn /etc/services 
stun-turn       3478/tcp                        # Coturn
stun-turn       3478/udp                        # Coturn
stun-turn-tls   5349/tcp                        # Coturn
stun-turn-tls   5349/udp                        # Coturn
turnserver-cli  5766/tcp                        # Coturn
```

Enable UFW

```
ufw allow ssh
ufw allow http
ufw allow https
ufw allow stun-turn
ufw allow stun-turn-tls
ufw enable
```

### Digital Ocean Rules

This terraform sets up firewall rules to block all communication to the server on 80 and 443 except for traffic from CloudFlare servers.

{% gist a16780803bb9662c7825ce9e5becd8fa tf-digital-ocean-rules.tf %}

N.b. CloudFlare doesn't proxy TURN traffic, so we recommend running your TURN server on a separate host.

## Conclusion

At this point you should have a running and functional 

Next up, we'll look at how to automate deploying Synapse, Riot, and Coturn.
