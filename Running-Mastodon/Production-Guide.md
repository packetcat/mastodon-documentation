# Mastodon Production Guide

## What is this guide?

This guide a walk through of the setup process of a Mastodon instance.

I will be using example.com for all mentions of a domain or sub-domain.
Replace all instances of example.com with your instance domain or sub-domain.

## Pre-requisites

You will need the following:
* A server running Ubuntu 16.04
* Root access to aforementioned server
* A domain or sub-domain to use for your instance

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

### Install all other required software and dependencies

Now we will install all the required software:

```sh
apt -y install imagemagick ffmpeg libpq-dev libxml2-dev libxslt1-dev file git curl g++ libprotobuf-dev protobuf-compiler pkg-config nodejs gcc-6 autoconf bison build-essential libssl-dev libyaml-dev libreadline6-dev zlib1g-dev libncurses5-dev libffi-dev libgdbm3 libgdbm-dev nginx redis-server redis-tools postgresql postgresql-contrib nginx letsencrypt
```

Install `yarn` from npm:

```sh
npm install -g yarn
```