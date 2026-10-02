---
title: "SQL"
description: "Databases, queries, transactions, and the indexes that decide whether a query takes 2ms or 2s."
summary: "Where every request eventually ends up: a connection, a transaction boundary, and a query that either uses an index or doesn't."
ShowToc: false
---

Every application eventually comes down to data sitting on disk, and SQL is how you get it back out — or lose it. This section covers connection pooling, transaction boundaries and isolation levels, and the query plans and indexes that separate a fast system from a slow one, across Postgres, MariaDB, and whatever engine shows up next.
