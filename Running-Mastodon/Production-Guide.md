# Mastodon Production Guide

## What is this guide?

This guide a walk through of the setup process of a Mastodon instance.

I will be using example.com for all mentions of a domain or sub-domain.
Replace all instances of example.com with your instance domain or sub-domain.

## Pre-requisites

You will need the following for this guide:
* A server running Ubuntu Server 16.04
* Root access to aforementioned server
* A domain or sub-domain to use for your instance

Disclaimer:

This guide supports Ubuntu Server 16.04 only. If you try to use this on a different
version of Ubuntu, you may run into issues.

If you are looking for a guide on a different distribution, this repository is welcoming to
contributions.

## DNS

Before we do anything on the server, add the DNS records necessary.

You will need to add:
* An A record (IPv4 address) for example.com
* An AAAA record (IPv6 address, if IPv6 connectivity is present) for example.com

## A helpful and optional note

Use `tmux` when following through with this guide.

Not only will this help you not lose your place if you disconnect, it will let you have
multiple terminal windows inside it for separate contexts (root user vs. mastodon user).

You can install `tmux` from the package manager:

```sh
apt -y install tmux
```

## Dependency installation

All of this dependency installation should be done as root.

### node.js repository addition
You will need to add a new external repository so we can have the version of node.js we
require.

Download this script:

```sh
wget https://deb.nodesource.com/setup_6.x
```

Review the script download before running:

```sh
less setup_6.x
```

Once you have reviewed the script, run it

```sh
bash setup_6.x
```

The required node.js repository is now added.

### Install all other required software and dependencies (root user)

Now we will install all the required software:

```sh
apt -y install imagemagick ffmpeg libpq-dev libxml2-dev libxslt1-dev file git curl g++ libprotobuf-dev protobuf-compiler pkg-config nodejs gcc-6 autoconf bison build-essential libssl-dev libyaml-dev libreadline6-dev zlib1g-dev libncurses5-dev libffi-dev libgdbm3 libgdbm-dev nginx redis-server redis-tools postgresql postgresql-contrib nginx letsencrypt
```

Install `yarn` from npm:

```sh
npm install -g yarn
```

### Install dependencies as the mastodon system user

As the title of this sub-section suggests, you will need to install some Mastodon dependencies
as a non-root user.

Let us create this user first:

```sh
adduser mastodon
```

Log in as the `mastodon` user:

```sh
# If you are using tmux like previously suggested you can do this in a new window 
# Ctrl-B -> Shift-:new-window
su - mastodon
```

First thing, we will need to set up `rbenv` and `ruby-build`:

```sh
git clone https://github.com/rbenv/rbenv.git ~/.rbenv
cd ~/.rbenv && src/configure && make -C src
echo 'export PATH="$HOME/.rbenv/bin:$PATH"' >> ~/.bashrc
# Restart shell
exec bash
# Check if rbenv is correctly installed
type rbenv
# Install ruby-build as rbenv plugin
git clone https://github.com/rbenv/ruby-build.git ~/.rbenv/plugins/ruby-build
```

Now that `rbenv` and `ruby-build` are installed, we will need to install the correct
Ruby version that Mastodon needs. Aforementioned Ruby version will also need to be enabled.

This is how we do that:
```sh
rbenv install 2.4.1
rbenv global 2.4.1
```

The compilation of Ruby can take some time depending on how much system resources are
available. This is a good time to go take a break, get a drink, stretch your legs, etc.

#### node.js and Ruby dependencies

Now that we have Ruby compiled and ready to go, we can clone the Mastodon git repository
and install the node.js and Ruby dependencies needed.

This is how we do that:
```sh
# Return to mastodon user's home directory
cd ~
# Clone the mastodon git repository into ~/live
git clone https://github.com/tootsuite/mastodon.git live
# Change directory to ~live
cd ~/live
# Checkout to the latest stable branch
git checkout $(git tag -l | sort -V | tail -n 1)
# Install bundler
gem install bundler
# Use bundler to install the rest of the Ruby dependencies
bundle install --deployment --without development test
# Use yarn to install node.js dependencies
yarn install --pure-lockfile
```
That is all we need to do for now with the mastodon user, you can `exit` back to root
or if using `tmux` switch back to the window where you are logged in as root.

## PostgreSQL database creation

Mastodon requires access to a PostgreSQL instance.

Let us create a user for that access:
```
# Launch psql as the postgres user
sudo -u postgres psql

# In the following prompt
CREATE USER mastodon CREATEDB;
\q
```

Note that we do not set up a password of any kind, this is because we will be using 
ident authentication. This allows local users to the database without a
password.

## nginx configuration

You will need to configure nginx to correctly serve your Mastodon instance to the world.

(Reminder: Make sure to replace all instances of example.com with your own instance's domain
sub-domain.)

Open a file in `/etc/nginx/sites-available`:

`nano /etc/nginx/sites-available/example.com.conf`

Copy and paste the following and then edit as necessary:

```nginx
map $http_upgrade $connection_upgrade {
  default upgrade;
  ''      close;
}

server {
  listen 80;
  listen [::]:80;
  server_name example.com;
  # Useful for Let's Encrypt
  location /.well-known/acme-challenge/ { allow all; }
  location / { return 301 https://$host$request_uri; }
}

server {
  listen 443 ssl http2;
  listen [::]:443 ssl http2;
  server_name example.com;

  ssl_protocols TLSv1.2;
  ssl_ciphers HIGH:!MEDIUM:!LOW:!aNULL:!NULL:!SHA;
  ssl_prefer_server_ciphers on;
  ssl_session_cache shared:SSL:10m;

  ssl_certificate     /etc/letsencrypt/live/example.com/fullchain.pem;
  ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;

  keepalive_timeout    70;
  sendfile             on;
  client_max_body_size 0;

  root /home/mastodon/live/public;

  gzip on;
  gzip_disable "msie6";
  gzip_vary on;
  gzip_proxied any;
  gzip_comp_level 6;
  gzip_buffers 16 8k;
  gzip_http_version 1.1;
  gzip_types text/plain text/css application/json application/javascript text/xml application/xml application/xml+rss text/javascript;

  add_header Strict-Transport-Security "max-age=31536000";

  location / {
    try_files $uri @proxy;
  }

  location ~ ^/(packs|system/media_attachments/files|system/accounts/avatars) {
    add_header Cache-Control "public, max-age=31536000, immutable";
    try_files $uri @proxy;
  }

  location @proxy {
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto https;
    proxy_set_header Proxy "";
    proxy_pass_header Server;

    proxy_pass http://127.0.0.1:3000;
    proxy_buffering off;
    proxy_redirect off;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection $connection_upgrade;

    tcp_nodelay on;
  }

  location /api/v1/streaming {
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto https;
    proxy_set_header Proxy "";

    proxy_pass http://127.0.0.1:4000;
    proxy_buffering off;
    proxy_redirect off;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection $connection_upgrade;

    tcp_nodelay on;
  }

  error_page 500 501 502 503 504 /500.html;
}
```

Activate the nginx config we just added:

```sh 
cd /etc/nginx/sites-enabled
ln -s ../sites-available/example.com.conf
```

This config assumes you are using Let's Encrypt as your TLS certificate provider.

If you are going to be using Let's Encrypt as your TLS certificate provider, see the 
next sub-section. If not edit the `ssl_certificate` and `ssl_certificate_key` values
accordingly.

### Let's Encrypt

This section is only relevant if you are using [Let's Encrypt](https://letsencrypt.org/)
as your TLS certificate provider.

#### Generation of certificate

We will need to generate Let's Encrypt certificates.

Make sure to replace any instances of 'example.com' with your Mastodon instance's domain.

Also make sure that nginx is stopped at this point:

```sh
systemctl stop nginx
```

We will be creating the certificate twice, once with TLS SNI validation in standalone mode
and the second time we will be using the webroot method. This is required due to the way
nginx and the letsencrypt tool works.

The TLS SNI standalone method requires nginx stopped as previously mentioned:

```sh 
letsencrypt certonly --standalone -d example.com
```

After that successfully completes, we will use the webroot method. This requires nginx
to be started and running:

```sh
systemctl start nginx
# The letsencrypt tool will ask if you want issue a new cert, please choose that option
letsencrypt certonly --webroot -d example.com -w /home/mastodon/live/public/
```

#### Automated renewal of Let's Encrypt certificate

Let's Encrypt certificates have a validity period of 90 days.

You need to renew your certificate before the expiration date. Failure to do so will
result in your users being unable to access your instance and other instances being unable 
to federate with yours.

We can do this with a cron job that runs daily:

```sh
nano /etc/cron.daily/letsencrypt-renew
```

Copy and paste this script into that file:

```sh
#!/usr/bin/env bash
letsencrypt renew
systemctl reload nginx
```

Save and exit the file.

Make the script executable and restart the cron daemon so that the script runs daily:
```sh
chmod +x /etc/cron.daily/letsencrypt-renew
systemctl restart cron
```

That is it. Your server will now automatically renew your Let's Encrypt certificate(s).

## Mastodon application configuration

Now we will need to configure the Mastodon application.

For this we will need to go back to the mastodon system user (or if you are using tmux
switch back to the window that has that user logged in):

```sh
su - mastodon
```

Change directory to ~live and edit the Mastodon application configuration:

```sh
cd ~/live
cp .env.production.sample .env.production
nano .env.production
```

For the purposes of this guide, these are the values that need to be edited:

```
# Your Redis host
REDIS_HOST=127.0.0.1
# Your Redis port
REDIS_PORT=6379
# Your PostgreSQL host
DB_HOST=/var/run/postgresql
# Your PostgreSQL user
DB_USER=mastodon
# Your PostgreSQL DB name
DB_NAME=mastodon_production
# Leave DB password empty
DB_PASS=
# Your DB_PORT
DB_PORT=5432

# Your instance's domain
LOCAL_DOMAIN=example.com
# We have HTTPS enabled
LOCAL_HTTPS=true

# Application secrets
# Generate each eith `RAILS_ENV=production bundle exec rake secret`
PAPERCLIP_SECRET=
SECRET_KEY_BASE=
OTP_SECRET=

# All SMTP details, Mailgun and Sparkpost have free tiers
SMTP_SERVER=
SMTP_PORT=
SMTP_LOGIN=
SMTP_PASSWORD=
SMTP_FROM_ADDRESS=
```

After that is complete, we will need to set up the PostgreSQL database for the first time:

```sh
RAILS_ENV=production bundle exec rails db:setup
```

And then we will need to pre-compile all CSS and JavaScript files:

```sh
RAILS_ENV=production bundle exec rails assets:precompile
```

The assets pre-compilation takes a couple minutes, so this is a good time to take
another break.