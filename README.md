# Análise de Acoplamento Arquitetural via LLM

Prompts prontos para usar com agentes de IA com acesso a terminal/arquivos (ex: [Claude Code](https://claude.com/product/claude-code), Cursor, Aider, etc.) para analisar um repositório de código e gerar automaticamente:

- Um **grafo de dependências** entre módulos/componentes/serviços
- **Métricas de acoplamento** (Fan-in, Fan-out, Instabilidade)
- Uma análise de **Main Sequence** (Instabilidade × Abstração), classificando cada módulo em Região Saudável, Zona de Dor ou Zona de Inutilidade
- Um **relatório técnico** (`relatorio.md`) com achados priorizados, evidências e recomendações acionáveis

## O problema que isso resolve

Analisar acoplamento arquitetural manualmente dá trabalho: é preciso ler código, mapear dependências, montar um grafo mental, calcular métricas e só então interpretar os sinais para decidir onde agir. Isso não escala para projetos grandes e normalmente só é feito quando o problema já está caro para resolver.

Este prompt automatiza esse fluxo — mas **não substitui** ferramentas de análise estática dedicadas (veja [Limitações](#limitações-conhecidas)). Ele é útil como primeira varredura rápida e para gerar um documento de discussão para o time.

## Conceitos aplicados

- **Acoplamento aferente (Ca) e eferente (Ce)** — quantas coisas dependem de um módulo vs. de quantas coisas ele depende
- **Instabilidade**: `I = Ce / (Ce + Ca)` — próximo de 0 é estável, próximo de 1 é instável
- **Abstração**: `A = elementos abstratos / total de elementos do módulo`
- **Main Sequence** (Robert C. Martin, *Agile Software Development: Principles, Patterns, and Practices*): a relação saudável entre estabilidade e abstração, e as duas zonas de risco:
  - 🔴 **Zona de Dor**: módulo concreto, rígido, e muito dependido — mudar é caro e arriscado
  - 🟡 **Zona de Inutilidade**: módulo muito abstrato e pouco usado — excesso de engenharia sem adoção
- **Detecção de ciclos** via DFS (busca em profundidade) com pilha de recursão

## Prompts disponíveis

| Arquivo | Uso |
|---|---|
| [`prompts/acoplamento-generico.md`](prompts/acoplamento-generico.md) | Independente de linguagem — funciona com qualquer stack |
| [`prompts/acoplamento-java-spring.md`](prompts/acoplamento-java-spring.md) | Especializado para monorepos Java/Spring (Maven/Gradle multi-módulo, Feign, Kafka/RabbitMQ, JPA) |

## Como usar

1. Abra um agente de IA com acesso a terminal e arquivos (ex: `claude` no terminal, dentro do Claude Code) na raiz do repositório que você quer analisar.
2. Copie e cole o conteúdo do prompt adequado (genérico ou Java/Spring) diretamente na conversa.
3. Deixe o agente trabalhar de forma autônoma — ele vai explorar os arquivos, rodar buscas (grep/find), montar o grafo e calcular as métricas sozinho.
4. Ao final, você terá três arquivos gerados na raiz do projeto analisado:
   - `acoplamento.json` — dados brutos do grafo (nós, arestas, evidências)
   - `acoplamento.mmd` — grafo em sintaxe [Mermaid](https://mermaid.js.org/), pronto para renderizar
   - `relatorio.md` — o relatório final consolidado, pronto para levar para discussão de arquitetura

Veja um exemplo de saída em [`exemplos/relatorio-exemplo.md`](exemplos/relatorio-exemplo.md).

## O que esperar do relatório

- Resumo executivo do estado de saúde arquitetural
- Tabela de métricas por módulo
- Gráfico Main Sequence com as 3 zonas
- Lista de ciclos de dependência detectados, com caminho completo e evidência
- Top 5 riscos priorizados, cada um com recomendação concreta de ação
- Seção explícita de limitações da própria análise

## Limitações conhecidas

Este prompt depende de análise estática via leitura de código e busca por padrões (grep/regex), executada por um LLM — não é uma ferramenta de análise de bytecode ou AST. Isso significa:

- **Não detecta** acoplamento via URL montada dinamicamente em runtime (ex: concatenação de strings, service discovery resolvido apenas em produção).
- O cálculo de **Abstração (A)** é uma aproximação heurística (proporção de interfaces/classes abstratas por módulo), não uma métrica exata — trate como indicativo, não como número absoluto.
- Em repositórios muito grandes (>25 módulos), o prompt agrupa automaticamente por domínio para não estourar contexto — o detalhamento módulo a módulo fica disponível sob pedido específico.
- A qualidade do resultado depende de quão convencional é o seu código (nomes de pacotes, anotações padrão, estrutura de build) — projetos com muitas exceções às convenções tendem a gerar mais avisos de "não confirmado".

Para análises com garantias mais fortes, considere complementar com ferramentas dedicadas:
- [ArchUnit](https://www.archunit.org/) — testa regras de arquitetura como testes automatizados
- [jdeps](https://docs.oracle.com/en/java/javase/17/docs/specs/man/jdeps.html) — análise de dependências nativa do JDK
- [Structure101](https://structure101.com/) / [CodeScene](https://codescene.com/) — ferramentas comerciais de visualização e métricas de arquitetura

## Contribuindo

Sugestões, correções e novas variações de linguagem/stack são bem-vindas via issues ou pull requests. Veja o [`CHANGELOG.md`](CHANGELOG.md) para o histórico de revisões deste prompt.

## Licença

[MIT](LICENSE) — use, adapte e distribua livremente.
