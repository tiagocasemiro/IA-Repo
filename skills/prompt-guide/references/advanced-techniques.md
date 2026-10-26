# Advanced Prompting Guide para AI-Assisted Engineering

Fonte primária: [Advanced Prompting Guide for AI-Assisted Engineering](https://getdx.com/uploads/advanced-prompting-guide-to-ai-assisted-engineering.pdf), da DX (edição/copyright 2026).

Para código completo, transcripts e saídas-modelo, consulte também [code-and-output-examples.md](code-and-output-examples.md). Os exemplos desse arquivo são implementações adaptadas e didáticas dos cenários do PDF; não são transcrições integrais do documento.

Este material sintetiza a introdução, os nove padrões, seus casos de uso, benefícios, formatos de saída e exemplos operacionais do PDF. Os padrões são tool- e architecture-agnostic. Use-os principalmente quando confiança, controle de escopo e reversibilidade forem mais importantes que velocidade de geração.

## Princípio geral

Pedidos como “reescreva o arquivo”, “corrija os bugs” ou “implemente com qualidade” deixam o modelo decidir escopo, qualidade e casos de borda. Os padrões abaixo tornam essas decisões explícitas e produzem artefatos que podem ser inspecionados: diffs, grafos de restrições, duas implementações, dados adversariais, rubrics, planos, transcripts e migrações de ida e volta.

O objetivo não é aplicar todos os padrões em toda tarefa. Comece com práticas básicas de engenharia assistida por IA e adicione estes padrões em sistemas críticos, bibliotecas compartilhadas, migrações, serviços interconectados e outras mudanças de alto risco. A motivação central do guia é confiança: uma equipe pode aceitar IA para boilerplate e pequenos refactors antes de confiar nela em serviços críticos, bibliotecas compartilhadas ou migrações de dados. Os padrões reduzem esse risco por meio de escopo controlado, efeitos colaterais menores, diffs, testes e workflows reversíveis.

## Mapa de casos de uso e benefícios

| Padrão | Casos de uso destacados no PDF | Benefício principal |
| --- | --- | --- |
| Diff-only | codebases grandes, refactors direcionados, code review | preserva contexto, reduz omissões e economiza tokens |
| Constraint Graph | design de sistemas, APIs, scheduling/planning | revela conflitos, prioridades e dependências ocultas |
| Dual-Implementation | lógica financeira, refactor contra legado, edge cases | encontra erros por divergência entre abordagens |
| Failure-First | bug fixing, TDD, prevenção de over-engineering | dá critério objetivo e evita scope creep |
| Adversarial Data Generator | segurança, validação, stress testing | expõe casos concretos que quebram soluções ingênuas |
| Rubric-Then-Solution | padrões de qualidade, arquitetura, code review | converte “bom” em critérios mensuráveis e iteráveis |
| Plan-and-Apply | refactor multi-arquivo, features, dependências | coordena mudanças, documenta decisões e permite execução parcial |
| Protocol Simulation | sistemas distribuídos, protocolos, integration testing | torna mensagens, ordem, falhas e estados concretos antes do código |
| Reversible Migration | schema, IaC, transformação de dados | preserva dados e permite rollback parcial testável |

## 1. Diff-only Refactor Prompting

### Quando usar

- Arquivos grandes demais para regeneração segura ou confortável no contexto.
- Refactors pontuais em vários arquivos.
- Code review que precisa mostrar exatamente o que mudou.

### Ideia e benefícios

Peça somente um unified diff, nunca a regeneração integral do arquivo. O diff preserva contexto, comentários e formatação, reduz omissões acidentais, economiza tokens e pode ser aplicado por ferramentas padrão.

### Template

```text
Vou fornecer um arquivo [LINGUAGEM]. Faça [MUDANÇA]. Responda SOMENTE com um unified diff que:

- preserve todas as APIs públicas;
- melhore [legibilidade/testabilidade, se necessário];
- não adicione dependências;
- não altere comportamento fora do objetivo.

Use exatamente este formato:

--- a/[ARQUIVO]
+++ b/[ARQUIVO]
@@ ...
-linha antiga
+linha nova

Não inclua explicações, arquivo completo ou alterações fora do diff.

Arquivo:
[CONTEÚDO]
```

### Verificação

Confira que o patch é sintaticamente aplicável, toca apenas o alvo, preserva APIs e passa os testes relevantes. “Diff preciso” é requisito de saída, não garantia de correção.

## 2. Constraint-Graph Prompting

### Quando usar

- Design de sistemas com requisitos que competem entre si.
- APIs com validações dependentes.
- Planejamento com estados, ordem temporal, recursos ou regras condicionais.

### Como modelar

Represente explicitamente:

- **hard constraints**: regras não negociáveis;
- **soft constraints**: preferências que podem ser relaxadas;
- **raízes**: condições independentes;
- **dependências**: regra que só pode ser avaliada depois de outra;
- **conflitos**: qual regra vence e qual comportamento ocorre;
- **backtracking**: estado para o qual o fluxo retorna quando uma condição falha.

### Template

```text
Projete [SISTEMA] usando um grafo de dependências.

HARD CONSTRAINTS (obrigatórias):
- C1: [regra] [ROOT ou depende de ...]
- C2: [regra]

SOFT CONSTRAINTS (preferíveis):
- C3: [regra] [depende de ...]

CONFLICT RESOLUTION:
- Se C1 falhar -> [erro/comportamento].
- Se C3 entrar em conflito com C2 -> [qual regra vence].

DEPENDENCIES:
- [C...] só pode ser avaliada depois de [C...].

BACKTRACKING RULES:
- Se [condição] falhar -> retornar a [estado] e [ação].

Mostre o grafo, explicite conflitos descobertos e só então proponha a implementação.
```

### Forma útil para fluxos por estado

```text
STATE: Cart
|- C1: pelo menos um item [REQUIRED]
|- C2: todos os itens em estoque [REQUIRED]
v
STATE: Shipping
|- C3: endereço válido [REQUIRED, depende de C1-C2]
|- C4: frete grátis se total > limite [OPTIONAL, modifica seleção]
v
STATE: Payment
|- C5: método de pagamento [REQUIRED]
|- C6: valor = total + frete - descontos [REQUIRED]
```

### Verificação

Procure combinações contraditórias, caminhos sem tratamento e dependências avaliadas cedo demais. O grafo deve tornar visíveis casos que um pedido linear esconderia.

## 3. Dual-Implementation Self-Check Prompting

### Quando usar

- Cálculos financeiros, transformações e lógica de negócio crítica.
- Validação de uma otimização contra uma implementação legada.
- Algoritmos com muitos casos de borda.

### Ideia

Peça duas implementações independentes do mesmo contrato, baseadas em abordagens realmente diferentes, e um harness que compare os resultados. A independência aumenta a chance de que bugs diferentes produzam uma discrepância detectável. As duas implementações também formam uma especificação viva uma da outra.

### Template

```text
Implemente [FUNÇÃO] com duas abordagens independentes:

- Implementação A: [abordagem 1].
- Implementação B: [abordagem 2, matematicamente/algoritmicamente diferente].

Inclua um test harness que:
1. execute pelo menos 50 entradas diversas;
2. compare os resultados com tolerância de [TOLERÂNCIA, se numérico];
3. reporte cada discrepância com os inputs e ambos os outputs;
4. cubra zero, valores negativos, valores grandes e demais bordas relevantes.

Defina tipos, contrato e política para entradas inválidas. Não considere a execução sem discrepâncias como prova absoluta: revise os casos e a independência das abordagens.
```

O exemplo do guia usa juros compostos: uma implementação iterativa e outra logarítmica, tolerância de `0.01`, 50+ inputs e bordas como principal/taxa zero, valores negativos e números grandes.

## 4. Failure-First Prompting

### Quando usar

- Teste unitário ou de integração falhando.
- Logs de erro, stack trace ou bug report reproduzível.
- TDD incremental em que o sucesso é objetivo e específico.

### Ideia e limites

Forneça primeiro a falha concreta e o arquivo relevante. O modelo deve explicar brevemente a causa e fazer apenas o patch mínimo para satisfazer o critério. Isso reduz escopo, alucinação, over-engineering e adição de features não solicitadas.

### Template

```text
Vou fornecer a saída de um teste que falhou e o arquivo relevante.

Sua tarefa:
- explique em um parágrafo por que o teste falha;
- proponha um patch mínimo em unified diff para fazê-lo passar;
- não altere assinaturas públicas;
- não adicione features, logging, padrões ou otimizações não exigidos.

Responda exatamente com:
1) uma justificativa de um parágrafo;
2) um unified diff.

Falha:
[LOG/TESTE]

Arquivo:
[ARQUIVO]
```

### Verificação

Rode o teste que falhou e a suíte relacionada. Se o patch exigir uma mudança fora do arquivo fornecido, pare e peça o contexto mínimo adicional em vez de ampliar o escopo silenciosamente.

## 5. Adversarial Data-Generator Prompting

### Quando usar

- Validadores, parsers e importadores.
- Segurança de entrada: SQL injection, XSS, path traversal e payloads malformados.
- Stress tests e algoritmos suscetíveis a pior caso.

### Ideia

Separe ataque e defesa: primeiro gere dados difíceis, maliciosos ou de fronteira; depois escreva o código que precisa sobreviver a eles. Pedir exemplos que “provavelmente quebrariam implementações ingênuas” revela pontos cegos que um pedido genérico de robustez costuma esconder.

### Template

```text
Você é um gerador adversarial de dados para uma API [FORMATO/DOMÍNIO].

Passo 1: gere 50 exemplos que provavelmente quebrariam implementações ingênuas, incluindo:
- casos Unicode e normalização;
- números muito grandes e tipos inesperados;
- objetos profundamente aninhados;
- campos ausentes de formas sorrateiras;
- [ameaças e bordas específicas do domínio].

Passo 2: usando a definição de tipo abaixo, escreva
`[NOME](input: unknown): [TIPO] | null` para aceitar somente casos válidos.

Retorne:
- os 50 exemplos adversariais;
- o validador;
- os critérios usados para aceitar ou rejeitar cada classe de entrada.

Tipo/contrato:
[DEFINIÇÃO]
```

O exemplo do guia usa importação JSON/JSONL de usuários com espaços Unicode, identificadores enormes, campos com tipo errado, email ausente e objetos aninhados. Em JSONL, cada linha não vazia deve ser parseada isoladamente; linhas inválidas devem ser rejeitadas sem corromper os registros válidos, com erro contextualizado.

### Verificação

Execute os exemplos adversariais, acrescente casos descobertos durante a implementação e verifique que o validador não confia em casts, truthiness ou apenas na forma superficial do objeto.

## 6. Rubric-Then-Solution Prompting

### Quando usar

- “Qualidade” precisa ser definida para uma tarefa ou equipe.
- Há trade-offs de performance, manutenibilidade, segurança e observabilidade.
- Code review automatizado ou iteração orientada por uma meta mensurável.

### Processo em duas fases

**Fase 1 — rubric:** peça uma rubric específica com 8–12 critérios, cada um pontuado de 0 a 5. Cubra os atributos relevantes, crie `RUBRIC.md`, revise a rubric e itere até ela ser aceitável.

**Fase 2 — solução:** implemente visando maximizar a pontuação da rubric, produza testes e faça uma autoavaliação em tabela com critério, nota e justificativa. O usuário pode pedir melhorias dirigidas aos itens de menor nota.

### Template

```text
Fase 1: você é um engenheiro sênior definindo a rubric de “qualidade de [ARTEFATO] em [STACK]”.

1. Produza 8–12 critérios concretos, cada um com escala 0–5 e descrições observáveis para cada nota.
2. Cubra: [error handling, observabilidade, testabilidade, retries/timeouts, separação de responsabilidades e outros atributos do domínio].
3. Escreva a rubric em `RUBRIC.md`.
4. Avalie a própria rubric e itere se houver critérios vagos, sobrepostos ou ausentes.

Fase 2: usando `RUBRIC.md`, implemente [SOLUÇÃO].

Retorne:
1) implementação;
2) testes;
3) tabela de autoavaliação por critério, com nota 0–5 e justificativa.
```

### Dimensões de referência

Para um cliente HTTP Node.js, o exemplo do guia avalia: tratamento/modos de falha; retries, timeouts e backoff; observabilidade; testabilidade; API e separação de responsabilidades; configuração/extensibilidade/ambiente; dados, serialização e validação; segurança e resiliência; uso idiomático do runtime/bibliotecas; documentação e experiência do desenvolvedor.

Uma rubric madura diferencia, por exemplo, retry ingênuo de retry limitado a falhas transitórias e métodos idempotentes; backoff fixo de exponencial com jitter; timeout de transporte de timeouts por requisição e globais; e erro genérico de uma taxonomia tipada com contexto, redaction e contrato previsível.

### Verificação

Não aceite a autoavaliação como nota final. Rode os testes, revise os critérios contra a implementação e use a rubric para pedir melhorias específicas.

## 7. Repository-Aware Plan-and-Apply Prompting

### Quando usar

- Refactors e features multi-arquivo.
- Mudanças que cruzam API, worker, banco, lógica, UI ou pacotes compartilhados.
- Atualizações de dependência com múltiplos consumidores.

### Fase 1 — plano

Forneça a árvore do repositório e os arquivos-chave. Peça somente um plano numerado, com no máximo oito passos. Cada passo deve nomear arquivos específicos, ser pequeno o suficiente para um único PR e ter um critério de sucesso curto. Não escreva código.

### Fase 2 — aplicação

Depois de revisar o plano, peça `implemente somente o passo N`. Restrinja o agente aos arquivos listados naquele passo e exija unified diff; inclua testes no diff quando forem necessários.

### Templates

```text
Fase 1 — plano somente.

Analise esta árvore e os arquivos-chave:
[TREE + ARQUIVOS]

Objetivo: [OBJETIVO]

Proponha um plano numerado com no máximo 8 passos. Cada passo deve:
- apontar arquivos específicos;
- ser pequeno o suficiente para um único PR;
- incluir um critério de sucesso.

Não escreva código ainda.
```

```text
Fase 2 — aplicação.

Implemente somente o passo [N] do plano aprovado.
- toque apenas os arquivos listados naquele passo;
- retorne um unified diff;
- inclua testes no diff se necessário;
- se o contexto for insuficiente ou o passo exigir outro arquivo, pare e explique o bloqueio.
```

### Verificação

Revise o plano antes de aplicar, verifique cada diff contra a lista autorizada e rode o critério de sucesso de cada passo antes de prosseguir. O benefício é coordenação antecipada, decisões auditáveis e execução parcial verificável.

## 8. Protocol Simulation Prompting

### Quando usar

- Sistemas distribuídos, microserviços, filas e eventos.
- gRPC, WebSockets, TCP ou protocolos customizados.
- Integrações em que ordem, sincronização, falhas e estados importam.

### Processo

Trate o modelo como simulador de um sistema distribuído. Primeiro produza um transcript concreto, indicando emissor, receptor, mensagem, ordem e estado após cada troca. Inclua mensagens inválidas, erros, reconexão/encerramento e condições de corrida relevantes. Depois de revisar a simulação, extraia os handlers de cliente e servidor.

### Template

```text
Simule uma interação [PROTOCOLO] entre [COMPONENTE A] e [COMPONENTE B].

Contrato:
- serviço/RPC/evento: [CONTRATO]
- estados: [ESTADOS]
- mensagens: [TIPOS]

Simule, em ordem:
1. [abertura/handshake]
2. [mensagem válida e resposta]
3. [transição de estado]
4. [mensagem inválida e tratamento]
5. [falha/timeout/erro, sem perder invariantes]
6. [condição terminal e encerramento]

Mostre cada troca como `A -> B: mensagem`, indique o estado resultante e destaque ambiguidades, corridas e mensagens sem tratamento. Só depois extraia os handlers de ambos os lados.
```

O exemplo do PDF simula um stream bidirecional `PlayGame`: cliente envia `PLAYER_JOIN`, servidor responde com estado inicial, cliente envia `MOVE`, servidor atualiza e responde, uma jogada inválida produz `ERROR` sem fechar o stream, uma jogada termina o jogo com `GAME_OVER`, e `QUIT` fecha o stream. A simulação serve como prova de entendimento executável antes do código.

### Verificação

Confira que cada mensagem tem um consumidor, cada estado possui transições válidas, erros não deixam o protocolo em estado impossível e o encerramento é definido para ambos os lados. Gere testes de integração a partir do transcript quando fizer sentido.

## 9. Reversible Migration Prompting

### Quando usar

- Alterações de schema e dados.
- Infraestrutura como código que pode precisar ser desmontada.
- Migração entre formatos sem perda de dados.
- Deploys graduais em que código antigo e novo coexistem.

### Requisitos

Peça sempre a migração forward e a backward, além de passos de verificação. Force o modelo a considerar preservação de dados, idempotência, compatibilidade durante a transição e rollback parcial; evite operações destrutivas sem uma estratégia de recuperação.

### Template

```text
Estado atual:
[SCHEMA/INFRA/FORMATO]

Novo requisito:
[MUDANÇA]

Produza:
1. migração forward com [recursos adicionados, backfill, compatibilidade];
2. migração backward que reverta a mudança de forma limpa;
3. testes/verificações para executar em staging;
4. exemplo de leitura/escrita segura durante a transição em [STACK].

Preserve dados existentes, mantenha compatibilidade com código antigo quando necessário e faça as operações idempotentes. Explique quais passos são seguros para reexecução e quais exigem pré-condições.
```

No exemplo do guia, `users.full_name` é dividido em `first_name` e `last_name` no PostgreSQL: a forward adiciona colunas, faz backfill e cria view/camada de compatibilidade se necessário; a backward desfaz a alteração; e um snippet TypeScript com `pg` demonstra leitura/escrita durante a transição. A prioridade é não quebrar consumidores antigos nem perder dados.

### Verificação

Execute forward e backward em staging com uma cópia representativa, valide contagens/checksums e reexecução, teste o período de compatibilidade e confirme que o rollback não apaga informação que não possa ser reconstruída.

## Prática de adoção

O fechamento do guia recomenda três ações:

1. **Medir:** estabelecer uma linha de base de uso e impacto de ferramentas de IA com métricas multidimensionais, como as do AI Measurement Framework da DX.
2. **Distribuir:** usar este material como complemento avançado de um guia básico de engenharia assistida por IA, reservando os padrões para sistemas e equipes de maior risco.
3. **Iterar:** tratar prompts como templates vivos, adaptá-los à stack e ao fluxo da equipe, centralizar os que funcionarem e reforçar guardrails onde as métricas mostrarem necessidade.
