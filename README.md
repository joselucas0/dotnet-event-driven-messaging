<p align="center">
  <img height="140" src="https://raw.githubusercontent.com/dotnetcore/CAP/master/docs/content/img/logo.svg" alt="CAP Logo">
</p>

# DotNetCore.CAP — Estudo e Documentação

[![Build](https://github.com/dotnetcore/CAP/actions/workflows/deploy-docs-and-dashboard.yml/badge.svg?branch=master)](https://github.com/dotnetcore/CAP/actions/workflows/deploy-docs-and-dashboard.yml)
[![AppVeyor](https://ci.appveyor.com/api/projects/status/v8gfh6pe2u2laqoa/branch/master?svg=true)](https://ci.appveyor.com/project/yang-xiaodong/cap/branch/master)
[![NuGet](https://img.shields.io/nuget/v/DotNetCore.CAP.svg)](https://www.nuget.org/packages/DotNetCore.CAP/)
[![Licença MIT](https://img.shields.io/badge/license-MIT-blue.svg)](https://raw.githubusercontent.com/dotnetcore/CAP/master/LICENSE.txt)


>
> **Documentação técnica detalhada:** [`docs/`](./docs/)

---

## O que é o CAP?

CAP é uma biblioteca .NET que oferece uma solução leve, fácil de usar e de alto desempenho para **transações distribuídas** e **integração via Event Bus**.

Em arquiteturas de microsserviços, a comunicação entre serviços via eventos é essencial — mas simplesmente usar uma fila de mensagens não garante a consistência dos dados. Uma publicação pode falhar na metade do processo, deixando o sistema em um estado inconsistente.

O CAP resolve esse problema integrando uma **tabela de mensagens local** (local message table) ao banco de dados da aplicação — uma implementação do **Outbox Pattern**. Com isso, a gravação no banco e a publicação do evento são parte da mesma transação ACID, garantindo que **nenhuma mensagem seja perdida**.

Além disso, o CAP pode ser usado como um **Event Bus standalone**, tornando a publicação e assinatura de eventos mais simples sem a necessidade de herdar interfaces complexas.

---

## Por que estudar esse projeto?

Quando comecei a explorar microsserviços, percebi que um dos problemas mais sutis e difíceis de debugar é justamente a **inconsistência eventual causada por falhas parciais na comunicação entre serviços**. O CAP endereça esse problema de forma elegante e pragmática.

Meus objetivos de aprendizado com este repositório:

- Entender o **Outbox Pattern** na prática e como ele garante consistência
- Ver como um **Event Bus** é implementado sobre diferentes message brokers (RabbitMQ, Kafka, etc.)
- Explorar a integração do CAP com **Entity Framework** e **ADO.NET**
- Estudar o **dashboard integrado** para monitoramento de mensagens
- Analisar a arquitetura extensível via plugins de transporte e armazenamento

> Para a análise arquitetural detalhada, veja [`docs/architecture.md`](./docs/architecture.md).

---

## Funcionalidades Principais

### Core

| Funcionalidade | Descrição |
|---|---|
| **Transações Distribuídas** | Garante consistência entre microsserviços via tabela de mensagens local (Outbox Pattern) |
| **Event Bus** | Bus de eventos de alto desempenho para comunicação desacoplada |
| **Entrega Garantida** | Mensagens nunca são perdidas; falhas disparam retentativas automáticas |

### Mensageria Avançada

| Funcionalidade | Descrição |
|---|---|
| **Mensagens com Atraso** | Publicação de mensagens com delay nativo, sem depender de features do broker |
| **Subscrições Flexíveis** | Suporta atributos, wildcards (`*`, `#`) e partial topic subscriptions |
| **Consumer Groups / Fan-Out** | Implementa padrões de competing consumers ou broadcast de forma simples |
| **Processamento Paralelo e Serial** | Configure consumidores para alto throughput ou execução ordenada |
| **Backpressure** | Controle automático de velocidade de processamento para evitar OOM |

### Extensibilidade

| Funcionalidade | Descrição |
|---|---|
| **Arquitetura Plugável** | Suporte a múltiplos brokers (RabbitMQ, Kafka, Azure Service Bus...) e bancos (SQL Server, PostgreSQL, MongoDB...) |
| **Sistemas Heterogêneos** | Mecanismos para integração com sistemas legados ou sem CAP |
| **Filtros e Serialização** | Pipeline customizável com filtros e suporte a diferentes formatos de serialização |

### Observabilidade

| Funcionalidade | Descrição |
|---|---|
| **Dashboard em Tempo Real** | Interface web para monitorar mensagens, status e acionar retentativas manualmente |
| **Service Discovery** | Integração com Consul e Kubernetes para descoberta de nós em ambientes distribuídos |
| **OpenTelemetry** | Instrumentação nativa para distributed tracing end-to-end |

---

## Visão Geral da Arquitetura

![Arquitetura CAP](https://raw.githubusercontent.com/dotnetcore/CAP/master/docs/content/img/architecture-new.png)

O CAP implementa o **Outbox Pattern**: ao publicar um evento, a mensagem é primeiro gravada em uma tabela local no banco de dados, dentro da mesma transação do negócio. Um processo em background então lê essa tabela e encaminha as mensagens ao broker. Isso elimina o problema clássico de *dual-write* (escrever no banco e no broker de forma independente).

> Para um detalhamento completo do fluxo, veja [`docs/event-flow.md`](./docs/event-flow.md).
> Para o aprofundamento no padrão Outbox, veja [`docs/outbox-pattern.md`](./docs/outbox-pattern.md).

---

## Instalação

### Pacote Principal

```shell
PM> Install-Package DotNetCore.CAP
```

### Transportes (Message Brokers)

```shell
PM> Install-Package DotNetCore.CAP.Kafka
PM> Install-Package DotNetCore.CAP.RabbitMQ
PM> Install-Package DotNetCore.CAP.AzureServiceBus
PM> Install-Package DotNetCore.CAP.AmazonSQS
PM> Install-Package DotNetCore.CAP.NATS
PM> Install-Package DotNetCore.CAP.RedisStreams
PM> Install-Package DotNetCore.CAP.Pulsar
```

### Armazenamento (Bancos de Dados)

A tabela de log de eventos será criada e integrada ao banco que você escolher.

```shell
PM> Install-Package DotNetCore.CAP.SqlServer
PM> Install-Package DotNetCore.CAP.MySql
PM> Install-Package DotNetCore.CAP.PostgreSql
PM> Install-Package DotNetCore.CAP.MongoDB    // Requer MongoDB 4.0+ em cluster
```

---

## Configuração

Configure o CAP no `Program.cs` (ou `Startup.cs` em projetos mais antigos):

```csharp
public void ConfigureServices(IServiceCollection services)
{
    // Contexto do Entity Framework (se aplicável)
    services.AddDbContext<AppDbContext>();

    services.AddCap(x =>
    {
        // Usando Entity Framework — o CAP detecta a connection string automaticamente
        x.UseEntityFramework<AppDbContext>();

        // Usando ADO.NET diretamente
        x.UseSqlServer("sua-connection-string");
        x.UseMySql("sua-connection-string");
        x.UsePostgreSql("sua-connection-string");

        // Usando MongoDB (requer cluster 4.0+)
        x.UseMongoDB("sua-connection-string");

        // Escolha o transporte de mensagens
        x.UseRabbitMQ("hostname");
        x.UseKafka("connection-string");
        x.UseAzureServiceBus("connection-string");
        x.UseAmazonSQS(options => { /* ... */ });
        x.UseNATS("connection-string");
        x.UsePulsar("connection-string");
        x.UseRedisStreams("connection-string");
    });
}
```

---

## Publicando Mensagens

Injete `ICapPublisher` no seu controller ou serviço. A partir da versão 7.0, mensagens com atraso também são suportadas.

```csharp
public class PublishController : Controller
{
    private readonly ICapPublisher _capBus;

    public PublishController(ICapPublisher capPublisher)
    {
        _capBus = capPublisher;
    }

    // Publicando dentro de uma transação ADO.NET (auto-commit)
    [Route("~/adonet/transaction")]
    public IActionResult AdonetComTransacao()
    {
        using (var connection = new MySqlConnection(ConnectionString))
        using (var transaction = connection.BeginTransaction(_capBus, autoCommit: true))
        {
            // Lógica de negócio...
            _capBus.Publish("pedidos.criado", DateTime.Now);
        }
        return Ok();
    }

    // Publicando dentro de uma transação Entity Framework (auto-commit)
    [Route("~/ef/transaction")]
    public IActionResult EFComTransacao([FromServices] AppDbContext dbContext)
    {
        using (var trans = dbContext.Database.BeginTransaction(_capBus, autoCommit: true))
        {
            // Lógica de negócio...
            _capBus.Publish("pedidos.criado", DateTime.Now);
        }
        return Ok();
    }

    // Publicando mensagem com atraso de 30 segundos
    [Route("~/publish/delay")]
    public async Task<IActionResult> PublicarComAtraso()
    {
        await _capBus.PublishDelayAsync(TimeSpan.FromSeconds(30), "pedidos.criado", DateTime.Now);
        return Ok();
    }
}
```

> **Ponto chave:** ao usar `BeginTransaction`, a mensagem só é enfileirada no broker **depois** que a transação do banco de dados é confirmada. É essa atomicidade que garante a consistência.

---

## Assinando Mensagens

### Em um Controller

Adicione o atributo `[CapSubscribe]` diretamente na action:

```csharp
public class PedidosController : Controller
{
    [CapSubscribe("pedidos.criado")]
    public void AoReceberPedido(DateTime horaDoCriação)
    {
        Console.WriteLine($"Pedido recebido em: {horaDoCriação}");
    }
}
```

### Em um Serviço de Negócio

Se o subscriber estiver fora de um controller, a classe deve implementar `ICapSubscribe`:

```csharp
namespace MeuApp.Servicos
{
    public interface IServicoDePedidos
    {
        void ProcessarPedido(DateTime horaDoCriação);
    }

    public class ServicoDePedidos : IServicoDePedidos, ICapSubscribe
    {
        [CapSubscribe("pedidos.criado")]
        public void ProcessarPedido(DateTime horaDoCriação)
        {
            // Processar o pedido recebido
        }
    }
}
```

Registre o serviço no `ConfigureServices`:

```csharp
services.AddTransient<IServicoDePedidos, ServicoDePedidos>();
services.AddCap(x => { /* ... */ });
```

### Assinatura Assíncrona

```csharp
public class AssinantesAssincronos : ICapSubscribe
{
    [CapSubscribe("pedidos.criado")]
    public async Task ProcessarAsync(Mensagem msg, CancellationToken cancellationToken)
    {
        await AlgumaOperacaoAsync(msg, cancellationToken);
    }
}
```

### Partial Topic Subscriptions

Agrupe tópicos em nível de classe para organizar subscribers relacionados:

```csharp
[CapSubscribe("clientes")]
public class ClientesSubscriber : ICapSubscribe
{
    // Escuta o tópico "clientes.criar"
    [CapSubscribe("criar", isPartial: true)]
    public void AoCriar(Cliente cliente) { /* ... */ }

    // Escuta o tópico "clientes.atualizar"
    [CapSubscribe("atualizar", isPartial: true)]
    public void AoAtualizar(Cliente cliente) { /* ... */ }
}
```

### Grupos de Subscribers (Consumer Groups)

Similar aos consumer groups do Kafka. Permitem balancear a carga entre múltiplas instâncias de um serviço ou implementar broadcast.

- **Mesmo grupo:** apenas um consumidor recebe a mensagem (competing consumers)
- **Grupos diferentes:** todos os consumidores recebem (fan-out / broadcast)

```csharp
// Apenas um dos dois handlers será executado para cada mensagem
[CapSubscribe("pedidos.criado", Group = "servico-de-estoque")]
public void AtualizarEstoque(DateTime data) { /* ... */ }

[CapSubscribe("pedidos.criado", Group = "servico-de-notificacao")]
public void EnviarNotificacao(DateTime data) { /* ... */ }
```

Configurando o grupo padrão:

```csharp
services.AddCap(x =>
{
    x.DefaultGroup = "meu-servico-default";
});
```

---

## Dashboard

O CAP inclui um dashboard web para monitoramento em tempo real das mensagens publicadas e recebidas, incluindo status de entrega e opção de reenvio manual.

```shell
PM> Install-Package DotNetCore.CAP.Dashboard
```

Disponível por padrão em `http://localhost:xxx/cap`. Para customizar o caminho:

```csharp
services.AddCap(x =>
{
    x.UseDashboard(opt => { opt.PathMatch = "/meu-dashboard"; });
});
```

### Service Discovery para Dashboard Distribuído

- **Consul:** [Ver documentação](https://cap.dotnetcore.xyz/user-guide/en/monitoring/consul/)
- **Kubernetes:** Instale `DotNetCore.CAP.Dashboard.K8s` — [Ver documentação](https://cap.dotnetcore.xyz/user-guide/en/monitoring/kubernetes/)

---

## Suporte ao Azure Service Bus Emulator

O [emulador do Azure Service Bus](https://learn.microsoft.com/en-us/azure/service-bus-messaging/overview-emulator) usa portas separadas para AMQP (5672) e a API HTTP de administração (5300). Como o CAP usa uma única connection string, é necessário desabilitar o `AutoProvision` e criar as entidades manualmente:

```csharp
services.AddCap(x =>
{
    x.UseAzureServiceBus(opt =>
    {
        opt.ConnectionString = "Endpoint=sb://localhost;SharedAccessKeyName=...;UseDevelopmentEmulator=true;";
        opt.AutoProvision = false; // Entidades devem existir previamente
    });
});
```

---

## Documentação Técnica

Para aprofundamento técnico na arquitetura e nos padrões utilizados:

| Documento | Descrição |
|---|---|
| [`docs/architecture.md`](./docs/architecture.md) | Visão geral da arquitetura, componentes e motivações de design |
| [`docs/event-flow.md`](./docs/event-flow.md) | Fluxo completo de um evento, do publish ao consume |
| [`docs/outbox-pattern.md`](./docs/outbox-pattern.md) | Aprofundamento no Outbox Pattern implementado pelo CAP |

---

## Licença
> **Nota:** Este repositório é um fork do projeto original [dotnetcore/CAP](https://github.com/dotnetcore/CAP). Estou utilizando-o como base de estudo para entender na prática como implementar **mensageria orientada a eventos** e **transações distribuídas** em ecossistemas .NET. A documentação abaixo foi escrita por mim em português, combinando o conteúdo original com minhas anotações de aprendizado.
Este repositório é um fork. O projeto original é licenciado sob a [Licença MIT](https://github.com/dotnetcore/CAP/blob/master/LICENSE.txt).
