---
title: "Linux: Processo, Socket e Nginx"
date: 2026-10-01T09:20:00-03:00
draft: false
weight: 3
tags: ["linux", "nginx", "sockets", "processos", "série"]
categories: ["java"]
description: "Como o kernel do Linux transforma um fluxo de bytes em algo que um processo de servidor consegue ler, e por que o nginx fica na frente do Java em quase todo lugar."
summary: "Antes da sua aplicação Spring ver qualquer coisa, o kernel já aceitou uma conexão, entregou pra um socket, e o nginx já decidiu o que fazer com ela."
ShowToc: true
---

Terceiro post da série — **[comece pelo hub]({{< ref "/java" >}})**. O [post anterior]({{< ref "http-and-networking.pt.md" >}}) terminou com uma conexão TCP carregando uma requisição HTTP chegando a um servidor. Este post é sobre o que acontece no instante em que ela pousa no Linux, antes de uma única linha de código Java rodar.

## Um socket é um arquivo

No Linux, quase tudo é um descritor de arquivo, e conexões de rede não são exceção. Quando um processo quer aceitar conexões, ele:

1. Cria um **socket** (`socket()`) — recebe de volta um descritor de arquivo.
2. Faz o **bind** dele a um endereço e porta (`bind()`) — por exemplo, `0.0.0.0:443`.
3. Coloca ele em **listen** (`listen()`) — avisa o kernel pra começar a enfileirar conexões que chegam.
4. **Aceita** conexões (`accept()`) — cada conexão aceita é *outro* descritor de arquivo, distinto do socket que está escutando.

Isso importa porque explica uma confusão bem comum: o socket no qual seu servidor "escuta" e o socket que ele usa pra conversar com um cliente específico não são a mesma coisa. Um único socket de escuta pode gerar milhares de sockets de conexão individuais, cada um um descritor de arquivo separado que o kernel rastreia de forma independente — buffer de leitura, buffer de escrita, estado do TCP, tudo isso.

## Por que o nginx fica na frente da sua aplicação Java

Num deploy típico (estilo SEMA, e na maioria dos setups Java em produção), o processo que de fato está vinculado à porta 443 **não é** a aplicação Java — é o nginx. Sua aplicação Spring Boot, implantada no Tomcat, geralmente escuta em algo como `localhost:8080`, inacessível diretamente pela internet. O nginx fica na frente e faz proxy reverso pra ela:

```nginx
server {
    listen 443 ssl;
    server_name api.example.com;

    location / {
        proxy_pass http://127.0.0.1:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

Isso não é burocracia por burocracia. O nginx está fazendo trabalho de verdade que a JVM faz, comparativamente, pior:

- **Terminar o TLS** — o handshake caro do [post anterior]({{< ref "http-and-networking.pt.md" >}}) acontece aqui, uma vez, em vez de dentro da JVM.
- **Servir arquivos estáticos** diretamente, sem acordar uma thread Java pra um arquivo `.css`.
- **Bufferizar clientes lentos** — o nginx consegue absorver um cliente que manda bytes aos poucos, em vez de prender uma thread do Tomcat esperando por ele.
- **Balancear carga** entre múltiplas instâncias do backend, e atuar como ponto único pra rate limiting, compressão e logs de acesso.

Do ponto de vista da JVM, toda requisição que ela recebe nesse cenário na verdade se originou como uma *conexão nova e separada vinda do nginx*, não diretamente do cliente original. É por isso que existem os headers `X-Forwarded-For` e `X-Real-IP` — sem eles, seu código Java acharia que toda requisição veio de `127.0.0.1`.

## Processos, não mágica

Tanto o nginx quanto a JVM rodando sua aplicação Spring são processos Linux comuns — visíveis no `ps`, escalonáveis pelo kernel, cada um com seu próprio espaço de memória, tabela de descritores de arquivo e conjunto de threads. Quando o nginx faz proxy de uma requisição, é literalmente isto: ler bytes do descritor de arquivo conectado ao cliente, escrever (basicamente) os mesmos bytes em outro descritor de arquivo conectado a `127.0.0.1:8080`. O escalonador do kernel decide quando as threads de cada processo realmente rodam num núcleo de CPU — não existe tratamento especial pro "servidor web" em relação a qualquer outro processo.

É também por isso que limites de recursos importam de um jeito que surpreende quem vem de linguagens de mais alto nível: um processo Linux tem um limite de descritores de arquivo abertos (`ulimit -n`), e toda conexão aberta — cliente-pro-nginx, nginx-pro-Tomcat, Tomcat-pro-banco — consome um. Bata nesse limite sob carga, e você recebe erros de `Too many open files` que não têm nada a ver com a lógica da sua aplicação.

## Pra onde a requisição vai agora

O fluxo de bytes que o nginx repassou pra `127.0.0.1:8080` agora está sentado num socket que pertence ao processo da JVM — mais especificamente, ao pool de threads do conector do Tomcat. O que acontece no instante em que o Tomcat lê esses bytes — como ele os transforma num `HttpServletRequest`, entrega pra uma thread, e como o Spring roteia o trabalho dessa thread pro seu controller — é onde o próprio modelo de concorrência da JVM assume o controle.

**Próximo: [Java e a JVM: Threads e Spring]({{< ref "jvm-threads-and-spring.pt.md" >}})**
