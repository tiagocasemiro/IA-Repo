---
name: prompt-guide
description: Aplicar padrões avançados de prompting para engenharia assistida por IA em refactors, debugging, testes, design complexo, mudanças multi-arquivo, protocolos e migrações reversíveis. Use quando a tarefa exigir controle de escopo, validação explícita ou redução de risco em um código existente.
---

<!-- Fonte: "Advanced Prompting Guide for AI-Assisted Engineering"; autor institucional: DX; copyright 2026. Conteúdo sintetizado e adaptado do PDF: https://getdx.com/uploads/advanced-prompting-guide-to-ai-assisted-engineering.pdf -->

# Prompt Guide para engenharia assistida por IA

Use esta skill para transformar pedidos vagos de engenharia em interações verificáveis, delimitadas e adequadas ao risco. Ela é agnóstica a ferramenta, linguagem, arquitetura e stack: pode ser aplicada a agentes, assistentes de código e fluxos orientados por especificação.

Leia [references/advanced-techniques.md](references/advanced-techniques.md) antes de aplicar um padrão. A referência contém os templates, os critérios e os detalhes de cada técnica. Quando o usuário pedir código, uma saída-modelo ou um prompt pronto para copiar, leia também [references/code-and-output-examples.md](references/code-and-output-examples.md).

## Roteamento rápido

Escolha o menor padrão que controla o risco principal:

| Situação | Padrão |
| --- | --- |
| Alteração pequena em arquivos grandes ou revisão precisa | Diff-only Refactor |
| Requisitos encadeados, conflitantes ou dependentes | Constraint Graph |
| Lógica crítica ou validação de refatoração | Dual-Implementation Self-Check |
| Bug já observado em teste, log ou relatório | Failure-First |
| Validador, parser ou entrada não confiável | Adversarial Data Generator |
| Qualidade subjetiva ou trade-offs difíceis | Rubric-Then-Solution |
| Mudança coordenada em vários arquivos/pacotes | Repository-Aware Plan-and-Apply |
| Cliente/servidor, eventos, streams ou protocolo | Protocol Simulation |
| Schema, dados ou infraestrutura que precisam de rollback | Reversible Migration |

Padrões podem ser combinados quando os riscos forem diferentes. Por exemplo: planeje uma mudança multi-arquivo com Repository-Aware Plan-and-Apply, exija Diff-only Refactor na aplicação e use Failure-First para corrigir os testes que falharem.

## Contrato operacional

Ao montar ou executar um prompt:

1. Declare o objetivo e o limite da mudança. Preserve APIs públicas, comportamento não relacionado e dependências existentes salvo instrução explícita em contrário.
2. Transforme requisitos em restrições observáveis. Distinga regras obrigatórias, preferências, dependências, conflitos e condições de retorno/backtracking.
3. Dê ao modelo apenas o contexto necessário: árvore do repositório, arquivos relevantes, falha reproduzível, tipos, protocolo ou estado atual.
4. Escolha um formato de saída aplicável e auditável: plano sem código, diff unificado, transcript de protocolo, rubric, testes ou migrações forward/backward.
5. Separe descoberta, planejamento, implementação e verificação quando a tarefa tiver mais de uma etapa. Não deixe uma etapa posterior alterar silenciosamente o escopo.
6. Defina o critério de conclusão antes da geração: testes passando, equivalência dentro de tolerância, rubric pontuada, diff limitado aos arquivos autorizados ou rollback verificado.
7. Prefira mudanças mínimas e reversíveis em sistemas grandes ou de alto risco. Não aceite “melhorias” não solicitadas apenas porque parecem úteis.

## Como responder quando a skill for usada

- Identifique o risco dominante e diga qual padrão está sendo aplicado.
- Adapte o template da referência ao idioma, ao stack e ao fluxo do usuário; não preserve exemplos de TypeScript, Node, PostgreSQL ou gRPC se eles não forem pertinentes.
- Se ainda faltarem artefatos para uma aplicação segura, peça somente os arquivos, logs, árvore ou contratos necessários.
- Se a tarefa pedir implementação, aplique o padrão até o critério de verificação; se pedir apenas um prompt, entregue o prompt pronto para copiar.
- Nunca trate a saída do modelo como prova de correção: inclua testes, comparação, simulação, revisão do diff ou validação de rollback conforme o padrão escolhido.

## Formato de entrega recomendado

Quando houver espaço para uma resposta completa, entregue nesta ordem: padrão escolhido e motivo; prompt final; contexto que o usuário precisa fornecer; exemplo de saída esperada; verificações e condição de parada. Se a tarefa for de implementação, aplique o prompt e reporte os artefatos realmente produzidos, não apenas o que seria ideal produzir.

## Adoção em equipe

Use uma progressão em três fases: medir o uso e o impacto atuais; distribuir estes padrões como complemento a práticas básicas de engenharia assistida por IA; iterar os templates com base nos resultados, centralizando os que funcionarem e adicionando guardrails onde os dados mostrarem risco.
