---
name: spec-driven
description: Enriquecer prompts e criar PRDs, especificações técnicas e tarefas de desenvolvimento, em etapas isoladas ou em um fluxo integrado. Usar para transformar ideias e especificações de negócio em planejamento documentado, com PRD independente de tecnologia e decisões técnicas para Android Kotlin e backend Kotlin Spring Boot.
---

# Planejamento spec-driven

Orquestre as quatro habilidades desta pasta para transformar uma solicitação em documentos consistentes. Esta skill produz planejamento; não implementa a funcionalidade descrita nos documentos.

## Seleção da etapa

Identifique o resultado solicitado e os artefatos já existentes. Execute apenas as etapas pedidas; não inicie o fluxo completo por inferência quando o usuário solicitar somente um documento.

| Solicitação | Referência a ler integralmente | Entrada | Entrega |
| --- | --- | --- | --- |
| Enriquecer um prompt | [create_prompt.md](create_prompt.md) | Texto ou arquivo com a ideia original | Prompt estruturado nas seis seções do modelo |
| Criar um PRD | [create_prd.md](create_prd.md) | Especificação de negócio e esclarecimentos | `prd.md` |
| Criar uma Tech Spec | [create_techspec.md](create_techspec.md) | PRD completo e contexto do projeto | `techspec.md` |
| Criar tarefas | [create_tasks.md](create_tasks.md) | PRD e Tech Spec completos | `tasks.md` e `[num]_task.md` |
| Executar o fluxo completo | Ler cada referência antes da etapa correspondente | Ideia ou especificação inicial | Prompt enriquecido → PRD → Tech Spec → tarefas |

Resolva os links acima em relação à pasta desta skill, inclusive quando ela estiver instalada por link simbólico. As entradas e saídas do usuário são relativas à raiz do projeto de destino, não à pasta da skill.

Leia somente as referências das etapas que serão executadas, sempre completas antes de agir. Preserve os modelos e critérios de qualidade de cada referência. As instruções de integração abaixo resolvem o encadeamento entre elas.

## Regras de integração

- A proibição de gerar outros documentos em `create_prompt.md` vale para a etapa de enriquecimento. No fluxo completo, conclua essa etapa e então prossiga para a próxima habilidade autorizada.
- A regra de aprovação prévia de arquivos em `create_tasks.md` vale para os arquivos de tarefas, não para o prompt, o PRD e a Tech Spec produzidos nas etapas anteriores.
- A criação dos documentos não autoriza implementar código. Ao concluir as tarefas, encerre com os caminhos gerados; só inicie implementação mediante solicitação explícita.
- Aproveite respostas, decisões e aprovações já fornecidas na conversa. Não repita perguntas respondidas nem exija uma nova aprovação para a mesma lista de tarefas. Uma solicitação genérica de gerar tarefas não equivale à aprovação de uma lista ainda não apresentada.
- Execute etapas posteriores quando seus pré-requisitos estiverem satisfeitos e o usuário tiver solicitado esse encadeamento. Não acrescente aprovações entre etapas além das exigidas pelas referências ou de escolhas que realmente dependam do usuário.
- Preserve os arquivos de origem. Ao atualizar documentos existentes, leia-os primeiro e mantenha decisões e alterações do usuário. Se uma mudança anterior invalidar uma entrega posterior, indique a divergência e ajuste apenas os artefatos incluídos na solicitação.

## Organização dos artefatos

Use um único slug em kebab-case para a funcionalidade. Reutilize o slug e o diretório existentes; os nomes `nome-da-feature` e `nome-da-funcionalidade` das referências representam o mesmo identificador.

Sem caminhos específicos do usuário, use:

```text
<raiz-do-projeto>/tasks/prd-<slug>/
├── prompt.md
├── prd.md
├── techspec.md
├── tasks.md
├── 1_task.md
└── 2_task.md
```

`prompt.md` é o prompt enriquecido, criado por padrão somente no fluxo completo. No enriquecimento isolado, respeite a referência: responda com o prompt ou salve no destino informado. Crie apenas os arquivos correspondentes às etapas solicitadas. Numere as tarefas principais como `1.0`, `2.0` e as subtarefas como `1.1`, `1.2`; o arquivo da tarefa `1.0` é `1_task.md`.

Quando o usuário fornecer entradas ou destinos diferentes, respeite esses caminhos em todo o encadeamento, inclusive nas referências entre documentos. Não procure um segundo PRD no caminho padrão se o usuário já indicou o arquivo correto.

## Condução do fluxo

### 1. Enriquecer

Siga [create_prompt.md](create_prompt.md). Preserve objetivo, restrições e funcionalidades opcionais. Mantenha os rótulos de sugestão, proposta e ponto a validar em todas as entregas posteriores até que o usuário confirme uma decisão.

No fluxo completo, salve `prompt.md` e use seus requisitos de negócio, objetivo e critérios de aceite como entrada para o PRD. Separe suas propostas de arquitetura e contratos para a etapa técnica; não as converta em regras de negócio.

### 2. Definir o negócio

Siga [create_prd.md](create_prd.md). Faça os esclarecimentos de negócio necessários e aguarde as respostas que determinam o escopo antes de redigir. Considere esclarecimentos já concluídos na conversa.

O PRD deve conter resultados, requisitos funcionais numerados, regras e restrições de negócio, experiência do usuário e exclusões. Mantenha tecnologias, frameworks, endpoints, contratos de API e arquitetura fora do PRD, mesmo quando constarem no prompt enriquecido.

Salve `prd.md` com o modelo da referência e até 2.000 palavras.

### 3. Especificar a solução

Siga [create_techspec.md](create_techspec.md). Confirme e leia o PRD completo; inspecione o projeto e suas convenções antes dos esclarecimentos técnicos. Se não houver código, registre que se trata de uma solução nova e diferencie componentes propostos de componentes existentes.

Use Android nativo com Kotlin, Jetpack Compose, MVVM e Clean Architecture no aplicativo e Kotlin com Spring Boot no backend. Aproveite bibliotecas e versões existentes. Uma funcionalidade restrita a uma das partes não exige criar a outra. Se a entrada conflitar com essa stack, explicite a incompatibilidade e use a decisão expressa do usuário; não migre sistemas por conta própria.

Trate propostas técnicas do prompt como propostas até validá-las com o projeto e os esclarecimentos. Resolva lacunas de contratos, fluxo de dados, dependências e testes antes de salvar `techspec.md`, seguindo o modelo e o limite aproximado de 2.000 palavras.

### 4. Planejar as entregas

Siga [create_tasks.md](create_tasks.md). Confirme e leia o PRD e a Tech Spec completos. Se faltar algum deles, informe qual é o pré-requisito e só o crie quando isso fizer parte da solicitação.

Apresente uma lista concreta de tarefas de alto nível para aprovação antes de gerar `tasks.md` ou os arquivos individuais. Se ainda não houver aprovação dessa lista, encerre a etapa aguardando a resposta do usuário. Na retomada, recupere a lista aprovada e continue sem reiniciar o fluxo.

Após a aprovação, gere os arquivos com entregáveis, dependências, critérios de sucesso e testes unitários e de integração por tarefa; inclua E2E quando aplicável. Evite mais de dez tarefas principais. Referencie as seções da Tech Spec em vez de duplicar sua implementação.

## Ferramentas e convenções disponíveis

As referências nasceram em outro ambiente. Aplique estas adaptações sem pressupor integrações instaladas:

- Para esclarecimentos, use uma ferramenta de perguntas disponível e adequada ao ambiente ou uma pergunta direta em texto. Respostas necessárias continuam pendentes até o usuário fornecê-las.
- Quando a Tech Spec solicitar Context7, use-o se estiver disponível; caso contrário, consulte documentação oficial com os mecanismos de pesquisa disponíveis. Não dependa de uma skill chamada Context7 nem invente resultados de ferramentas.
- Com pesquisa disponível, preserve as pelo menos três buscas prévias da referência técnica e consulte fontes pertinentes ao projeto. Sem acesso à pesquisa, informe a limitação e registre o que precisa de verificação; não apresente informações externas não verificadas como conclusões.
- Interprete `@.claude/rules` e `@.claude/skills` como locais a consultar somente se existirem no projeto. Considere também os arquivos `AGENTS.md` aplicáveis, as convenções em `.agents` e `.codex` e as skills efetivamente disponíveis no ambiente. Não crie essas pastas nem declare conformidade com recursos ausentes.
- Ao listar rules ou skills em documentos e tarefas, registre somente as pertinentes e efetivamente verificadas. Mantenha os títulos dos modelos mesmo quando nenhuma regra ou skill adicional se aplicar.

## Retomada e entrega

Retome pela etapa solicitada ou pelo primeiro artefato pendente dentro do fluxo autorizado. Reutilize os documentos válidos; não os regenere apenas para percorrer as quatro habilidades.

Confira a coerência entre escopo, regras de negócio, decisões técnicas, tarefas e testes. No encerramento, informe brevemente os caminhos criados ou atualizados e a etapa atingida. Se houver uma pergunta ou aprovação pendente, indique exatamente o que falta para continuar.

