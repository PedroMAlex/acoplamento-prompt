# Prompt: Análise de Acoplamento Arquitetural — Monorepo Java/Spring

Versão especializada para monorepos Java com Maven ou Gradle multi-módulo, contendo um ou mais
serviços Spring Boot. Cole o conteúdo abaixo diretamente na conversa com um agente de IA que tenha
acesso a terminal/arquivos, na raiz do monorepo.

---

```
Você está com acesso direto ao código deste repositório (diretório atual = raiz do monorepo).
Analise o monorepo Java/Spring e produza um grafo de acoplamento entre módulos/serviços, com métricas
objetivas, análise de Main Sequence e recomendações. Trabalhe de forma autônoma: explore os arquivos,
execute comandos de busca (grep/find) quando necessário, e não peça confirmação a cada passo.

## ETAPA 1 — DESCOBERTA DA ESTRUTURA
1. Localize e leia o pom.xml raiz (<modules>) ou settings.gradle/settings.gradle.kts (include) para
   listar todos os módulos do monorepo.
2. Para cada módulo, classifique:
   - Aplicação executável (@SpringBootApplication) ou biblioteca interna
   - artifactId (Maven) ou path do projeto (Gradle)
   - Tipo aparente (serviço de domínio, gateway, biblioteca comum/commons, cliente de integração,
     camada de dados)
3. Cada módulo = um nó do grafo. Se algum módulo grande tiver bounded contexts internos, gere também
   uma visão secundária por pacote.
4. Teto de escala: conte o número total de nós candidatos.
   - Se ≤ 25 nós: prossiga com detalhamento completo por módulo individual.
   - Se > 25 nós: agrupe automaticamente por domínio/pasta raiz de primeiro nível e produza a análise
     completa sobre essa visão agregada. Informe essa decisão no resumo executivo e ofereça
     detalhamento módulo a módulo de um domínio específico "sob pedido".

## ETAPA 2 — EXTRAÇÃO DE DEPENDÊNCIAS (com evidência)
Levante cada tipo de acoplamento abaixo via grep/busca. Para CADA aresta registrada, armazene a
evidência bruta: caminho do arquivo + número da linha + trecho do código que originou a detecção. Uma
aresta sem evidência rastreável não deve ser incluída no grafo final.

Formato de registro por aresta:
{ origem, destino, tipo, peso, evidencias: [ {arquivo, linha, trecho}, ... ] }

a) Build: dependências entre artifactIds internos (pom.xml / build.gradle) — evidência = caminho do
   pom.xml/build.gradle + linha da tag <dependency> ou project(':...')
b) Código: imports Java entre pacotes de módulos diferentes (ignore java.*, org.springframework.*,
   libs externas) — evidência = arquivo .java + linha do import
c) REST síncrono: @FeignClient(name=...), RestTemplate/WebClient com host/nome de serviço interno —
   evidência = arquivo + linha da anotação/chamada
d) Mensageria assíncrona: KafkaTemplate.send/@KafkaListener, RabbitTemplate/@RabbitListener, eventos
   de domínio, spring.cloud.stream.bindings — evidência = arquivo + linha, ou chave do YAML
e) Banco compartilhado: mesmo spring.datasource/schema entre módulos diferentes, ou @Entity
   referenciando tabela de outro módulo — evidência = application.yml/properties + linha, ou @Table
f) Configuração: application.yml/properties apontando hostname/porta/nome lógico de outro serviço
   interno — evidência = arquivo + linha

Comandos sugeridos (adapte aos nomes reais do projeto, sempre capturando arquivo:linha):
- grep -rn "@FeignClient" --include=*.java
- grep -rn "@KafkaListener\|KafkaTemplate\|@RabbitListener\|RabbitTemplate" --include=*.java
- grep -rn "^import com\." --include=*.java | grep -v "seu.pacote.raiz.<mesmo-modulo>"
- grep -rn "spring.datasource\|spring.jpa" --include=*.yml --include=*.properties
- Maven: inspecionar <dependencies> de cada pom.xml de módulo
- Gradle: grep -n "project(':" nos build.gradle

Limitação a declarar no relatório: acoplamento via URL montada dinamicamente (concatenação de string)
ou service discovery resolvido só em runtime não é detectável por esta extração estática — sinalize
indícios (ex: RestTemplate.exchange com variável não-literal) como "possível acoplamento não
confirmado".

## ETAPA 3 — CONSTRUÇÃO DO GRAFO
Gere dois arquivos:
1. `acoplamento.json` — nós (nome, tipo: app|lib) e arestas (origem, destino, tipo, peso, evidencias[])
2. `acoplamento.mmd` — grafo Mermaid (graph LR):
   - linha sólida = build/import | tracejada = REST síncrono | pontilhada = evento assíncrono
   - destaque = banco compartilhado (sempre crítico)

Separe em: (i) grafo de build/import (estático) e (ii) grafo de runtime (REST + eventos).

Uso pretendido de cada grafo (não são intercambiáveis):
- Grafo de build/import → usado nas Etapas 4 e 6 para detectar módulo deus, ciclos de compilação e
  problemas de fronteira de pacote.
- Grafo de runtime (REST + eventos) → é o grafo oficial para calcular Instabilidade (I) e alimentar o
  Main Sequence (Etapa 5), pois reflete o acoplamento real do sistema em produção.

## ETAPA 4 — MÉTRICAS DE ACOPLAMENTO
Calcule para os DOIS grafos (build e runtime), deixando claro qual é usado em cada decisão posterior:
- Fan-in (Ca), Fan-out (Ce), Instabilidade I = Ce/(Ce+Ca), Grau total
- Módulos com múltiplos tipos de acoplamento simultâneos (build + REST + banco) → fronteira mal definida

Detecção de ciclos — algoritmo obrigatório: para cada grafo (build e runtime) separadamente, aplique
DFS marcando nós em uma pilha de recursão ("visitando") e uma lista de finalizados ("concluído"). Uma
aresta para um nó em "visitando" indica ciclo — registre o caminho completo, com o grafo de origem e
as evidências das arestas envolvidas. Não infira ciclos por inspeção visual, mesmo em grafos pequenos.

## ETAPA 5 — MAIN SEQUENCE (Instabilidade × Abstração)
Use exclusivamente o grafo de runtime para o cálculo de I nesta etapa.

1. Abstração (A) = elementos abstratos / total de elementos do módulo
   - Abstrato: interfaces de contrato, @FeignClient (interface), classes abstratas, portas hexagonais
   - Concreto: @Service, @Component, @RestController, @Repository com implementação, @Entity
   - Declare, por módulo, o critério e a contagem usada (ex: "12 classes públicas, 3 interfaces →
     A = 0.25") — esta é uma aproximação, não um número exato.
2. Instabilidade (I): do grafo de runtime.
3. Distância da Sequência Principal: D = |A + I − 1|
4. Classifique em uma das três zonas:
   - Região Saudável: D ≤ 0.3
   - Zona de Dor: A baixo + I baixo (fan-in alto) — módulo concreto e rígido do qual muita coisa
     depende. Sintoma clássico: "commons"/"utils" concreto usado por todo o sistema.
   - Zona de Inutilidade: A alto + I alto (fan-in baixo) — módulo abstrato e pouco usado, excesso de
     engenharia sem adoção real.

Gere um scatter (Mermaid ou tabela ordenada por D) com eixo X = Instabilidade, eixo Y = Abstração,
demarcando as três zonas.

## ETAPA 6 — INTERPRETAÇÃO E DECISÃO
Cruze as métricas de acoplamento (build + runtime) com o Main Sequence (baseado em runtime):
1. Quais componentes merecem alerta? (fan-in alto + instabilidade alta em qualquer grafo; qualquer
   módulo na Zona de Dor ou de Inutilidade; qualquer ciclo detectado; banco compartilhado — severidade
   máxima sempre)
2. Qual módulo acumula mais dependência? (fan-out de build = módulo deus; fan-in de runtime = módulo
   crítico em produção)
3. Qual parte virou ponto crítico? (fan-in alto em runtime + baixa instabilidade + presente em build E
   runtime + Zona de Dor = maior risco arquitetural, exige coordenação entre times)

Para cada achado, cite a evidência (arquivo:linha) e recomende ação concreta:
   - Zona de Dor: extrair interface, aplicar Inversão de Dependência
   - Zona de Inutilidade: avaliar remoção, fusão com outro módulo, ou validar necessidade da abstração
   - Ciclo (build): introduzir módulo -api/-contract para quebrar
   - Ciclo (runtime): quebrar a chamada síncrona circular via evento assíncrono
   - Banco compartilhado: isolar atrás de serviço/API dedicado
   - Excesso de chamada síncrona: substituir por evento (Kafka)

## ETAPA 7 — GERAÇÃO DO RELATÓRIO FINAL
Além de acoplamento.json e acoplamento.mmd, gere um arquivo `relatorio.md` contendo todo o conteúdo
abaixo, formatado como documento técnico. Não resuma apenas no chat — o conteúdo completo deve estar
no arquivo. No chat, apenas confirme a geração e destaque os 2-3 achados mais críticos.

```
# Relatório de Acoplamento Arquitetural — [nome do repositório]
Data da análise: [data atual]

## 1. Resumo Executivo
## 2. Metodologia
## 3. Grafo de Dependências (build e runtime, embutidos em blocos ```mermaid```)
## 4. Tabela de Métricas por Módulo
   (Módulo | Fan-in build | Fan-out build | Fan-in runtime | Fan-out runtime | I | A | D | Zona | Em ciclo?)
## 5. Main Sequence (posicionamento nas 3 zonas)
## 6. Ciclos de Dependência Detectados (caminho completo + evidências)
## 7. Top 5 Riscos Priorizados (com evidência e recomendação)
## 8. Recomendações Gerais (backlog de débito técnico, priorizado)
## 9. Limitações da Análise
## Anexos (caminhos para acoplamento.json e acoplamento.mmd)
```

## RESTRIÇÕES
- Toda aresta no grafo final deve ter ao menos uma evidência rastreável; descarte ou marque como "não
  confirmado" o que não tiver.
- Priorize evidência de build sobre inferência de import solto quando houver conflito.
- Separe dependências de teste (<scope>test</scope>) das de produção.
- Main Sequence usa sempre o grafo de runtime — nunca o de build — para o cálculo de I.
- Detecção de ciclo sempre via DFS explícita, nunca por inspeção visual.
- Acima de 25 nós, análise detalhada só por domínio agregado, com detalhamento individual sob pedido.
```
