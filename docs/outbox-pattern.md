# Outbox Pattern — Implementação no DotNetCore.CAP

## O Problema: Dual-Write

Imagine um cenário clássico em microsserviços: um serviço de pedidos precisa:
1. Salvar o pedido no banco de dados
2. Publicar o evento `PedidoCriado` no message broker para outros serviços reagirem

A implementação ingênua seria:

```csharp
// ⚠️ PROBLEMA: dual-write sem garantia de atomicidade
await _dbContext.Pedidos.AddAsync(pedido);
await _dbContext.SaveChangesAsync();
// --- PONTO DE FALHA ---
await _messageBus.PublishAsync("pedidos.criado", pedido); // pode falhar!
```

**Cenários de falha:**

| Falha | Consequência |
|---|---|
| App cai entre `SaveChanges` e `Publish` | Pedido salvo, evento nunca enviado — serviços downstream nunca ficam sabendo |
| Broker fora do ar no momento do Publish | Idem — mensagem perdida |
| Broker aceita, mas App cai antes de commitar | Evento enviado, pedido não salvo — serviços ficam com dados fantasma |

Qualquer um desses cenários leva a **inconsistência de dados** — difícil de detectar e corrigir.

---

## A Solução: Outbox Pattern

O padrão **Transactional Outbox** resolve o dual-write ao transformar os dois passos em um único:

> Gravar o evento no banco de dados **junto** com a operação de negócio, na mesma transação ACID.

```
┌──────────────────────────────────────────────────────┐
│                   TRANSAÇÃO ÚNICA                    │
│                                                      │
│  INSERT INTO pedidos (...)                           │
│  INSERT INTO outbox (topic, payload, status)         │
│                                                      │
│                       COMMIT                         │
└──────────────────────────────────────────────────────┘

         (após o commit, de forma assíncrona)

┌──────────────────────────────────────────────────────┐
│              Background Process                      │
│                                                      │
│  SELECT * FROM outbox WHERE status = 'pending'       │
│  → Envia para o broker                               │
│  UPDATE outbox SET status = 'sent'                   │
└──────────────────────────────────────────────────────┘
```

**Garantia:** ou ambos (pedido + evento) são persistidos, ou nenhum é. Não há estado intermediário inconsistente.

---

## Como o CAP Implementa o Outbox Pattern

### Tabelas Criadas Automaticamente

O CAP cria e gerencia duas tabelas no seu banco de dados:

**`cap.published`** — mensagens a serem enviadas (Outbox):
```sql
Id          BIGINT
Version     NVARCHAR(20)
Name        NVARCHAR(200)    -- tópico
Content     NVARCHAR(MAX)    -- payload serializado
Retries     INT
Added       DATETIME
ExpiresAt   DATETIME
StatusName  NVARCHAR(40)     -- Scheduled, Succeeded, Failed...
```

**`cap.received`** — mensagens recebidas do broker (Inbox):
```sql
Id          BIGINT
Version     NVARCHAR(20)
Name        NVARCHAR(200)    -- tópico
Group       NVARCHAR(200)    -- consumer group
Content     NVARCHAR(MAX)    -- payload serializado
Retries     INT
Added       DATETIME
ExpiresAt   DATETIME
StatusName  NVARCHAR(40)     -- Processing, Succeeded, Failed...
```

### Publicando com Atomicidade

```csharp
// ✅ CORRETO: usando a transação do EF
using (var trans = await _dbContext.Database.BeginTransactionAsync(_capBus, autoCommit: true))
{
    _dbContext.Pedidos.Add(pedido);           // grava pedido
    await _dbContext.SaveChangesAsync();

    await _capBus.PublishAsync("pedidos.criado", pedido);  // grava na outbox
    // autoCommit: true → commit automático ao final do using
}
// Neste ponto, AMBOS o pedido E a entrada na outbox foram commitados juntos
```

```csharp
// ✅ CORRETO: usando ADO.NET diretamente
using var conn = new SqlConnection(_connectionString);
using var trans = conn.BeginTransaction(_capBus, autoCommit: true);

var cmd = conn.CreateCommand();
cmd.Transaction = trans;
cmd.CommandText = "INSERT INTO Pedidos ...";
cmd.ExecuteNonQuery();

_capBus.Publish("pedidos.criado", pedido); // grava na outbox
// autoCommit: true → commit ao final do using
```

### Background Dispatcher

Um `IHostedService` executa continuamente em background:

```
Loop a cada N segundos:
  1. SELECT * FROM cap.published WHERE StatusName = 'Scheduled' LIMIT batch_size
  2. Para cada mensagem:
     a. Envia ao broker
     b. Se ACK recebido → UPDATE StatusName = 'Succeeded'
     c. Se falha → mantém 'Scheduled', incrementa Retries
  3. Se Retries >= MaxRetries → StatusName = 'Failed'
```

---

## Idempotência — O Lado do Consumidor

O Outbox garante que mensagens não serão *perdidas*. Mas brokers em modo *at-least-once delivery* podem *reentregas* a mesma mensagem mais de uma vez (ex: se o consumer cair antes de confirmar o ACK).

O CAP mitiga isso com a tabela **`cap.received`** (o "Inbox"):

1. Quando uma mensagem chega do broker, o CAP verifica se ela já existe na tabela `received`
2. Se já existe com status `Succeeded`, a mensagem é descartada (deduplicação)
3. Caso contrário, é persistida e processada

**Observação:** a idempotência de negócio ainda é de responsabilidade do handler. O CAP garante *best-effort deduplication*, mas recomenda-se que handlers sejam idempotentes por design.

---

## Configuração de Retry

```csharp
services.AddCap(x =>
{
    x.FailedRetryCount = 5;                    // máximo de retentativas (padrão: 50)
    x.FailedRetryInterval = 60;                // intervalo em segundos (padrão: 60)
    x.SucceedMessageExpiredAfter = 24 * 3600;  // expirar mensagens bem-sucedidas após N segundos
    x.FailedThresholdCallback = (type, name, content) =>
    {
        // Callback chamado quando uma mensagem excede o limite de retries
        // Útil para alertas, dead-letter queues customizadas, etc.
        _logger.LogError("Mensagem falhou definitivamente: {Topic}", name);
    };
});
```

---

## Comparação: Com e Sem Outbox

| Aspecto | Sem Outbox | Com CAP (Outbox) |
|---|---|---|
| Atomicidade | ❌ Não garantida | ✅ ACID com o banco |
| Mensagem perdida | ✅ Possível | ❌ Impossível após commit |
| Mensagem duplicada ao produzir | ✅ Possível | ❌ Controlada |
| Broker fora do ar | ❌ Falha imediata | ✅ Mensagem fica na fila local |
| Complexidade de implementação | Simples | Moderada (gerenciada pelo CAP) |
| Rastreabilidade | Difícil | ✅ Dashboard + tabelas auditáveis |

---

## Quando Não Usar o Outbox Pattern

- **Sistemas simples sem consistência crítica**: se a perda eventual de um evento for aceitável, o overhead do Outbox pode não valer a pena
- **Latência ultra-baixa**: o processo assíncrono de dispatch introduz um atraso (geralmente milissegundos a poucos segundos)
- **Stack sem banco relacional/transacional**: em sistemas puramente em memória ou com brokers como único storage, o Outbox perde seu mecanismo de garantia

---

## Referências

- [Pattern: Transactional Outbox — microservices.io](https://microservices.io/patterns/data/transactional-outbox.html)
- [eShop on .NET — Designing Atomicity and Resiliency](https://docs.microsoft.com/en-us/dotnet/standard/microservices-architecture/multi-container-microservice-net-applications/subscribe-events#designing-atomicity-and-resiliency-when-publishing-to-the-event-bus)
- [Documentação oficial do CAP](https://cap.dotnetcore.xyz/)
- [`docs/event-flow.md`](./event-flow.md) — Fluxo completo de um evento
- [`docs/architecture.md`](./architecture.md) — Componentes e arquitetura geral
