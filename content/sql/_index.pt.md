---
title: "SQL"
description: "Bancos de dados, consultas, transações, e os índices que decidem se uma consulta leva 2ms ou 2s."
summary: "Onde toda requisição acaba chegando: uma conexão, uma fronteira de transação, e uma consulta que usa um índice ou não usa."
ShowToc: false
---

Toda aplicação, no fim, se resume a dados sentados num disco, e o SQL é como você traz eles de volta — ou perde eles. Esta seção cobre pool de conexões, fronteiras de transação e níveis de isolamento, e os planos de execução e índices que separam um sistema rápido de um lento, passando por Postgres, MariaDB, e o que mais aparecer pela frente.
