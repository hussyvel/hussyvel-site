---
title: "Java e a JVM: Threads e Spring"
date: 2026-10-01T09:30:00-03:00
draft: false
weight: 4
tags: ["java", "jvm", "threads", "spring", "série"]
categories: ["java"]
description: "A requisição cai num thread pool, o DispatcherServlet faz o roteamento, e a JVM faz uma quantidade surpreendente de trabalho que você nunca pediu."
summary: "Este é o centro da série. A requisição finalmente chega no seu código — mas precisou de um thread pool, um dispatcher e um coletor de lixo que você nunca chama diretamente pra chegar até aqui."
ShowToc: true
---

Quarto post, e o centro desta série — **[veja a cadeia completa]({{< ref "/java" >}})**. O [post anterior]({{< ref "linux-process-socket-nginx.pt.md" >}}) parou com bytes sentados num socket que pertence ao processo da JVM. É aqui que eles finalmente viram algo parecido com "o seu código".

## O thread pool do Tomcat: a primeira decisão no nível da JVM

O Tomcat embutido do Spring Boot não cria uma nova thread do sistema operacional por requisição indefinidamente — ele mantém um **thread pool** limitado (`server.tomcat.threads.max`, padrão 200). Uma thread do conector aceita a conexão, interpreta os bytes crus em um `HttpServletRequest`, e repassa a execução pra uma thread trabalhadora desse pool.

Esse limite não é um ajuste de performance que você pode ignorar — é um teto rígido de concorrência. Se as 200 threads estiverem ocupadas (digamos, cada uma bloqueada esperando uma consulta lenta no banco), a requisição número 201 entra na fila. É exatamente por isso que uma consulta SQL lenta — do tipo que o [último post desta série]({{< ref "sql-connection-transaction-index.pt.md" >}}) trata diretamente — não deixa lenta só *aquela* requisição. Ela pode esgotar o thread pool e deixar a aplicação inteira sem resposta, até pra requisições que nem tocam no banco.

![Tirinha: um desenvolvedor em pânico porque tudo está lento, até o health check, e descobre que uma única consulta lenta está bloqueando as 200 threads do pool](/img/comics/thread-pool-exhausted.pt.svg)

## A thread da JVM, não a thread do sistema operacional

Vale ser preciso aqui: uma `Thread` do Java é, na JVM HotSpot padrão (antes do Project Loom/virtual threads), um invólucro fino em torno de uma thread nativa do sistema operacional. Quando a thread trabalhadora do Tomcat bloqueia numa chamada JDBC esperando o banco de dados, ela não está fazendo nenhum escalonamento cooperativo esperto — ela está genuinamente estacionando uma thread do SO, que o escalonador do kernel Linux simplesmente não executa até que o I/O termine. Essa é a ligação direta e nada glamourosa com o [modelo de processos e escalonamento]({{< ref "linux-process-socket-nginx.pt.md" >}}) do post anterior: o modelo de concorrência do Java, no Spring MVC tradicional (não reativo), roda inteiramente em cima de threads do sistema operacional e I/O bloqueante.

```
Thread do conector do Tomcat
  → interpreta a requisição HTTP
  → pega emprestada uma thread trabalhadora do pool
     → thread trabalhadora bloqueia numa chamada JDBC (bloqueio real, no nível do SO)
     → ... o banco de dados faz o trabalho dele ...
     → thread trabalhadora retoma, monta a resposta
  → thread trabalhadora devolvida ao pool
```

É também exatamente por isso que as **virtual threads** (Project Loom, padrão desde o Java 21) importam: elas permitem escrever o mesmo código com cara de bloqueante, mas é a JVM — não o sistema operacional — que estaciona a virtual thread, leve, e libera a thread "carregadora" real do SO pra fazer outro trabalho enquanto espera. O Spring Boot 3.2+ consegue rodar num executor de virtual threads basicamente sem mudança de código, o que transforma aquele teto rígido de 200 threads em algo bem mais elástico.

## O DispatcherServlet do Spring: uma única porta de entrada

Assim que uma thread trabalhadora tem a requisição em mãos, o `DispatcherServlet` do Spring MVC é o ponto de entrada único que decide pra onde ela vai:

```java
@RestController
@RequestMapping("/orders")
public class OrderController {

    private final OrderService orderService;

    @GetMapping("/{id}")
    public ResponseEntity<OrderDto> getOrder(@PathVariable Long id) {
        return ResponseEntity.ok(orderService.findById(id));
    }
}
```

O `DispatcherServlet` casa o `GET /orders/42` recebido com os handler mappings registrados, resolve os argumentos do método (`@PathVariable Long id` é extraído direto da URL), executa interceptors ou filters configurados (é aqui que, geralmente, autenticação, log e — fazendo a ponte com o [post sobre JavaScript]({{< ref "javascript-in-the-browser.pt.md" >}}) — os headers de CORS são aplicados), e por fim invoca o método do seu controller na thread atual. Por padrão, não existe nenhuma troca de thread adicional aqui: a mesma thread trabalhadora que o Tomcat entregou a requisição é a que executa sua lógica de negócio.

## O que a JVM faz que você não pediu

Enquanto o código do seu controller e service roda, a própria JVM está fazendo trabalho que você nunca invoca explicitamente:

- **Coleta de lixo (garbage collection)** — todo `OrderDto`, toda `String` intermediária, todo `Optional` que você aloca vira lixo no instante em que deixa de ser referenciado, e uma thread de GC (rodando concorrentemente, em coletores modernos como G1 ou ZGC) eventualmente recupera essa memória. Uma pausa de GC sob carga é uma fonte muito real e muito mensurável de picos de latência.
- **Compilação JIT** — seu bytecode começa interpretado, mas a JVM cria perfis dos métodos mais usados e os compila pra código de máquina nativo em tempo de execução (os compiladores C1/C2), o que explica em parte por que serviços Java costumam ser mais lentos nas primeiras requisições ("aquecimento") do que depois de carga sustentada.
- **Carregamento de classes** — o component scanning do Spring, a geração de proxies pra métodos `@Transactional`, e os recursos baseados em AOP acontecem via carregamento dinâmico de classes e geração de bytecode na inicialização, o que explica boa parte do tempo de startup perceptível de aplicações Spring Boot, comparado, por exemplo, a um binário Go.

## Repassando pro banco de dados

Quando `orderService.findById(id)` precisa de dados, ele não abre uma conexão TCP nova com o Postgres ou MariaDB a cada chamada — ele pega uma emprestada de um pool de conexões (o HikariCP, padrão do Spring Boot), executa o SQL nela, e devolve quando termina. Essa passagem de bastão — uma conexão vinda de um pool, uma fronteira de transação, e o índice que o banco usa (ou deixa de usar) pra atender a consulta — é o último trecho da jornada de ida.

**Próximo: [SQL: Conexão, Transação e Índice]({{< ref "sql-connection-transaction-index.pt.md" >}})**
