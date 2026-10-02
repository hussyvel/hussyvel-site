---
title: "Linux"
description: "Processos, sockets, descritores de arquivo, e as decisões do kernel acontecendo por baixo de todo servidor que você coloca no ar."
summary: "A camada embaixo de todas as outras: processos, sockets, e o kernel decidindo silenciosamente o que roda onde."
ShowToc: false
---

Tudo nesse site, no fim, roda em cima de Linux — um processo com seu próprio espaço de memória, um socket que no fundo é só um descritor de arquivo, um escalonador do kernel decidindo qual thread pega a CPU agora. Esta seção cobre essa camada: gerenciamento de processos e descritores de arquivo, rede, nginx, e o dia a dia de colocar Java (e tudo o mais) em produção.
