---
title: "SQL: Conexão, Transação e Índice"
date: 2026-10-01T09:40:00-03:00
draft: false
weight: 5
tags: ["sql", "banco de dados", "postgresql", "transações", "índices", "série"]
categories: ["java"]
description: "A requisição finalmente toca os dados: uma conexão vinda de um pool, uma transação com fronteiras bem definidas, e o índice que decide se a consulta leva 2ms ou 2s."
summary: "O último trecho de ida, e o primeiro de volta. É aqui que a requisição para de ser abstrata e passa a tocar linhas de verdade no disco."
ShowToc: true
---

Quinto post da série — **[comece pelo hub]({{< ref "/java" >}})**. O [post anterior]({{< ref "jvm-threads-and-spring.pt.md" >}}) parou com um service do Spring prestes a chamar o banco de dados. Este é o ponto mais profundo da jornada — e onde a viagem de volta começa.

## A conexão nunca foi realmente "nova"

Quando `orderService.findById(id)` executa, ele não abre uma conexão TCP nova com o banco — isso significaria pagar o custo do handshake TCP e da autenticação do zero em toda requisição, o que é lento demais pra fazer por requisição. Em vez disso, o pool padrão do Spring Boot, o **HikariCP**, mantém um conjunto pequeno de conexões já autenticadas e já abertas, e entrega uma delas sob demanda:

```yaml
spring:
  datasource:
    hikari:
      maximum-pool-size: 10
      minimum-idle: 10
```

Esse número — 10 conexões, tipicamente — é um segundo teto rígido de concorrência, em cima do limite de 200 threads do Tomcat visto no [post anterior]({{< ref "jvm-threads-and-spring.pt.md" >}}). É comum, e muitas vezes correto, que o pool do banco seja *bem menor* que o pool de threads web: um punhado de conexões, usadas rapidamente e devolvidas logo em seguida, consegue atender muito mais requisições concorrentes do que esse número sugere — desde que cada consulta seja rápida. Se as consultas são lentas, geralmente é aqui que a folga acaba primeiro, bem antes de faltar thread na aplicação.

## Uma transação é uma promessa sobre consistência, não um recurso de performance

O `@Transactional` num método de service do Spring não deixa nada mais rápido — se alguma coisa, ele adiciona overhead. O que ele garante é **atomicidade**: ou todo comando dentro do escopo daquele método é commitado junto, ou nenhum é.

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

Se `to.credit(amount)` lançar uma exceção depois que `from.debit(amount)` já rodou, o gerenciador de transações do Spring desfaz tudo — o débito nunca chega a persistir de verdade. Por baixo dos panos, isso é implementado com exatamente o mesmo mecanismo de proxy AOP e geração dinâmica de bytecode mencionado no [post anterior]({{< ref "jvm-threads-and-spring.pt.md" >}}): o Spring embrulha seu bean num proxy que abre a transação antes do seu método rodar e faz commit ou rollback depois que ele retorna (ou lança uma exceção).

Os níveis de isolamento (`READ_COMMITTED`, `REPEATABLE_READ`, `SERIALIZABLE`) determinam o que uma transação pode enxergar de *outras* transações rodando ao mesmo tempo — e é aqui que o travamento no nível do banco pode se propagar e travar uma thread do seu pool de conexões, que por sua vez pode travar uma thread trabalhadora no Tomcat, que é exatamente a cadeia de custódia que esta série inteira vem rastreando.

## O índice: a diferença entre rápido e inutilizável

Uma consulta sem o índice certo não falha — ela simplesmente lê cada linha pra encontrar as que combinam, um **sequential scan**:

```sql
EXPLAIN ANALYZE SELECT * FROM orders WHERE customer_id = 42;
-- Seq Scan on orders  (cost=0.00..18334.00 rows=12 width=96)
--   Filter: (customer_id = 42)
```

Com um índice em `customer_id`, a mesma consulta vira uma busca numa árvore-B — logarítmica em vez de linear:

```sql
CREATE INDEX idx_orders_customer_id ON orders(customer_id);

EXPLAIN ANALYZE SELECT * FROM orders WHERE customer_id = 42;
-- Index Scan using idx_orders_customer_id on orders
--   (cost=0.29..8.31 rows=12 width=96)
```

Numa tabela pequena a diferença é invisível. Numa tabela com dez milhões de linhas, é a diferença entre uma consulta que retorna em 2 milissegundos e uma que leva 2 segundos — e uma consulta de 2 segundos segurando uma conexão daquele pool de 10 mencionado acima já é suficiente, sob tráfego real, pra esgotar o pool e começar a enfileirar todas as outras requisições da aplicação, não importa o que elas estivessem tentando fazer.

![Tirinha: um desenvolvedor acha que um WHERE vai ser rápido, o banco de dados sua fazendo um sequential scan completo que leva 2000ms, e a piada final pergunta se a coluna chegou a ser indexada](/img/comics/missing-index.pt.svg)

## A viagem de volta

Assim que a consulta retorna linhas, o driver JDBC as mapeia pra objetos Java, a conexão é devolvida ao pool (não fechada), a transação é commitada, e seu método de service retorna. A partir daí, a resposta refaz toda a cadeia ao contrário: o Spring serializa seu objeto em JSON, o servlet escreve isso no socket que o Tomcat possui, a thread trabalhadora da JVM é liberada de volta pro seu pool, os bytes viajam de volta pelo nginx, de volta pela conexão TCP/TLS, e finalmente chegam de volta na chamada `fetch()` onde esta série começou — onde uma microtask retoma uma função `async` suspensa, e o navegador redesenha uma tela com um número que, algumas centenas de milissegundos antes, era uma linha quieta no disco.

```
JavaScript (navegador) ← HTTP/rede ← Linux (processo, socket, nginx)
   ← Java (JVM, threads, Spring) ← SQL (conexão, transação, índice)
```

Esse é o ciclo completo. **[Volte pro hub]({{< ref "/java" >}})** pra reler qualquer camada, ou pra seguir um modo de falha específico — uma consulta lenta, um thread pool esgotado, um erro de CORS — pela camada exata que de fato causa ele.
