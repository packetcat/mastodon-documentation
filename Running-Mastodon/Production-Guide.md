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
