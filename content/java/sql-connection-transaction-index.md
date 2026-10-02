---
title: "SQL: Connection, Transaction, and Index"
date: 2026-10-01T09:40:00-03:00
draft: false
weight: 5
tags: ["sql", "database", "postgresql", "transactions", "indexes", "series"]
categories: ["java"]
description: "The request finally touches data: a pooled connection, a transaction boundary, and the index that decides if the query takes 2ms or 2s."
summary: "The last leg out, and the first leg back. This is where the request stops being abstract and starts touching actual rows on disk."
ShowToc: true
---

Fifth post in the series — **[start from the hub]({{< ref "/java" >}})**. The [previous post]({{< ref "jvm-threads-and-spring.md" >}}) left off with a Spring service about to call the database. This is the deepest point of the journey — and where the trip back begins.

## The connection was never really "new"

When `orderService.findById(id)` executes, it doesn't open a fresh TCP connection to the database — that would mean paying the TCP handshake and authentication cost from scratch on every single request, which is far too slow to do per-request. Instead, Spring Boot's default pool, **HikariCP**, maintains a small set of already-authenticated, already-open connections and hands one out on demand:

```yaml
spring:
  datasource:
    hikari:
      maximum-pool-size: 10
      minimum-idle: 10
```

That number — 10 connections, typically — is a second hard concurrency ceiling, layered on top of Tomcat's 200-thread limit from the [previous post]({{< ref "jvm-threads-and-spring.md" >}}). It's common, and often correct, for the database pool to be *much smaller* than the web thread pool: a handful of connections, used briefly and returned quickly, can serve far more concurrent requests than that number suggests — as long as each query is fast. If queries are slow, this is usually the first place you run out of headroom, long before you run out of application threads.

## A transaction is a promise about consistency, not a performance feature

`@Transactional` on a Spring service method doesn't make anything faster — if anything, it adds overhead. What it guarantees is **atomicity**: either every statement inside that method's scope commits together, or none of them do.

```java
@Transactional
public void transferFunds(Long fromId, Long toId, BigDecimal amount) {
    Account from = accountRepository.findById(fromId).orElseThrow();
    Account to = accountRepository.findById(toId).orElseThrow();

    from.debit(amount);
    to.credit(amount);

    accountRepository.save(from);
    accountRepository.save(to);
}
```

If `to.credit(amount)` throws after `from.debit(amount)` already ran, Spring's transaction manager rolls back the entire thing — the debit never actually persists. Under the hood, this is implemented with the exact AOP proxying and dynamic bytecode generation mentioned in the [previous post]({{< ref "jvm-threads-and-spring.md" >}}): Spring wraps your bean in a proxy that opens the transaction before your method runs and commits or rolls back after it returns (or throws).

Isolation levels (`READ_COMMITTED`, `REPEATABLE_READ`, `SERIALIZABLE`) determine what a transaction is allowed to see of *other* transactions running concurrently — and this is where database-level locking can reach back and stall a thread in your connection pool, which in turn can stall a worker thread in Tomcat, which is precisely the chain of custody this whole series has been tracing.

## The index: the difference between fast and unusable

A query without the right index doesn't fail — it just reads every row to find the ones that match, a **sequential scan**:

```sql
EXPLAIN ANALYZE SELECT * FROM orders WHERE customer_id = 42;
-- Seq Scan on orders  (cost=0.00..18334.00 rows=12 width=96)
--   Filter: (customer_id = 42)
```

With an index on `customer_id`, the same query becomes a lookup in a B-tree — logarithmic instead of linear:

```sql
CREATE INDEX idx_orders_customer_id ON orders(customer_id);

EXPLAIN ANALYZE SELECT * FROM orders WHERE customer_id = 42;
-- Index Scan using idx_orders_customer_id on orders
--   (cost=0.29..8.31 rows=12 width=96)
```

On a small table the difference is invisible. On a table with ten million rows, it's the difference between a query that returns in 2 milliseconds and one that takes 2 seconds — and a 2-second query holding a connection from that 10-connection pool mentioned above is enough, under real traffic, to exhaust the pool and start queuing every other request in the application, regardless of what those requests were trying to do.

![Comic: a developer assumes a WHERE clause will be fast, the database sweats through a full sequential scan taking 2000ms, and the punchline asks whether the column was ever indexed](/img/comics/missing-index.svg)

## The trip back

Once the query returns rows, the JDBC driver maps them into Java objects, the connection is released back to the pool (not closed), the transaction commits, and your service method returns. From there, the response retraces the entire chain in reverse: Spring serializes your object to JSON, the servlet writes it to the socket Tomcat owns, the JVM's worker thread is freed back to its pool, the bytes travel back through nginx, back across the TCP/TLS connection, and finally arrive back at the `fetch()` call where this series started — where a microtask resumes a suspended `async` function, and a browser repaints a screen with a number that, a few hundred milliseconds earlier, was a row sitting quietly on disk.

```
JavaScript (browser) ← HTTP/network ← Linux (process, socket, nginx)
   ← Java (JVM, threads, Spring) ← SQL (connection, transaction, index)
```

That's the full loop. **[Back to the hub]({{< ref "/java" >}})** to read any layer again, or to follow a specific failure mode — a slow query, a thread pool exhausted, a CORS error — through the exact layer that actually causes it.
