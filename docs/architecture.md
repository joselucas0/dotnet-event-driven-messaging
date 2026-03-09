# Arquitetura do DotNetCore.CAP

## Visão Geral

O CAP é estruturado em camadas bem definidas que separam as preocupações de **transporte de mensagens**, **persistência** e **lógica de entrega**. Essa separação é o que permite sua arquitetura plugável, suportando diferentes combinações de brokers e bancos de dados sem alterar o núcleo.

```
┌─────────────────────────────────────────────────────┐
│                   Sua Aplicação                      │
│   (Controllers, Services, Hosted Services...)        │
└──────────────────────┬──────────────────────────────┘
                       │  ICapPublisher / [CapSubscribe]
┌──────────────────────▼──────────────────────────────┐
│                   CAP Core (núcleo)                  │
│  ┌─────────────────┐   ┌──────────────────────────┐ │
│  │  Publisher      │   │  Subscriber / Dispatcher │ │
│  │  (Outbox Write) │   │  (Outbox Read + Invoke)  │ │
│  └────────┬────────┘   └────────────┬─────────────┘ │
└───────────┼────────────────────────┼───────────────┘
            │                        │
┌───────────▼──────────┐  ┌──────────▼──────────────┐
│  Storage Provider     │  │  Transport Provider      │
│  (SQL/Mongo/Postgres) │  │  (RabbitMQ/Kafka/etc.)   │
└──────────────────────┘  └──────────────────────────┘
```

---

## Componentes Principais

### 1. `ICapPublisher`

Interface injetada nos serviços da aplicação para publicar eventos. Internamente, ao chamar `Publish(...)`, o CAP **não envia a mensagem diretamente ao broker** — ele grava a mensagem na tabela local do banco de dados, dentro da transação corrente.

**Por que isso importa:** se a transação falhar (rollback), a mensagem também não é persistida. Zero dual-write.

### 2. Storage Provider (Persistência)

Responsável por manter o log de mensagens publicadas e recebidas. Cada mensagem tem um estado (`Scheduled`, `Processing`, `Succeeded`, `Failed`).

Implementações disponíveis:

| Pacote | Banco de Dados |
|---|---|
| `DotNetCore.CAP.SqlServer` | SQL Server |
| `DotNetCore.CAP.MySql` | MySQL / MariaDB |
| `DotNetCore.CAP.PostgreSql` | PostgreSQL |
| `DotNetCore.CAP.MongoDB` | MongoDB 4.0+ |
| `DotNetCore.CAP.InMemoryStorage` | In-Memory (testes) |

### 3. Transport Provider (Mensageria)

Responsável por enviar e receber mensagens do broker. O CAP abstrai as diferenças entre os brokers, expondo uma interface uniforme.

| Pacote | Broker |
|---|---|
| `DotNetCore.CAP.RabbitMQ` | RabbitMQ |
| `DotNetCore.CAP.Kafka` | Apache Kafka |
| `DotNetCore.CAP.AzureServiceBus` | Azure Service Bus |
| `DotNetCore.CAP.AmazonSQS` | Amazon SQS |
| `DotNetCore.CAP.NATS` | NATS |
| `DotNetCore.CAP.RedisStreams` | Redis Streams |
| `DotNetCore.CAP.Pulsar` | Apache Pulsar |

### 4. Processor (Background Worker)

Um ou mais `IHostedService` em background que executa continuamente:
- **Dispatcher:** lê mensagens com status `Scheduled` no banco e as envia ao broker
- **Fetcher:** consome mensagens do broker e as persiste como `Received` no banco, para então invocar os handlers locais
- **Retry Processor:** reprocessa mensagens `Failed` após o intervalo configurado

### 5. `[CapSubscribe]` — Subscriber Registry

O CAP escaneia a aplicação no startup e registra todos os métodos anotados com `[CapSubscribe]`, criando um mapeamento `tópico → handler`. Quando uma mensagem chega do broker, o dispatcher busca o handler correspondente e o invoca.

### 6. Dashboard

Pacote opcional (`DotNetCore.CAP.Dashboard`) que expõe uma interface web para:
- Visualizar histórico de mensagens (publicadas e recebidas)
- Checar status de entrega individual
- Retentar mensagens falhas manualmente
- Navegar entre nós em ambientes distribuídos (via Consul ou K8s)

---

## Motivações de Design

### Por que um Outbox local em vez de publicar direto no broker?

A publicação direta no broker cria o problema de **dual-write**: você grava no banco de dados E no broker de forma independente. Se o sistema cair entre um e outro, os dados ficam inconsistentes — uma mensagem perdida ou uma mensagem enviada sem a operação de negócio ter sido confirmada.

O Outbox Pattern resolve isso transformando os dois passos em um só: a mensagem é gravada no banco **dentro da mesma transação ACID** do negócio. O envio ao broker fica para depois, via processo assíncrono. Se o broker estiver fora do ar, a mensagem fica em fila no banco e é enviada quando o serviço voltar.

### Por que a arquitetura plugável?

Diferentes equipes usam diferentes stacks. Ao separar o núcleo dos providers de transporte e armazenamento, o CAP permite que você troque de RabbitMQ para Kafka, ou de SQL Server para PostgreSQL, com mínima mudança de código — apenas a configuração muda.

### Por que manter o log de mensagens recebidas também?

Idempotência. Ao persistir as mensagens recebidas, o CAP consegue detectar duplicatas (mensagens entregues mais de uma vez pelo broker) e evitar reprocessamento. Isso é fundamental em sistemas distribuídos onde brokers guarantem *at-least-once delivery*.

---

## Diagrama Oficial de Arquitetura

![Arquitetura do CAP](https://raw.githubusercontent.com/dotnetcore/CAP/master/docs/content/img/architecture-new.png)

> Fonte: repositório oficial [dotnetcore/CAP](https://github.com/dotnetcore/CAP)

---

## Referências

- [Documentação oficial do CAP](https://cap.dotnetcore.xyz/)
- [eShop on .NET — Designing Atomicity and Resiliency](https://docs.microsoft.com/en-us/dotnet/standard/microservices-architecture/multi-container-microservice-net-applications/subscribe-events#designing-atomicity-and-resiliency-when-publishing-to-the-event-bus)
- [Outbox Pattern — Microsoft](https://learn.microsoft.com/en-us/azure/architecture/best-practices/transactional-outbox-cosmos)
