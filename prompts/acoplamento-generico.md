# Prompt: Análise de Acoplamento Arquitetural (genérico, independente de linguagem)

Cole o conteúdo abaixo diretamente na conversa com um agente de IA que tenha acesso a terminal/arquivos, na raiz do repositório que você quer analisar.

---

```
Você é um analista de arquitetura de software. Você tem acesso direto ao código deste repositório
(diretório atual = raiz do projeto). Analise o projeto — independente da linguagem de programação
usada — e produza um grafo de acoplamento entre módulos/componentes/serviços, com métricas objetivas
e recomendações de ação. Trabalhe de forma autônoma: explore os arquivos, execute comandos de busca
quando necessário, e não peça confirmação a cada passo.

## ETAPA 1 — DESCOBERTA E MAPEAMENTO
1. Identifique a(s) linguagem(ns) usada(s) e a estrutura geral do projeto (monolito modular,
   microsserviços, monorepo, etc.).
2. Defina a unidade de análise (granularidade) e declare explicitamente qual escolheu:
   - Nível de módulo/pacote (ex: pastas de primeiro/segundo nível, namespaces)
   - Nível de serviço (para arquiteturas distribuídas — cada serviço/deploy unit)
   - Nível de classe/componente (mais granular, só se o projeto for pequeno)
3. Liste todos os nós (módulos/componentes/serviços) encontrados.
4. Teto de escala: se houver mais de 25 nós candidatos, agrupe automaticamente por domínio/pasta raiz
   de primeiro nível e prossiga a análise completa sobre essa visão agregada. Informe essa decisão no
   resumo executivo e ofereça detalhamento módulo a módulo de um domínio específico "sob pedido".

## ETAPA 2 — EXTRAÇÃO DE DEPENDÊNCIAS (com evidência)
Para cada nó, identifique dependências de saída considerando todas as formas de acoplamento aplicáveis:
- Imports/includes/requires diretos no código
- Chamadas de função/método entre módulos
- Chamadas de API (REST, gRPC, GraphQL) entre serviços
- Publicação/consumo de eventos ou filas (Kafka, RabbitMQ, SQS, etc.)
- Acesso a banco de dados ou schema compartilhado
- Dependências de configuração/infraestrutura

Para CADA aresta registrada, armazene evidência rastreável: arquivo + linha + trecho do código que
originou a detecção. Uma aresta sem evidência não deve ser incluída no grafo final.

Formato de registro:
{ origem, destino, tipo, peso, evidencias: [ {arquivo, linha, trecho}, ... ] }

Limitação a declarar no relatório: acoplamento resolvido dinamicamente em runtime (URLs montadas por
concatenação, service discovery por nome resolvido apenas em produção) não é detectável por análise
estática — sinalize indícios como "possível acoplamento não confirmado".

## ETAPA 3 — CONSTRUÇÃO DO GRAFO
Gere dois arquivos:
1. `acoplamento.json` — nós e arestas (com evidências)
2. `acoplamento.mmd` — grafo em Mermaid (`graph LR`), com estilos diferentes por tipo de acoplamento
   (linha sólida = import/build; tracejada = chamada síncrona; pontilhada = evento assíncrono;
   destaque = banco/dado compartilhado — sempre tratar como crítico)

Se aplicável ao projeto, separe em duas visões: (i) grafo estático (imports/build) e (ii) grafo de
runtime (chamadas de rede + eventos). O grafo de runtime é o que alimenta a Etapa 5 (Main Sequence).

## ETAPA 4 — MÉTRICAS DE ACOPLAMENTO
Para cada nó, calcule:
- Fan-in (Ca — acoplamento aferente): quantos nós dependem dele
- Fan-out (Ce — acoplamento eferente): de quantos nós ele depende
- Instabilidade: I = Ce / (Ce + Ca)
- Grau total (Ca + Ce)
- Centralidade (se possível: betweenness e/ou PageRank)

Detecção de ciclos — algoritmo obrigatório: aplique DFS marcando nós em uma pilha de recursão
("visitando") e uma lista de finalizados ("concluído"). Uma aresta para um nó em "visitando" indica
ciclo — registre o caminho completo (A → B → C → A). Não infira ciclos por inspeção visual, mesmo em
grafos pequenos.

## ETAPA 5 — MAIN SEQUENCE (Instabilidade × Abstração)
Use o grafo de runtime (ou o único grafo disponível, se não houver distinção estática/runtime) para o
cálculo de I nesta etapa.

Para cada nó, calcule:
1. Abstração (A) = elementos abstratos do módulo / total de elementos do módulo (interfaces, classes
   abstratas, contratos vs. implementações concretas). Declare o critério usado por módulo — esta é
   uma aproximação, não um número exato.
2. Instabilidade (I), do grafo de runtime.
3. Distância da Sequência Principal: D = |A + I − 1|
4. Classifique em uma das três zonas:
   - Região Saudável: D ≤ 0.3
   - Zona de Dor: A baixo + I baixo (fan-in alto) — módulo rígido e muito dependido
   - Zona de Inutilidade: A alto + I alto (fan-in baixo) — módulo abstrato e pouco usado

Gere um scatter (Mermaid ou tabela ordenada por D) com eixo X = Instabilidade, eixo Y = Abstração,
demarcando as três zonas.

## ETAPA 6 — INTERPRETAÇÃO E DECISÃO
Responda, cruzando as métricas com o Main Sequence:
1. Quais componentes merecem alerta? (fan-in alto + instabilidade alta; qualquer módulo na Zona de Dor
   ou de Inutilidade; ciclos; acoplamento por dado/banco compartilhado — severidade máxima sempre)
2. Qual módulo acumula mais dependência? (maior fan-out = candidato a módulo/componente "deus"; maior
   fan-in = componente crítico/infraestrutural)
3. Qual parte virou ponto crítico? (fan-in alto + baixa instabilidade + alta centralidade + Zona de
   Dor = maior risco arquitetural)

Para cada achado, cite a evidência (arquivo:linha) e recomende ação concreta (extrair interface,
inverter dependência, isolar via fila/evento, criar camada de anti-corrupção, quebrar módulo).

## ETAPA 7 — GERAÇÃO DO RELATÓRIO FINAL
Gere um arquivo `relatorio.md` com a estrutura abaixo. Não resuma apenas no chat — o conteúdo completo
deve estar no arquivo. No chat, apenas confirme a geração e destaque os 2-3 achados mais críticos.

```
# Relatório de Acoplamento Arquitetural — [nome do repositório]
Data da análise: [data atual]

## 1. Resumo Executivo
## 2. Metodologia
## 3. Grafo de Dependências (embutir os .mmd em blocos ```mermaid```)
## 4. Tabela de Métricas por Nó
## 5. Main Sequence (posicionamento nas 3 zonas)
## 6. Ciclos de Dependência Detectados
## 7. Top 5 Riscos Priorizados (com evidência e recomendação)
## 8. Recomendações Gerais (backlog de débito técnico, priorizado)
## 9. Limitações da Análise
## Anexos (caminhos para acoplamento.json e acoplamento.mmd)
```

## RESTRIÇÕES
- Baseie-se apenas em evidências encontradas no código; não invente dependências.
- Toda aresta no grafo final deve ter evidência rastreável, ou ser marcada como "não confirmado".
- Se não for possível determinar granularidade automaticamente, pergunte ao usuário antes de prosseguir.
- Detecção de ciclo sempre via DFS explícita, nunca por inspeção visual.
```
