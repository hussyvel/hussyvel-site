---
title: "Java"
description: "Java como hub da jornada de uma requisição: do clique no navegador até a linha no banco de dados, e de volta."
summary: "Um clique no navegador vira uma linha num banco de dados, e vira resposta de volta na tela. Essa série acompanha essa requisição por todas as camadas, com o Java no centro."
ShowToc: false
---

Um clique acontece no navegador. Algumas centenas de milissegundos depois, uma página é atualizada com dados que, até aquele momento, só existiam como uma linha num banco de dados em outro continente. No meio do caminho, essa requisição atravessa cinco mundos completamente diferentes, cada um com suas próprias regras, formas de falhar e vocabulário.

Essa série acompanha essa jornada de ponta a ponta:

```
JavaScript (navegador) → HTTP/rede → Linux (processo, socket, nginx)
   → Java (JVM, threads, Spring) → SQL (conexão, transação, índice) → e volta
```

O Java fica bem no meio dessa cadeia, e é exatamente por isso que ele funciona como hub desta série: pra entender de verdade o que um controller do Spring está fazendo, você precisa entender o que chegou antes dele (uma conexão TCP aceita pelo kernel, intermediada por um nginx) e pra onde ele repassa o trabalho em seguida (um pool de conexões, uma transação, uma busca em índice). A maioria das explicações sobre "backend" começa e termina no código da aplicação. Esta não.

## A cadeia

1. **[JavaScript no Navegador]({{< ref "javascript-in-the-browser.pt.md" >}})** — onde a requisição nasce: `fetch`, o event loop, e por que o navegador não trava esperando a resposta.
2. **[HTTP e Rede]({{< ref "http-and-networking.pt.md" >}})** — DNS, o handshake do TCP, TLS, e como é, na prática, uma requisição HTTP em bytes no fio.
3. **[Linux: Processo, Socket e Nginx]({{< ref "linux-process-socket-nginx.pt.md" >}})** — como o kernel transforma um fluxo de bytes em algo que um processo de servidor consegue ler, e por que o nginx fica na frente do Java em quase todo lugar.
4. **[Java e a JVM: Threads e Spring]({{< ref "jvm-threads-and-spring.pt.md" >}})** — a requisição cai num thread pool, o `DispatcherServlet` faz o roteamento, e a JVM faz uma quantidade surpreendente de trabalho que você nunca pediu.
5. **[SQL: Conexão, Transação e Índice]({{< ref "sql-connection-transaction-index.pt.md" >}})** — a requisição finalmente toca os dados: uma conexão vinda de um pool, uma transação com fronteiras bem definidas, e o índice que decide se a consulta leva 2ms ou 2s.

Leia em ordem se quiser a viagem completa, ou vá direto pra camada que você está depurando hoje. Cada post assume que você sabe mais ou menos o que as outras camadas fazem, mas não os detalhes — é justamente esse o ponto de conectar tudo.
