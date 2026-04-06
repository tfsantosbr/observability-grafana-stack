# observability-grafana-stack

Stack de observabilidade completa baseada no ecossistema Grafana (LGTM) com coleta
centralizada via OpenTelemetry Collector. Cobre os três pilares de observabilidade:
**logs**, **métricas** e **traces** — todos correlacionados no Grafana.

## Índice

- [Arquitetura](#arquitetura)
- [Pré-requisitos](#pré-requisitos)
- [Iniciando a stack](#iniciando-a-stack)
- [Interfaces e acessos](#interfaces-e-acessos)
- [Configurando sua aplicação .NET](#configurando-sua-aplicação-net)
- [Verificando que os dados chegam](#verificando-que-os-dados-chegam)
- [Correlação entre sinais](#correlação-entre-sinais)
- [Adicionando dashboards](#adicionando-dashboards)
- [Habilitando autenticação no Grafana](#habilitando-autenticação-no-grafana)
- [Operações comuns](#operações-comuns)
- [Troubleshooting](#troubleshooting)
- [Estrutura do repositório](#estrutura-do-repositório)

---

## Arquitetura

```text
┌─────────────────────────────────────────────────────┐
│               Suas aplicações .NET                  │
│       (ASP.NET Core, Worker Service, Console)       │
└──────────────────────┬──────────────────────────────┘
                       │
          OTLP gRPC :4317  /  OTLP HTTP :4318
                       │
          ┌────────────▼────────────┐
          │  OpenTelemetry Collector │
          │  (fan-out + processors)  │
          └───┬──────────┬──────────┘
              │          │          │
           métricas    logs      traces
              │          │          │
     ┌────────▼──┐  ┌────▼───┐  ┌──▼────┐
     │Prometheus │  │  Loki  │  │ Tempo │
     │  :9090    │  │ :3100  │  │ :3200 │
     └────────┬──┘  └────┬───┘  └──┬────┘
              │           │         │
          ┌───▼───────────▼─────────▼───┐
          │           Grafana :3000      │
          │  (datasources provisionados) │
          └─────────────────────────────┘
```

| Componente          | Função                                                      |
|---------------------|-------------------------------------------------------------|
| **OTel Collector**  | Recebe OTLP, processa (batch, resource detection) e roteia  |
| **Prometheus**      | Armazena métricas com retenção de 15 dias                   |
| **Loki**            | Armazena logs com índice TSDB                               |
| **Tempo**           | Armazena traces distribuídos; gera service graph            |
| **Grafana**         | Visualização unificada com correlação entre os três sinais  |

---

## Pré-requisitos

- [Docker](https://docs.docker.com/get-docker/) ≥ 24
- [Docker Compose](https://docs.docker.com/compose/) ≥ 2.20 (incluso no Docker Desktop)

Verifique:

```bash
docker --version
docker compose version
```

---

## Iniciando a stack

### 1. Clone e entre no diretório

```bash
git clone https://github.com/seu-usuario/observability-grafana-stack.git
cd observability-grafana-stack
```

### 2. Suba todos os serviços em background

```bash
docker compose up -d
```

Na primeira execução o Docker baixa as imagens (~1-2 GB). As próximas inicializações
são instantâneas.

### 3. Confirme que todos os containers estão em execução

```bash
docker compose ps
```

Saída esperada (todos `running`):

```text
NAME             IMAGE                                    STATUS
grafana          grafana/grafana:10.4.2                   running
loki             grafana/loki:2.9.6                       running
otel-collector   otel/opentelemetry-collector-contrib:…   running
prometheus       prom/prometheus:v2.51.2                  running
tempo            grafana/tempo:2.4.1                      running
```

### 4. Acompanhe os logs (opcional)

```bash
# Todos os serviços
docker compose logs -f

# Apenas o collector (útil ao testar uma nova app)
docker compose logs -f otel-collector
```

### 5. Parando a stack

```bash
# Para os containers (preserva volumes/dados)
docker compose down

# Para e apaga todos os dados persistidos
docker compose down -v
```

---

## Interfaces e acessos

### Grafana — visualização principal

| Campo   | Valor                   |
|---------|-------------------------|
| URL     | <http://localhost:3000> |
| Usuário | *(acesso anônimo ativo)*|
| Senha   | *(sem autenticação)*    |

> O Grafana está configurado com acesso anônimo como `Admin`. Para habilitar login
> veja a seção [Habilitando autenticação no Grafana](#habilitando-autenticação-no-grafana).

Os três datasources já estão provisionados automaticamente:
**Prometheus**, **Loki** e **Tempo**.

Para verificar: **Connections → Data sources** no menu lateral.

---

### Prometheus — explorador de métricas

| Campo   | Valor                   |
|---------|-------------------------|
| URL     | <http://localhost:9090> |
| Usuário | *(sem autenticação)*    |
| Senha   | *(sem autenticação)*    |

Útil para validar se as métricas chegam antes de abrir o Grafana.
Navegue em **Graph** ou **Table** e execute uma query como `up`.

---

### Loki — logs

Loki não tem interface própria. Acesse via Grafana:
**Explore → datasource: Loki** e escreva uma query LogQL:

```logql
{job="minha-app"}
```

---

### Tempo — traces

Tempo não tem interface própria. Acesse via Grafana:
**Explore → datasource: Tempo** e busque por Trace ID ou use TraceQL:

```traceql
{ resource.service.name = "minha-app" }
```

---

## Configurando sua aplicação .NET

Aponte o exporter OTLP para o collector:

| Protocolo | Endpoint                | Quando usar                               |
|-----------|-------------------------|-------------------------------------------|
| gRPC      | `http://localhost:4317` | Padrão — menor overhead                   |
| HTTP      | `http://localhost:4318` | Ambientes que bloqueiam gRPC ou proxies   |

> Se sua aplicação roda no Docker na mesma rede `observability`, substitua
> `localhost` por `otel-collector`.

---

### Instalando os pacotes NuGet

```bash
dotnet add package OpenTelemetry.Extensions.Hosting
dotnet add package OpenTelemetry.Exporter.OpenTelemetryProtocol

# Auto-instrumentação ASP.NET Core
dotnet add package OpenTelemetry.Instrumentation.AspNetCore
dotnet add package OpenTelemetry.Instrumentation.Http
dotnet add package OpenTelemetry.Instrumentation.Runtime

# Métricas do processo e runtime
dotnet add package OpenTelemetry.Instrumentation.Process
```

---

### ASP.NET Core — `Program.cs`

```csharp
using OpenTelemetry.Logs;
using OpenTelemetry.Metrics;
using OpenTelemetry.Resources;
using OpenTelemetry.Trace;

var builder = WebApplication.CreateBuilder(args);

builder.Services
    .AddOpenTelemetry()
    .ConfigureResource(resource => resource
        .AddService(
            serviceName: "minha-app",
            serviceVersion: "1.0.0"))
    .WithTracing(tracing => tracing
        .AddAspNetCoreInstrumentation()
        .AddHttpClientInstrumentation()
        .AddOtlpExporter())
    .WithMetrics(metrics => metrics
        .AddAspNetCoreInstrumentation()
        .AddHttpClientInstrumentation()
        .AddRuntimeInstrumentation()
        .AddProcessInstrumentation()
        .AddOtlpExporter());

// Logs via OTel (substitui o provider padrão)
builder.Logging.AddOpenTelemetry(logging =>
{
    logging.IncludeFormattedMessage = true;
    logging.IncludeScopes = true;
    logging.ParseStateValues = true;
    logging.AddOtlpExporter();
});

var app = builder.Build();
app.Run();
```

---

### `appsettings.json`

```json
{
  "OTEL_SERVICE_NAME": "minha-app",
  "OTEL_EXPORTER_OTLP_ENDPOINT": "http://localhost:4317",
  "OTEL_EXPORTER_OTLP_PROTOCOL": "grpc"
}
```

---

### `appsettings.Development.json` (via `launchSettings.json`)

Outra opção é configurar via variáveis de ambiente no `launchSettings.json`
(não vai para o repositório, fica local):

```json
{
  "profiles": {
    "http": {
      "commandName": "Project",
      "environmentVariables": {
        "OTEL_SERVICE_NAME": "minha-app",
        "OTEL_EXPORTER_OTLP_ENDPOINT": "http://localhost:4317",
        "OTEL_EXPORTER_OTLP_PROTOCOL": "grpc",
        "OTEL_RESOURCE_ATTRIBUTES": "deployment.environment=dev,team=backend"
      }
    }
  }
}
```

---

### Worker Service / Console App

Para projetos sem ASP.NET Core, o padrão é o mesmo — basta não adicionar as
instrumentações de HTTP/ASP.NET:

```csharp
using OpenTelemetry.Resources;
using OpenTelemetry.Trace;

IHost host = Host.CreateDefaultBuilder(args)
    .ConfigureServices(services =>
    {
        services.AddOpenTelemetry()
            .ConfigureResource(r => r.AddService("minha-worker"))
            .WithTracing(tracing => tracing
                .AddOtlpExporter())
            .WithMetrics(metrics => metrics
                .AddRuntimeInstrumentation()
                .AddOtlpExporter());

        services.AddHostedService<MyWorker>();
    })
    .Build();

host.Run();
```

---

### Criando traces e métricas customizados

```csharp
using System.Diagnostics;
using System.Diagnostics.Metrics;

// Declare no topo da classe como static readonly
private static readonly ActivitySource Tracer = new("minha-app");
private static readonly Meter Meter = new("minha-app");
private static readonly Counter<long> PedidosCounter = Meter.CreateCounter<long>("pedidos.total");

// Use dentro dos métodos
public async Task ProcessarPedido(int pedidoId)
{
    using var activity = Tracer.StartActivity("ProcessarPedido");
    activity?.SetTag("pedido.id", pedidoId);

    PedidosCounter.Add(1, new KeyValuePair<string, object?>("status", "recebido"));

    try
    {
        // lógica do negócio...
        activity?.SetStatus(ActivityStatusCode.Ok);
    }
    catch (Exception ex)
    {
        activity?.SetStatus(ActivityStatusCode.Error, ex.Message);
        activity?.RecordException(ex);
        throw;
    }
}
```

> Registre o `ActivitySource` no OpenTelemetry para que os spans apareçam:
>
> ```csharp
> .WithTracing(tracing => tracing
>     .AddSource("minha-app")   // <-- nome do ActivitySource
>     .AddOtlpExporter())
> ```

---

### Correlacionando logs com o trace atual

O `ILogger` com o provider OTel já injeta `TraceId` e `SpanId` automaticamente
em cada log quando emitido dentro de um span ativo. Nenhuma configuração extra
é necessária.

Para garantir que o `trace_id` apareça nas mensagens de log (necessário para o
link Loki → Tempo no Grafana), adicione um scope no middleware:

```csharp
app.Use(async (context, next) =>
{
    var activity = Activity.Current;
    if (activity is not null)
    {
        using (logger.BeginScope(new Dictionary<string, object>
        {
            ["trace_id"] = activity.TraceId.ToString(),
            ["span_id"]  = activity.SpanId.ToString(),
        }))
        {
            await next();
        }
    }
    else
    {
        await next();
    }
});
```

---

## Verificando que os dados chegam

### Métricas

1. Abra <http://localhost:9090>
1. Execute a query: `up`
1. Deve aparecer `otel-collector` com valor `1`

Assim que sua app enviar métricas, consulte:

```promql
http_server_duration_milliseconds_bucket{job="minha-app"}
```

### Logs

1. Abra <http://localhost:3000>
1. Vá em **Explore** → selecione **Loki**
1. Escreva:

```logql
{job="minha-app"} |= "error"
```

### Traces

1. Abra <http://localhost:3000>
1. Vá em **Explore** → selecione **Tempo**
1. Busque pelo serviço:

```traceql
{ resource.service.name = "minha-app" }
```

---

## Correlação entre sinais

### Trace → Logs

No Tempo, ao abrir um span, clique no ícone do Loki para ver os logs daquela
requisição no mesmo intervalo de tempo. Requer que o `trace_id` esteja nos logs
(veja a seção de correlação de logs acima).

### Trace → Metrics (service graph)

O Tempo gera automaticamente um service graph a partir dos spans. Acesse via
**Explore → Tempo → Service Graph** para visualizar dependências entre serviços
e taxas de erro/latência.

### Log → Trace

Qualquer linha de log que contenha `trace_id=<valor>` ou `traceID=<valor>` vira
um link clicável direto para o trace no Tempo.

### Metrics → Traces (exemplars)

Habilite exemplars no exporter de métricas para vincular pontos no gráfico
diretamente a um trace:

```csharp
.WithMetrics(metrics => metrics
    .AddOtlpExporter(options =>
    {
        options.ExportProcessorType = ExportProcessorType.Batch;
    }))
```

---

## Adicionando dashboards

Coloque arquivos `.json` de dashboard na pasta `grafana/provisioning/dashboards/`.
Eles são carregados automaticamente pelo Grafana (sem precisar reiniciar).

Para exportar um dashboard existente:
**Dashboard → Share → Export → Save to file**.

Dashboards recomendados da comunidade Grafana:

| Dashboard                    | ID    |
|------------------------------|-------|
| ASP.NET Core                 | 19924 |
| .NET Runtime                 | 17707 |
| OpenTelemetry Collector      | 15983 |
| Loki Dashboard               | 13639 |

Importe via **Dashboards → New → Import** e informe o ID.

---

## Habilitando autenticação no Grafana

Por padrão o Grafana está com acesso anônimo. Para habilitar login, edite as
variáveis de ambiente do serviço `grafana` no `docker-compose.yml`:

```yaml
grafana:
  environment:
    - GF_AUTH_ANONYMOUS_ENABLED=false
    - GF_AUTH_DISABLE_LOGIN_FORM=false
    - GF_SECURITY_ADMIN_USER=admin
    - GF_SECURITY_ADMIN_PASSWORD=troque-esta-senha
```

Depois reinicie o serviço:

```bash
docker compose up -d grafana
```

Acesse <http://localhost:3000> com as credenciais definidas acima.

---

## Operações comuns

```bash
# Reiniciar apenas um serviço
docker compose restart otel-collector

# Recarregar config do Prometheus sem restart
curl -X POST http://localhost:9090/-/reload

# Ver uso de memória/CPU dos containers
docker stats

# Acessar o shell de um container
docker compose exec otel-collector sh

# Apagar apenas os dados de um backend (ex: Prometheus)
docker compose down
docker volume rm observability-grafana-stack_prometheus_data
docker compose up -d
```

---

## Troubleshooting

### Os dados não chegam no Grafana

1. Confirme que o collector está de pé:

```bash
docker compose ps otel-collector
docker compose logs otel-collector --tail=50
```

1. Teste o endpoint OTLP HTTP (deve retornar `405` — endpoint existe):

```bash
curl -i http://localhost:4318/v1/traces
```

1. Verifique se o exporter da app aponta para o host correto.
   Se a app roda no Docker na mesma rede, use `otel-collector:4317`, não `localhost:4317`.

### Prometheus não recebe métricas

```bash
curl -s "http://localhost:9090/api/v1/query?query=up" | jq .
docker compose logs prometheus --tail=30
```

### Loki rejeita logs com erro de timestamp

O Loki rejeita logs com mais de 1 semana de atraso (`reject_old_samples_max_age: 168h`).
Confirme que o clock da sua aplicação está sincronizado com o host.

### Tempo não armazena traces

```bash
docker compose logs tempo --tail=30
```

Se houver erro de permissão no volume:

```bash
docker compose exec tempo chown -R 10001:10001 /var/tempo
```

---

## Estrutura do repositório

```text
.
├── docker-compose.yml
├── otel-collector/
│   └── config.yaml              # receivers → processors → exporters
├── prometheus/
│   └── prometheus.yml           # scrape configs
├── loki/
│   └── config.yaml              # storage local, schema v13 (TSDB)
├── tempo/
│   └── config.yaml              # storage local, metrics generator (service graph)
└── grafana/
    └── provisioning/
        ├── datasources/
        │   └── datasources.yaml # Prometheus + Loki + Tempo (auto-provisionados)
        └── dashboards/
            └── dashboards.yaml  # loader — coloque seus .json aqui
```
