# Fluxo de Eventos no DotNetCore.CAP

Este documento descreve o caminho completo de uma mensagem, desde o momento em que é publicada pela aplicação até a execução do subscriber correspondente.

---

## Fluxo de Publicação (Producer Side)

```
Aplicação
   │
   ▼
ICapPublisher.Publish("topico", payload)
   │
   │  (dentro de uma transação de banco de dados)
   ▼
┌────────────────────────────────────────┐
│  Tabela published_messages (no banco)  │
│  Status: Scheduled                     │
│  Topic: "topico"                       │
│  Payload: { ... }                      │
└──────────────────┬─────────────────────┘
                   │
         COMMIT da transação
                   │
                   ▼
        Background Dispatcher
        (IHostedService — polling)
                   │
                   ▼
         Lê mensagens Scheduled
                   │
                   ▼
┌──────────────────────────────────────┐
│         Message Broker               │
│   (RabbitMQ / Kafka / etc.)          │
│   Exchange / Topic: "topico"         │
└──────────────────────────────────────┘
                   │
        Confirmação de entrega
                   │
                   ▼
        Status → Succeeded
```

### Passos Detalhados

1. **Aplicação chama `_capBus.Publish(...)`** — opcionalmente dentro de uma transação ADO.NET ou EF
2. **CAP grava a mensagem** na tabela `published_messages` com status `Scheduled` — **na mesma transação** de negócio
3. **Commit da transação** — neste ponto, a mensagem está segura no banco
4. **Background Dispatcher** (polling) detecta mensagens `Scheduled` e as envia ao broker
5. **Broker confirma o recebimento** (ACK do publish)
6. **Status atualizado para `Succeeded`** na tabela local

> Se o envio ao broker falhar nos steps 4-5, a mensagem permanece em `Scheduled` e o Dispatcher tenta novamente após o intervalo de retry configurado.

---

## Fluxo de Consumo (Consumer Side)

```
Message Broker
   │
   │  Mensagem em "topico"
   ▼
CAP Transport Consumer
(assinado ao broker no startup)
   │
   ▼
┌───────────────────────────────────────┐
│  Tabela received_messages (no banco)  │
│  Status: Processing                   │
│  Topic: "topico"                      │
│  Payload: { ... }                     │
└──────────────────┬────────────────────┘
                   │
                   ▼
        Subscriber Registry
        (busca handler para "topico")
                   │
                   ▼
┌───────────────────────────────────────┐
│  Handler encontrado:                  │
│  [CapSubscribe("topico")]             │
│  public void ProcessarEvento(...)     │
└──────────────────┬────────────────────┘
                   │
                   ▼
           Execução do handler
                   │
          ┌────────┴────────┐
       Sucesso           Falha
          │                 │
          ▼                 ▼
   Status: Succeeded   Status: Failed
                            │
                    Retry após intervalo
                    (até N tentativas)
```

### Passos Detalhados

1. **CAP Transport Consumer** recebe a mensagem do broker via assinatura ativa
2. **Mensagem persistida** na tabela `received_messages` com status `Processing`
3. **Broker recebe ACK** — a mensagem não será reentregue pelo broker (ACK é enviado antes de processar para evitar duplicatas a nível de broker)
4. **Subscriber Registry** encontra o handler anotado com `[CapSubscribe("topico")]`
5. **Handler é invocado** com deserialização automática do payload
6. **Em caso de sucesso:** status → `Succeeded`
7. **Em caso de falha (exceção):** status → `Failed`, e o Retry Processor tentará novamente após o intervalo configurado

---

## Fluxo de Retry (Reprocessamento)

```
Retry Processor (Background)
   │
   ▼
Lê mensagens com Status = Failed
AND retried_count < max_retries
   │
   ▼
Reexecuta o handler
   │
   ├── Sucesso → Status: Succeeded
   └── Falha   → retried_count++
                  │
                  Se retried_count >= max_retries
                  │
                  ▼
              Status: Skipped
              (requer intervenção manual via Dashboard)
```

---

## Estados de uma Mensagem

| Estado | Publicadas | Recebidas | Descrição |
|---|---|---|---|
| `Scheduled` | ✅ | — | Gravada no banco aguardando envio ao broker |
| `Processing` | — | ✅ | Recebida do broker, handler em execução |
| `Succeeded` | ✅ | ✅ | Processada com sucesso |
| `Failed` | ✅ | ✅ | Falha no envio ou processamento, aguarda retry |
| `Skipped` | ✅ | ✅ | Excedeu o número máximo de retries |

---

## Diagrama Completo (Publicação + Consumo)

```
┌─────────────────────┐         ┌─────────────────────────┐
│   Serviço A          │         │   Serviço B              │
│   (Producer)         │         │   (Consumer)             │
│                     │         │                          │
│  [Negócio]          │         │  [CapSubscribe("pedido")] │
│  + CAP Publish      │         │  ProcessarPedido(...)    │
│        │            │         │         ▲                │
│        ▼            │         │         │                │
│  [DB: published]    │         │  [DB: received]          │
│        │            │         │         │                │
│        ▼            │         │         │                │
│  [Dispatcher BG]    │         │  [Fetcher BG]            │
└─────────┬───────────┘         └──────────┬───────────────┘
          │                                │
          │     ┌─────────────────┐        │
          └────►│   Message Broker │◄───────┘
                │  (RabbitMQ etc) │
                └─────────────────┘
```

---

## Referências

- [Documentação oficial — Getting Started](https://cap.dotnetcore.xyz/user-guide/en/getting-started/quick-start/)
- [`docs/outbox-pattern.md`](./outbox-pattern.md) — Detalhamento do padrão Outbox
- [`docs/architecture.md`](./architecture.md) — Componentes e arquitetura geral
