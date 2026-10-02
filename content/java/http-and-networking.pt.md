---
title: "HTTP e Rede: A Requisição Chega ao Fio"
date: 2026-10-01T09:10:00-03:00
draft: false
weight: 2
tags: ["http", "rede", "tcp", "dns", "tls", "série"]
categories: ["java"]
description: "DNS, o handshake do TCP, TLS, e como é, na prática, uma requisição HTTP em bytes no fio."
summary: "Entre o navegador e o servidor não existe mágica nenhuma — só DNS, um handshake TCP, TLS, e um protocolo quase texto puro rodando em cima dos três."
ShowToc: true
---

Segundo post da série — **[comece pelo hub]({{< ref "/java" >}})** se você chegou direto aqui. [Da última vez]({{< ref "javascript-in-the-browser.pt.md" >}}), um navegador chamou `fetch()` e repassou o trabalho pro sistema operacional. Agora a requisição precisa de fato atravessar a rede.

## DNS: transformando um nome em endereço

`fetch("https://api.example.com/orders/42")` não sabe como chegar em `api.example.com` — TCP/IP roteia por endereço IP, não por nome de host. Então o primeiro passo é uma **consulta DNS**: o sistema operacional (ou o cache do navegador) resolve `api.example.com` pra algo como `203.0.113.10`. Isso geralmente significa uma consulta UDP a um resolver, que por sua vez pode consultar uma cadeia de nameservers se nada estiver em cache. Em produção, esse endereço quase nunca é um único servidor — normalmente é um load balancer ou a borda de uma CDN.

## TCP: o handshake antes de qualquer coisa útil acontecer

Com um endereço IP em mãos, o sistema operacional abre uma **conexão TCP** — um handshake de três vias:

```
Cliente → SYN     → Servidor
Cliente ← SYN-ACK ← Servidor
Cliente → ACK     → Servidor
```

Só depois que isso termina é que algum dado de aplicação se move. O TCP garante entrega ordenada e confiável — se um pacote se perde, ele é retransmitido; se pacotes chegam fora de ordem, eles são remontados em ordem antes que seu código sequer os veja. Essa confiabilidade tem um custo: aquele round trip antes do primeiro byte de dado real, e é por isso que o reaproveitamento de conexão (HTTP keep-alive) importa tanto pra performance — você não quer pagar o custo do handshake em toda requisição.

## TLS: criptografando a conexão

Se a URL é `https://`, outra negociação acontece em cima do TCP antes do HTTP começar: o **handshake TLS**. Cliente e servidor combinam uma cifra, o servidor apresenta um certificado (provando que ele é quem diz ser, assinado por uma autoridade certificadora em que o cliente confia), e os dois derivam uma chave simétrica compartilhada usada pra criptografar tudo que vem depois. O TLS 1.3 moderno faz isso em um round trip em vez de dois, mas ainda é overhead antes de qualquer dado HTTP fluir — mais um motivo pelo qual o reaproveitamento de conexão importa.

## HTTP: finalmente, a requisição de verdade

Depois que existe um fluxo de bytes criptografado e confiável, o navegador escreve uma requisição HTTP nele. Sem as bibliotecas no meio, é literalmente texto:

```http
GET /orders/42 HTTP/1.1
Host: api.example.com
Accept: application/json
Authorization: Bearer eyJhbGciOi...
Connection: keep-alive
```

E a resposta tem o mesmo formato, indo no sentido contrário:

```http
HTTP/1.1 200 OK
Content-Type: application/json
Content-Length: 128

{"id":42,"status":"SHIPPED","total":129.90}
```

É nessa camada que vivem as convenções REST: o método (`GET`, `POST`, `PUT`, `DELETE`) expressa intenção, o status code (`200`, `404`, `500`) expressa resultado, e os headers carregam metadados — tipo de conteúdo, tokens de autenticação, regras de cache — que os dois lados precisam combinar sem que isso faça parte do payload "de negócio".

## Pra onde esse fluxo de bytes está realmente indo

Daqui, essa conexão TCP — essa sequência exata de bytes — chega a uma máquina física ou virtual, onde o sistema operacional precisa decidir qual processo tem permissão de ler aquilo. Isso não é automático, e não faz parte do HTTP de jeito nenhum: é trabalho do kernel, e é onde o nginx normalmente entra em cena antes da sua aplicação Java ver um único byte sequer.

**Próximo: [Linux: Processo, Socket e Nginx]({{< ref "linux-process-socket-nginx.pt.md" >}})**
