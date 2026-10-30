# Changelog

## v2.0 — Revisão pós-análise crítica
- Fixado explicitamente que o grafo de **runtime** (não build) alimenta o cálculo de Instabilidade no
  Main Sequence, eliminando ambiguidade entre execuções.
- Adicionado **teto de escala**: acima de 25 nós, agregação automática por domínio, com detalhamento
  individual sob pedido.
- Adicionada exigência de **evidência rastreável** (arquivo:linha) por aresta do grafo — arestas sem
  evidência são descartadas ou marcadas como "não confirmado".
- Explicitado o **algoritmo de detecção de ciclos** (DFS com pilha de recursão), removendo dependência
  de inspeção visual do grafo.
- Adicionada **Etapa 7 — Geração do Relatório Final**, consolidando todo o output em um `relatorio.md`
  estruturado, em vez de depender apenas da resposta no chat.

## v1.1 — Incorporação do Main Sequence
- Adicionado cálculo de Abstração (A) e Distância da Sequência Principal (D = |A + I − 1|).
- Adicionadas as três zonas de design: Região Saudável, Zona de Dor, Zona de Inutilidade.
- Cruzamento das métricas de acoplamento com a posição de cada módulo no Main Sequence para
  priorização de riscos.

## v1.0 — Versão inicial
- Prompt genérico (independente de linguagem) cobrindo descoberta de estrutura, extração de
  dependências, construção de grafo (JSON + Mermaid), métricas de Fan-in/Fan-out/Instabilidade, e
  interpretação orientada a decisão.
- Especialização para monorepos Java/Spring: dependências de build (Maven/Gradle), imports, chamadas
  REST (Feign/RestTemplate/WebClient), mensageria (Kafka/RabbitMQ), banco compartilhado e configuração.
