# Mastodon Backups Guide

A production Mastodon instance has several pieces of data that needs to be regularly backed up to protect against data
loss.

Data that needs to be backed up regularly:
* PostgreSQL database
* User generated content (images, avatars, headers)

Data that needs to be backed up at least once:
* Mastodon application secrets (see [Production Guide](../Running-Mastodon/Production-Guide.md) for more details)

In the following sub-sections, some suggestions on how to backup all of this data will be provided.