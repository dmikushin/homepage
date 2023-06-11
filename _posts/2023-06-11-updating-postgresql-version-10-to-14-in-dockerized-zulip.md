---
layout: post
title: "Updating PostgreSQL version 10 to version 14 in dockerized Zulip instance"
tags:
- Zulip
- PostgreSQL
thumbnail_path: blog/2023-06-11-updating-postgresql-version-10-to-14-in-dockerized-zulip/zulip.png
---

Upgrade from Zulip 5 to Zulip 6 requires updating PostgreSQL version from 10 to 14.

Upgrade is supposed to be done by following [these instructions](https://zulip.readthedocs.io/en/latest/production/upgrade.html#upgrading-postgresql). However, they are only possible, if Zulip installation is NOT dockerized. When installing [Zulip with Docker](https://github.com/zulip/docker-zulip), postgresql database runs in a separate container. Therefore, `pg_upgradecluster` command in Zulip container does not make sense and will fail, because the old database is not running locally.

So how are we supposed to upgrade the Zulip database in this case?

In order to make this possible, I've developed a database upgrade utility container based on [tianon/docker-postgres-upgrade](https://github.com/tianon/docker-postgres-upgrade). It installs PostgreSQL versions 10 and 14 simultaneously and performs an update via `pg_upgrade`. Additionally, the required `pgroonga`, `hunspell-en-us`, and Zulip stop words are installed.

Please find the utility and usage instruction here: https://github.com/dmikushin/zulip-postgres-upgrade

