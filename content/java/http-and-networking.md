---
title: "HTTP and Networking: The Request Hits the Wire"
date: 2026-10-01T09:10:00-03:00
draft: false
weight: 2
tags: ["http", "networking", "tcp", "dns", "tls", "series"]
categories: ["java"]
description: "DNS, TCP's handshake, TLS, and what an HTTP request actually looks like as bytes on the wire."
summary: "Between the browser and the server there's no magic — just DNS, a TCP handshake, TLS, and a plain-text-ish protocol riding on top of all three."
ShowToc: true
---

Second post in the series — **[start from the hub]({{< ref "/java" >}})** if you landed here directly. [Last time]({{< ref "javascript-in-the-browser.md" >}}), a browser called `fetch()` and handed the work to the OS. Now the request has to actually cross the network.

## DNS: turning a name into an address

`fetch("https://api.example.com/orders/42")` doesn't know how to reach `api.example.com` — TCP/IP routes by IP address, not by hostname. So the first step is a **DNS lookup**: the OS (or browser cache) resolves `api.example.com` to something like `203.0.113.10`. This usually means a UDP query to a resolver, which may itself query a chain of nameservers if nothing is cached. In production this address is almost never a single server — it's typically a load balancer or the edge of a CDN.

## TCP: the handshake before anything useful happens

With an IP address in hand, the OS opens a **TCP connection** — a three-way handshake:

```
Client → SYN     → Server
Client ← SYN-ACK ← Server
Client → ACK     → Server
```

Only after this completes does any application data move. TCP guarantees ordered, reliable delivery — if a packet is lost, it's retransmitted; if packets arrive out of order, they're reassembled in order before your code ever sees them. This reliability has a cost: that round trip before the first byte of real data, which is why connection reuse (HTTP keep-alive) matters so much for performance — you don't want to pay the handshake cost on every request.

## TLS: encrypting the connection

If the URL is `https://`, another negotiation happens on top of TCP before HTTP starts: the **TLS handshake**. Client and server agree on a cipher suite, the server presents a certificate (proving it is who it claims to be, signed by a certificate authority the client trusts), and they derive a shared symmetric key used to encrypt everything that follows. Modern TLS 1.3 does this in one round trip instead of two, but it's still overhead before any HTTP data flows — another reason connection reuse matters.

## HTTP: finally, the actual request

Once there's an encrypted, reliable byte stream, the browser writes an HTTP request onto it. Stripped of the libraries, it's genuinely just text:

```http
GET /orders/42 HTTP/1.1
Host: api.example.com
Accept: application/json
Authorization: Bearer eyJhbGciOi...
Connection: keep-alive
```

And the response looks the same shape, going the other way:

```http
HTTP/1.1 200 OK
Content-Type: application/json
Content-Length: 128

{"id":42,"status":"SHIPPED","total":129.90}
```

This is the layer where REST conventions live: the method (`GET`, `POST`, `PUT`, `DELETE`) expresses intent, the status code (`200`, `404`, `500`) expresses outcome, and headers carry metadata — content type, auth tokens, caching rules — that both ends need to agree on without it being part of the "business" payload.

## Where this stream of bytes is actually going

From here, this TCP connection — this exact sequence of bytes — arrives at a physical or virtual machine, where the operating system has to decide which process even gets to read it. That's not automatic, and it's not part of HTTP at all: it's the kernel's job, and it's where nginx typically enters the picture before your Java application ever sees a single byte.

**Next: [Linux: Process, Socket, and Nginx]({{< ref "linux-process-socket-nginx.md" >}})**
