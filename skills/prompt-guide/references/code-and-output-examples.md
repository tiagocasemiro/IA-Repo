# Exemplos de código e saídas

Este arquivo complementa o resumo dos padrões com exemplos de prompts, código e respostas esperadas. Os snippets são intencionalmente pequenos para poderem ser adaptados; execute-os e ajuste contratos, segurança e convenções da aplicação antes de usá-los em produção.

## 0. Anatomia de um prompt avançado

Use estes blocos como checklist. Nem todos precisam aparecer literalmente, mas cada decisão relevante deve estar coberta:

```text
CONTEXTO
- stack, versão, arquitetura, arquivos/contratos relevantes
- estado atual e evidência observada

OBJETIVO
- mudança desejada e fora de escopo

INVARIANTES
- APIs, comportamento, dados, segurança e compatibilidade que não podem mudar

REGRAS
- HARD: obrigatórias
- SOFT: preferíveis
- DEPENDENCIES: ordem de avaliação
- CONFLICT RESOLUTION: regra que vence
- BACKTRACKING: para onde voltar quando falhar

ARTEFATOS
- diff, plano, testes, transcript, rubric, migrações ou código

FORMATO DE SAÍDA
- exatamente o que deve ser retornado e o que não deve ser incluído

VERIFICAÇÃO E PARADA
- testes/comparações/checagens e quando parar para pedir contexto
```

Um prompt avançado não é necessariamente longo: ele é explícito sobre o que o modelo não pode decidir sozinho.

## 1. Diff-only Refactor — código e saída

### Prompt pronto

```text
Vou fornecer `user.ts`. Corrija a tipagem de `parseUser`.

Responda SOMENTE com um unified diff:
- preserve a assinatura pública e o comportamento de aceitar um usuário válido;
- troque `any` por `unknown` e faça narrowing seguro;
- não adicione dependências;
- não altere outros arquivos nem inclua o arquivo completo.

Formato obrigatório:
--- a/user.ts
+++ b/user.ts
@@ ...

Arquivo atual:
[COLE user.ts AQUI]
```

### Entrada reduzida

```ts
export type User = { id: string; name: string };

export function parseUser(input: any): User | null {
  if (!input) return null;
  return { id: String(input.id), name: String(input.name) };
}
```

### Saída esperada

```diff
--- a/user.ts
+++ b/user.ts
@@
-export function parseUser(input: any): User | null {
-  if (!input) return null;
-  return { id: String(input.id), name: String(input.name) };
+export function parseUser(input: unknown): User | null {
+  if (!input || typeof input !== "object") return null;
+  const candidate = input as { id?: unknown; name?: unknown };
+  if (!candidate.id || !candidate.name) return null;
+  return { id: String(candidate.id), name: String(candidate.name) };
 }
```

### Como auditar

```bash
git apply --check change.patch
git diff --stat
npm test -- --runInBand
```

Rejeite uma resposta que contenha texto fora do diff, recrie um arquivo inteiro, mude uma API ou altere linhas não necessárias. O formato machine-parseable é uma vantagem do padrão, não um substituto para revisão.

## 2. Constraint Graph — API e checkout

### Prompt pronto para uma API

```text
Projete uma API de registro de usuário e mostre primeiro o grafo de restrições.

HARD:
- C1 email válido e único [ROOT]
- C2 senha com pelo menos 8 caracteres [ROOT]
- C3 username alfanumérico de 3–20 caracteres [ROOT]

SOFT:
- C4 username preferencialmente único [depende de C3]
- C5 sugerir senha forte [depende de C2]

CONFLITOS:
- C1 violada -> HTTP 409 e nenhum processamento downstream
- C4 violada -> permitir, anexando sufixo aleatório
- C2 violada -> rejeitar; não sugerir senha até a validação estrutural passar

Retorne: grafo, tabela de decisões, casos de borda e só então contrato/implementação.
```

### Saída-modelo

```text
C1 ──blocks──> C4
│
└──blocks──> C5
C2 ──enables──> C5
C3 ──enables──> C4

Caso: email duplicado
Resultado: 409 CONFLICT
Efeito: não validar username nem sugerir senha.

Caso: username já usado, mas C3 válido
Resultado: criar username com sufixo; registrar a transformação.
```

### Exemplo de saída de planejamento

Para um fluxo novo de checkout, uma saída útil antes do código pode ser:

```text
To-do
1. Configurar estrutura do projeto e dependências.
2. Criar validação das restrições e gerenciamento de estado.
3. Implementar Cart com C1–C3.
4. Implementar Shipping com C4–C7.
5. Implementar Payment com C8–C11.
6. Implementar backtracking de C6, C8 e C10.
7. Criar componentes de UI para cada estado.
8. Criar App principal e estilos.
```

O valor dessa lista não é a quantidade fixa de passos; é tornar a ordem, as dependências e os pontos de verificação explícitos antes da geração.

### Exemplo de estado em TypeScript

```ts
import { randomUUID } from "node:crypto";

type Registration = {
  email: string;
  password: string;
  username: string;
};

type Decision =
  | { ok: true; username: string; suggestion?: string }
  | { ok: false; status: 400 | 409; reason: string };

export function validateRegistration(
  input: Registration,
  emailExists: (email: string) => boolean,
  usernameExists: (username: string) => boolean,
): Decision {
  if (emailExists(input.email)) {
    return { ok: false, status: 409, reason: "email_not_unique" };
  }
  if (input.password.length < 8) {
    return { ok: false, status: 400, reason: "password_too_short" };
  }
  if (!/^[a-z0-9]{3,20}$/i.test(input.username)) {
    return { ok: false, status: 400, reason: "invalid_username" };
  }

  const username = usernameExists(input.username)
    ? `${input.username}-${randomUUID().slice(0, 6)}`
    : input.username;
  const suggestion = input.password === input.password.toLowerCase()
    ? "Use upper/lowercase, a number and a symbol."
    : undefined;
  return { ok: true, username, suggestion };
}
```

O ponto importante não é o regex: é a ordem observável das dependências e o tratamento explícito do conflito entre unicidade obrigatória e unicidade preferencial.

### Grafo para checkout com backtracking

```text
Cart
├─ C1 pelo menos um item [REQUIRED]
├─ C2 todos os itens em estoque [REQUIRED]
└─ C3 total > 0 [REQUIRED]
   ↓
Shipping
├─ C4 endereço válido [REQUIRED, depende de C1–C3]
├─ C5 método de envio selecionado [REQUIRED]
├─ C6 restrições do endereço compatíveis [REQUIRED, depende de C2]
└─ C7 frete grátis se total > $50 [OPTIONAL, altera C5]
   ↓
Payment
├─ C8 método de pagamento [REQUIRED]
├─ C9 endereço de cobrança se diferente [CONDITIONAL on C4]
├─ C10 valor = cart + shipping - discounts [REQUIRED]
└─ C11 CVV se cartão [CONDITIONAL on C8]

Backtracking:
- C6 falhou -> Cart, remover itens incompatíveis
- C8 falhou -> permanecer em Payment, permitir retry
- C10 falhou -> Cart, recalcular após mudança de preço
```

## 3. Dual-Implementation Self-Check — juros compostos

O PDF usa duas abordagens para juros compostos: cálculo iterativo e cálculo logarítmico, comparados em 50+ entradas, com tolerância `0.01`. Este exemplo explicita uma política para valores inválidos antes de comparar.

### Implementações independentes

```ts
type InterestInput = {
  principal: number;
  rate: number;      // decimal anual: 0.05 = 5%
  years: number;
  compounds: number; // vezes por ano
};

function assertValid(x: InterestInput): void {
  if (![x.principal, x.rate, x.years, x.compounds].every(Number.isFinite)) {
    throw new RangeError("all inputs must be finite");
  }
  if (x.principal < 0 || x.rate < 0 || x.years < 0 || x.compounds <= 0) {
    throw new RangeError("negative values and non-positive compounds are invalid");
  }
}

export function interestIterative(x: InterestInput): number {
  assertValid(x);
  let amount = x.principal;
  const periods = Math.round(x.years * x.compounds);
  const periodRate = x.rate / x.compounds;
  for (let i = 0; i < periods; i += 1) amount *= 1 + periodRate;
  return amount;
}

export function interestLogarithmic(x: InterestInput): number {
  assertValid(x);
  if (x.principal === 0) return 0;
  const periods = x.years * x.compounds;
  return x.principal * Math.exp(periods * Math.log1p(x.rate / x.compounds));
}
```

### Harness

```ts
function inputs(): InterestInput[] {
  return Array.from({ length: 50 }, (_, i) => ({
    principal: i === 0 ? 0 : 100 + i * 137.25,
    rate: (i % 11) / 100,
    years: i % 9,
    compounds: [1, 2, 4, 12, 365][i % 5],
  }));
}

const tolerance = 0.01;
const discrepancies: Array<{ input: InterestInput; a: number; b: number }> = [];

for (const input of inputs()) {
  const a = interestIterative(input);
  const b = interestLogarithmic(input);
  if (Math.abs(a - b) > tolerance) discrepancies.push({ input, a, b });
}

for (const invalid of [
  { principal: -1, rate: 0.1, years: 1, compounds: 12 },
  { principal: 100, rate: 0.1, years: 1, compounds: 0 },
]) {
  expect(() => interestIterative(invalid)).toThrow(RangeError);
  expect(() => interestLogarithmic(invalid)).toThrow(RangeError);
}

if (discrepancies.length) {
  console.error(JSON.stringify(discrepancies, null, 2));
  process.exitCode = 1;
} else {
  console.log(`OK: ${inputs().length} valid cases within ±${tolerance}`);
}
```

### Saída esperada

```text
OK: 50 valid cases within ±0.01
```

### Saída quando há bug

```text
Discrepancies: 1
input: {"principal":100,"rate":0.05,"years":10,"compounds":12}
iterative: 164.700949712
logarithmic: 164.700949712
absolute difference: 0.0142 (tolerance 0.01)
```

O harness deve reportar input e ambos os resultados. “As duas funções rodaram” não é suficiente; a diferença precisa ser analisável.

## 4. Failure-First — bug de formatação

### Entrada

```text
Falha do Jest:
Expected: "10.00"
Received: "10"

Arquivo:
export function formatPrice(value: number): string {
  return `${value}`;
}
```

### Prompt

```text
Explique em um parágrafo por que este teste falha e proponha o menor unified diff para fazê-lo passar.
Não altere a assinatura pública, não adicione features e não mude outros arquivos.
Responda com a justificativa e depois somente o diff.
```

### Saída-modelo

```text
O teste falha porque a interpolação converte um inteiro em `"10"`, mas o contrato exige sempre duas casas decimais. A correção mínima é formatar o número com `toFixed(2)`.

--- a/format.ts
+++ b/format.ts
@@
 export function formatPrice(value: number): string {
-  return `${value}`;
+  return value.toFixed(2);
 }
```

### Verificação

```ts
expect(formatPrice(10)).toBe("10.00");
expect(formatPrice(10.5)).toBe("10.50");
```

Se a falha for insuficiente para saber se `NaN`, negativos ou arredondamento fazem parte do contrato, o agente deve sinalizar a ambiguidade; não deve inventar uma política de produto.

## 5. Adversarial Data Generator — JSONL

### Prompt pronto

```text
Você é um gerador adversarial para uma API de importação de usuários em JSONL.

Gere 50 objetos que provavelmente quebrem validadores ingênuos. Inclua Unicode invisível/NBSP e normalização, números maiores que Number.MAX_SAFE_INTEGER, arrays e objetos onde strings são esperadas, nesting profundo, campos ausentes/null/duplicados, email inválido, whitespace e linhas vazias.

Depois escreva `parseJsonLines(input: string): UserRecord[]`.
Requisitos:
- parsear cada linha não vazia independentemente;
- aceitar somente objetos válidos;
- ignorar uma linha inválida sem perder as demais;
- informar o número da linha e a causa;
- não usar casts como validação.

Retorne primeiro os dados adversariais, depois o código e uma tabela de aceitação/rejeição.
```

### Amostra de dados adversariais

```json
{"id":"0001","name":"Jose\u00a0","email":"user@example.com"}
{"id":12345678901234567890,"name":"BigInt-ish","email":"big@example.com"}
{"id":"nested","name":{"first":"Jane","last":"Doe"},"email":"jane@example.com"}
{"id":"missing-email","name":"No Email"}
{"id":"null-name","name":null,"email":"null@example.com"}
{"id":"unicode-zero-width","name":"Jo\u200Bse","email":"x@example.com"}
```

### Validador TypeScript

```ts
type UserRecord = { id: string; name: string; email: string };

function parseUserRecord(input: unknown): UserRecord {
  if (!input || typeof input !== "object" || Array.isArray(input)) {
    throw new Error("record must be an object");
  }
  const value = input as Record<string, unknown>;
  if (typeof value.id !== "string" || value.id.length === 0) {
    throw new Error("id must be a non-empty string");
  }
  if (typeof value.name !== "string" || value.name.trim() === "") {
    throw new Error("name must be a non-empty string");
  }
  if (typeof value.email !== "string" || !/^[^@\s]+@[^@\s]+\.[^@\s]+$/.test(value.email)) {
    throw new Error("email is invalid");
  }
  return { id: value.id, name: value.name, email: value.email };
}

export function parseJsonLines(input: string): UserRecord[] {
  const records: UserRecord[] = [];
  for (const [index, line] of input.split(/\r?\n/).entries()) {
    const trimmed = line.trim();
    if (!trimmed) continue;
    try {
      records.push(parseUserRecord(JSON.parse(trimmed)));
    } catch (error) {
      const reason = error instanceof Error ? error.message : "invalid record";
      console.error(`Skipping invalid JSON on line ${index + 1}: ${reason}`);
    }
  }
  return records;
}
```

### Saída esperada

```text
Skipping invalid JSON on line 2: id must be a non-empty string
Skipping invalid JSON on line 3: name must be a non-empty string
Skipping invalid JSON on line 4: email is invalid
accepted: 3
rejected: 3
```

Em casos de segurança, acrescente payloads de SQL injection, XSS e path traversal ao conjunto de testes, mas trate-os como dados de teste: não os execute como comandos.

## 6. Rubric-Then-Solution — HTTP client

### Prompt completo em duas fases

```text
Fase 1. Crie `RUBRIC.md` para um cliente HTTP reutilizável em Node.js com `fetch`.
Use 10 critérios, cada um pontuado de 0 a 5, com descrições concretas para todas as notas. Cubra:
1. error handling e failure modes
2. retries, timeouts e backoff
3. observabilidade
4. testabilidade
5. API e separação de responsabilidades
6. configuração/extensibilidade/ambiente
7. serialização e validação de dados
8. segurança e resiliência
9. uso idiomático de Node.js/bibliotecas
10. documentação e developer experience
Revise a rubric até que os critérios não se sobreponham.

Fase 2. Usando `RUBRIC.md`, implemente o cliente e um teste Jest que injete/mocke `fetch`.
Priorize a maior pontuação possível sem adicionar dependências desnecessárias.
Retorne implementação, testes e tabela de autoavaliação critério/nota/justificativa.
```

### Recorte de rubric 0–5

| Critério | 0 | 3 | 5 |
| --- | --- | --- | --- |
| Erros | engole falhas ou deixa promises sem tratamento | classifica rede/timeout/HTTP/parsing | erros tipados, contexto, redaction, contrato documentado e invariantes consistentes |
| Retries/timeouts | pode travar indefinidamente | configuração e retry de falhas transitórias/idempotentes | políticas por operação, backoff exponencial com jitter e guardrails |
| Observabilidade | nenhuma | hooks básicos | logs/métricas/traces estruturados sem segredos |
| Testabilidade | depende de rede real | mock parcial | `fetch` injetável e testes de sucesso, falha e retry |
| Separação | tudo em uma função | camadas parcialmente separadas | transporte, política e serialização com responsabilidades claras |

O PDF também exemplifica distinguir falhas transitórias/permanentes, HTTP de rede/aplicação, timeout de conexão de timeout de resposta, retry de método idempotente de retry que pode duplicar efeitos e erro genérico de erro com URL/método/status/correlation ID.

### Exemplo de implementação

```ts
export class HttpClientError extends Error {
  constructor(
    message: string,
    readonly kind: "network" | "timeout" | "http" | "parse",
    readonly status?: number,
  ) { super(message); }
}

type FetchLike = (input: RequestInfo | URL, init?: RequestInit) => Promise<Response>;

export class HttpClient {
  constructor(
    private readonly fetchImpl: FetchLike = fetch,
    private readonly baseUrl = "",
  ) {}

  async getJson<T>(path: string, timeoutMs = 5_000): Promise<T> {
    const controller = new AbortController();
    const timer = setTimeout(() => controller.abort(), timeoutMs);
    try {
      let response: Response;
      try {
        response = await this.fetchImpl(`${this.baseUrl}${path}`, {
          method: "GET",
          headers: { accept: "application/json" },
          signal: controller.signal,
        });
      } catch (error) {
        if (controller.signal.aborted) {
          throw new HttpClientError("request timed out", "timeout");
        }
        throw new HttpClientError("network request failed", "network");
      }
      if (!response.ok) {
        throw new HttpClientError(`HTTP ${response.status}`, "http", response.status);
      }
      try {
        return (await response.json()) as T;
      } catch {
        throw new HttpClientError("response is not valid JSON", "parse");
      }
    } finally {
      clearTimeout(timer);
    }
  }
}
```

### Teste e saída

```ts
it("maps an HTTP failure to a typed error", async () => {
  const fetchMock = jest.fn<ReturnType<FetchLike>, Parameters<FetchLike>>(
    async () => new Response("no", { status: 503 }),
  );
  await expect(new HttpClient(fetchMock).getJson("/users/1"))
    .rejects.toMatchObject({ kind: "http", status: 503 });
});
```

```text
PASS http-client.test.ts
  ✓ maps an HTTP failure to a typed error (4 ms)

Self-evaluation:
1. Error Handling & Failure Modes       4  typed network/timeout/http/parse errors
2. Retries, Timeouts & Backoff          2  timeout exists; retry policy still absent
3. Observability                        1  no hooks yet
4. Testability                          4  fetch is injectable
5. API & Separation                     4  client owns transport and parsing separately enough
Overall: 15/25; next iteration: retry policy, hooks, redaction and broader tests.
```

### Saída de autoavaliação mais completa

```text
Criterion                                      Score  Evidence
1. Error Handling & Failure Modes                 5  typed hierarchy, context and redaction
2. Retries, Timeouts & Backoff                    4  configurable exponential backoff with jitter
3. Observability                                  4  request hooks and structured events
4. Testability & Automated Tests                  4  injectable fetch and failure coverage
5. API Design & Separation of Concerns            5  transport/policy/serialization separated
6. Configuration & Environment Management         4  centralized base URL and retry policy
7. Data Handling & Validation                     4  explicit JSON parsing and validation
8. Security & Resilience                          4  auth provider, safe headers and limits
9. Node.js & Library Usage                        4  async/await and AbortController
10. Documentation & Developer Experience          4  JSDoc, examples and predictable errors
```

Essa tabela reproduz a forma de saída exemplificada no PDF: critério, score de 0–5 e uma justificativa curta que permite escolher a próxima iteração.

A saída-modelo deve revelar lacunas. Se todos os critérios recebem 5 sem evidência, a autoavaliação está sendo usada como elogio, não como instrumento de iteração.

## 7. Repository-Aware Plan-and-Apply — monorepo

### Contexto de entrada

```text
Monorepo:
/services/api
/services/worker
/packages/shared
/packages/logger

Objetivo: introduzir `RequestId` desde a entrada da API até os logs do worker.
Arquivos-chave: api/src/middleware.ts, api/src/queue.ts,
worker/src/consumer.ts, packages/shared/src/messages.ts,
packages/logger/src/index.ts e testes correspondentes.
```

### Saída da fase 1

```text
1. packages/shared/src/messages.ts — adicionar RequestId ao envelope da mensagem.
   Sucesso: tipos compilam e consumidores existentes permanecem compatíveis.
2. api/src/middleware.ts — gerar/validar RequestId na entrada.
   Sucesso: toda requisição recebe um ID estável e não aceita valor inválido.
3. api/src/queue.ts — propagar o ID no publish.
   Sucesso: mensagem publicada contém o mesmo ID da requisição.
4. worker/src/consumer.ts — extrair o ID e abrir contexto de log.
   Sucesso: logs do processamento contêm RequestId.
5. packages/logger/src/index.ts — expor contexto sem vazar dados sensíveis.
   Sucesso: testes cobrem presença e limpeza do contexto.
6. adicionar testes unitários e de integração nos arquivos acima.
   Sucesso: suíte relevante passa.
```

### Prompt da fase 2

```text
Implemente somente o passo 3 do plano aprovado.
Toque apenas em `api/src/queue.ts` e no teste listado para esse passo.
Retorne unified diff. Não refatore outros trechos nem aplique passos futuros.
Se o tipo de mensagem exigir uma alteração fora da lista, pare e explique qual contexto falta.
```

### Saída-modelo da aplicação

```diff
--- a/services/api/src/queue.ts
+++ b/services/api/src/queue.ts
@@
-await queue.publish({ userId });
+await queue.publish({ userId, requestId: requestContext.id });
```

```text
Arquivos alterados: 1
Passo executado: 3/6
Verificação: teste `queue.test.ts` passou
Próximo passo não executado: 4
```

Se um agente tocar `worker/src/consumer.ts` durante o passo 3, rejeite o resultado mesmo que compile: a lista de arquivos autorizados é parte do controle de escopo.

## 8. Protocol Simulation — gRPC bidirectional stream

### Prompt do cenário do PDF

```text
Simule um RPC bidirecional:
service GameSession {
  rpc PlayGame(stream GameAction) returns (stream GameState);
}

Mostre `Cliente -> Servidor` e `Servidor -> Cliente`, incluindo estado após cada troca:
1. PLAYER_JOIN
2. GAME_STATE inicial
3. MOVE válido e novo GAME_STATE
4. MOVE inválido e ERROR, mantendo o stream aberto
5. MOVE que causa GAME_OVER
6. QUIT e fechamento limpo

Depois da simulação, extraia handlers cliente/servidor. Inclua EOF, erro de transporte, ação fora de ordem e tentativa de jogar após GAME_OVER.
```

### Transcript esperado

```text
1 Cliente -> Servidor
   GameAction{type: PLAYER_JOIN, player: "player-123"}
   Estado: session=created, board=empty, turn=player-123

2 Servidor -> Cliente
   GameState{type: GAME_STATE, message: "Game created. Your turn, player-123.", board: ["...","...","..."]}

3 Cliente -> Servidor
   GameAction{type: MOVE, row: 0, col: 0}
   Estado: board[0][0]=X, turn=opponent

4 Servidor -> Cliente
   GameState{type: GAME_STATE, board: ["X..","...","..."]}

5 Cliente -> Servidor
   GameAction{type: MOVE, row: 0, col: 0}

6 Servidor -> Cliente
   GameState{type: ERROR, code: INVALID_MOVE, message: "cell occupied"}
   Estado: inalterado; stream continua.

7 Cliente -> Servidor
   GameAction{type: MOVE, row: 1, col: 1}
8 Servidor -> Cliente
   GameState{type: GAME_OVER, winner: "player-123"}
   Estado: gameOver=true; ações futuras rejeitadas.

9 Cliente -> Servidor
   GameAction{type: QUIT}
10 Servidor -> Cliente
   EOF/close; libera sessão.
```

### Mensagens e código Go adaptado

```go
type GameAction struct {
    Type   string
    Player string
    Row    int32
    Col    int32
}

type GameState struct {
    Type    string
    Message string
}

func (s *gameSessionServer) PlayGame(stream pb.GameSession_PlayGameServer) error {
    state := newGameState()
    playerID := ""
    gameOver := false

    for {
        action, err := stream.Recv()
        if err == io.EOF {
            log.Printf("client stream closed (player=%s)", playerID)
            return nil
        }
        if err != nil {
            return status.Errorf(codes.Unknown, "receive action: %v", err)
        }

        switch action.Type {
        case "PLAYER_JOIN":
            if playerID != "" || action.Player == "" {
                _ = stream.Send(&pb.GameState{Type: "ERROR", Message: "invalid join"})
                continue
            }
            playerID = action.Player
            _ = stream.Send(initialState(playerID))
        case "MOVE":
            if gameOver || !state.ApplyMove(int(action.Row), int(action.Col)) {
                _ = stream.Send(&pb.GameState{Type: "ERROR", Message: "invalid move"})
                continue
            }
            if state.IsGameOver() {
                gameOver = true
                _ = stream.Send(&pb.GameState{Type: "GAME_OVER"})
            } else {
                _ = stream.Send(state.AsProto())
            }
        case "QUIT":
            return nil
        default:
            _ = stream.Send(&pb.GameState{Type: "ERROR", Message: "unknown action"})
        }
    }
}
```

O transcript torna visíveis ordem, mensagens inválidas, erro sem fechamento e estado terminal antes de se escrever um handler.

## 9. Reversible Migration — PostgreSQL e TypeScript

### Prompt do cenário do PDF

```text
Tabela atual `users(id uuid primary key, email text unique, full_name text)`.
Divida `full_name` em `first_name` e `last_name`.
Preserve os dados. O código novo só usa first/last, mas o antigo ainda consulta full_name.

Produza:
1. forward SQL: colunas, backfill e camada de compatibilidade;
2. backward SQL: reversão limpa;
3. TypeScript com `pg` para leitura/escrita segura durante a transição;
4. verificações de idempotência, contagens e rollback em staging.
```

### Forward migration adaptada

```sql
BEGIN;

ALTER TABLE users ADD COLUMN IF NOT EXISTS first_name text;
ALTER TABLE users ADD COLUMN IF NOT EXISTS last_name text;

UPDATE users
SET first_name = NULLIF(split_part(trim(full_name), ' ', 1), ''),
    last_name = NULLIF(regexp_replace(trim(full_name), '^\S+\s*', ''), '')
WHERE first_name IS NULL AND last_name IS NULL AND full_name IS NOT NULL;

-- Compatibilidade para consultas legadas; o nome da view deve ser ajustado
-- se a aplicação já possuir uma tabela/view com esse nome.
CREATE OR REPLACE VIEW users_v1 AS
SELECT id, email, COALESCE(NULLIF(trim(concat_ws(' ', first_name, last_name)), ''), full_name) AS full_name
FROM users;

COMMIT;
```

### Backward migration adaptada

```sql
BEGIN;

UPDATE users
SET full_name = NULLIF(trim(concat_ws(' ', first_name, last_name)), '')
WHERE first_name IS NOT NULL OR last_name IS NOT NULL;

DROP VIEW IF EXISTS users_v1;
ALTER TABLE users DROP COLUMN IF EXISTS first_name;
ALTER TABLE users DROP COLUMN IF EXISTS last_name;

COMMIT;
```

Antes de executar o backward, confirme que `full_name` consegue representar toda a informação nova. Uma divisão irreversível — por exemplo, dados estruturados que não cabem no campo antigo — exige uma estratégia de preservação adicional; não declare rollback seguro apenas porque o SQL executa.

### Leitura/escrita durante a transição

```ts
import type { PoolClient } from "pg";

type UserInput = {
  id: string;
  firstName: string;
  lastName: string;
};

export async function readUser(client: PoolClient, id: string) {
  const result = await client.query(
    `SELECT id, email,
            COALESCE(NULLIF(trim(concat_ws(' ', first_name, last_name)), ''), full_name) AS display_name
       FROM users WHERE id = $1`,
    [id],
  );
  return result.rows[0] ?? null;
}

export async function writeUser(client: PoolClient, input: UserInput) {
  await client.query("BEGIN");
  try {
    const fullName = `${input.firstName} ${input.lastName}`.trim();
    await client.query(
      `UPDATE users
          SET first_name = $1, last_name = $2, full_name = $3
        WHERE id = $4`,
      [input.firstName, input.lastName, fullName, input.id],
    );
    await client.query("COMMIT");
  } catch (error) {
    await client.query("ROLLBACK");
    throw error;
  }
}
```

### Saída/verificação esperada

```text
Forward migration: applied
Rows before: 100000
Rows backfilled: 100000
Rows with empty first/last: 0
Legacy view query check: passed
New read/write check: passed
Forward re-run: no destructive changes, 0 additional rows changed
Backward migration on staging: applied
Rows after rollback: 100000
Checksum/display-name comparison: passed
```

## Checklist final de saída

Antes de aceitar uma resposta gerada com esta skill, confira:

- O risco dominante foi tratado pelo padrão certo?
- O objetivo, fora de escopo, invariantes e formato de saída estão explícitos?
- O exemplo de saída é verificável e não apenas uma promessa?
- O código tem testes ou casos de borda correspondentes?
- Um diff toca somente os arquivos autorizados?
- Duas implementações são realmente independentes?
- A rubric tem critérios observáveis e notas justificadas?
- O transcript cobre ordem, falha, estado e encerramento?
- Forward e backward preservam dados e são testados em staging?
- O agente sabe quando parar e pedir contexto em vez de inventar?
