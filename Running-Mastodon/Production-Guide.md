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