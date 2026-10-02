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

O Java fica bem no meio dessa cadeia, e é exatamente por isso que ele funciona como hub aqui: pra entender de verdade o que um controller do Spring está fazendo, você precisa entender o que chegou antes dele (uma conexão TCP aceita pelo kernel, intermediada por um nginx) e pra onde ele repassa o trabalho em seguida (um pool de conexões, uma transação, uma busca em índice). A maioria das explicações sobre "backend" começa e termina no código da aplicação. Esta não vai.
