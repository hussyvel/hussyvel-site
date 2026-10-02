---
title: "JavaScript no Navegador: Onde a Requisição Começa"
date: 2026-10-01T09:00:00-03:00
draft: false
weight: 1
tags: ["javascript", "navegador", "http", "série"]
categories: ["java"]
description: "Toda requisição de backend começa com uma linha de JavaScript num navegador. Veja o que acontece antes de qualquer byte sair pela rede."
summary: "Antes do HTTP, antes do Java, antes do SQL — existe uma chamada fetch() parada num event loop. É aqui que a jornada da nossa requisição começa."
ShowToc: true
---

Este é o primeiro post de uma série que acompanha uma única requisição por cinco camadas: **[a cadeia completa está aqui]({{< ref "/java" >}})**. Começamos onde a requisição começa — não num servidor, mas numa aba de navegador, em JavaScript.

## A ilusão de uma chamada bloqueante

Um código como este parece uma chamada simples e síncrona:

```javascript
const response = await fetch("/api/orders/42");
const order = await response.json();
renderOrder(order);
```

Não é. O JavaScript no navegador roda numa **única thread**. Se o `fetch` realmente bloqueasse essa thread enquanto espera um servidor do outro lado do mundo, a página inteira — rolagem, cliques, animações — congelaria pelo tempo que a rede demorasse. Isso não acontece, porque o `fetch` não bloqueia nada.

## O event loop, em resumo

O motor de JavaScript do navegador tem uma única call stack e um único event loop. Quando você chama `fetch`, o motor não faz a parte de rede ali mesmo, inline — ele repassa a requisição pra APIs do navegador que vivem fora do motor de JS (escritas em C++, rodando em suas próprias threads), e devolve imediatamente uma `Promise`. Sua função continua executando, a stack esvazia, e a renderização e o tratamento de eventos do navegador continuam sem interrupção.

Quando a camada de rede finalmente recebe uma resposta — bytes chegando por um socket que o sistema operacional entregou ao processo do navegador, que é exatamente o assunto dos [próximos dois posts]({{< ref "http-and-networking.pt.md" >}}) desta série — ela também não volta direto pro seu código. Ela enfileira uma **microtask** (no caso de promises) que só executa quando a call stack estiver vazia. É por isso que o `await` "pausa" sua função assíncrona sem nunca bloquear a thread: por baixo dos panos, é só um callback de promise agendado pra depois.

```
fetch() chamado
  → motor JS repassa pra pilha de rede do navegador, devolve uma Promise
  → call stack esvazia, navegador continua responsivo
  → ... tempo passa, handshake TCP/TLS, troca HTTP acontece fora da thread ...
  → resposta chega → microtask é enfileirada
  → event loop pega a microtask → seu `await` é retomado
```

![Tirinha: um desenvolvedor insiste que o fetch() não funciona, enquanto o event loop responde calmamente que a promise resolveu há 4ms — ele só esqueceu o await](/img/comics/js-event-loop.pt.svg)

## O que o `fetch` realmente dispara

Chamar `fetch("/api/orders/42")` inicia uma cadeia que esta série acompanha camada por camada:

1. O navegador precisa de um endereço IP pro host — uma **consulta DNS**, possivelmente em cache.
2. Ele abre uma **conexão TCP** com esse IP na porta 443 (ou reaproveita uma conexão que já tem, via keep-alive/pool de conexões do próprio navegador).
3. Ele faz um **handshake TLS**, se a URL for `https://`.
4. Ele serializa sua requisição em **HTTP** de verdade — método, headers, corpo — e escreve isso no socket.

Tudo isso está no [próximo post, sobre HTTP e rede]({{< ref "http-and-networking.pt.md" >}}). Do ponto de vista do navegador, nada disso é tratado como caso especial — são só bytes saindo por um socket que o sistema operacional gerencia pro processo do navegador, igual a qualquer outro cliente de rede.

## CORS: o próprio porteiro do navegador

Uma coisa que é exclusiva da camada do navegador — nenhuma camada seguinte lida com isso — é o **CORS** (Cross-Origin Resource Sharing). Se o seu JavaScript, servido a partir de `https://app.example.com`, chama `fetch("https://api.example.com/orders")`, é o próprio navegador quem garante que a API permita explicitamente aquela origem através do header de resposta `Access-Control-Allow-Origin`. Essa verificação acontece inteiramente do lado do cliente; ela existe pra proteger *usuários*, não servidores — um site malicioso não consegue ler respostas da API do seu banco em seu nome só porque seu navegador está autenticado lá.

Isso importa pro resto da cadeia porque uma aplicação Spring Boot (à qual vamos chegar) precisa configurar ativamente as regras de CORS, ou clientes baseados em navegador vão ter requisições silenciosamente bloqueadas — mesmo que o servidor tenha respondido com um 200 perfeitamente válido.

## O que vem a seguir

O navegador já repassou sua requisição pra pilha de rede do sistema operacional. A próxima parada é o fio em si: resolução de DNS, o handshake TCP, TLS, e como é, na prática, uma requisição HTTP em bytes crus.

**Próximo: [HTTP e Rede]({{< ref "http-and-networking.pt.md" >}})**
