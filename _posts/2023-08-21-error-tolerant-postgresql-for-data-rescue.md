---
layout: post
title: "Error-tolerant PostgreSQL for data rescue"
tags:
- Zulip
- PostgreSQL
thumbnail_path: blog/2023-08-21-error-tolerant-postgresql-for-data-rescue/postgresql.png
---

Database corruption always happens before we prepare for it. "Back up or give up" is the most frequently recommended solution. The main reason is database engines are written presuming strict consistency of each query during processing. Could a database engine be made more error-tolerant for data rescue purposes, and always try to give some result, similarly to how a web browser does?

And it turns out the answer is yes! As a proof of concept, a PostgreSQL database has been patched to return warnings instead of fatal errors for missing attribute or data block corruption. Although some files in "/var/lib/postgres/data" were missing due to a disk failure and are replaced with earlier versions or placeholders, this patched PostgreSQL is still able to dump the database content for recovery (with lots of warnings). Good end of the story, but a kindly reminder: start your regular backups now!

Please find the patched PostgreSQL and building instructions [here](https://github.com/dmikushin/postgres-tolerant).

