# 🔭 Stack de Observabilidade LGTM

Stack completa de observabilidade usando **Grafana LGTM** (Loki, Grafana, Tempo, Mimir) com OpenTelemetry Collector.

## 🚀 Quick Start

```bash
docker compose up -d
```

## 📊 Acessos

| Serviço | URL | Descrição |
|---------|-----|-----------|
| **Grafana** | http://localhost:3000 | Dashboard principal (sem login) |
| **Tempo** | http://localhost:3200 | API de traces |
| **Mimir** | http://localhost:9009 | API de métricas |
| **Loki** | http://localhost:3100 | API de logs |

---

## 🎯 Como Visualizar no Grafana

### 1. Traces (Tempo)

1. Acesse http://localhost:3000
2. Menu lateral → **Explore**
3. Selecione datasource **Tempo**
4. Use a aba **Search** para buscar traces por:
   - Service Name
   - Span Name
   - Duration
   - Tags (ex: `http.status_code=500`)

![Tempo Search](https://grafana.com/media/docs/tempo/screenshot-grafana-tempo-search-tab.png)

### 2. Métricas (Mimir)

1. Menu lateral → **Explore**
2. Selecione datasource **Mimir**
3. Queries úteis (métricas RED geradas dos spans):

```promql
# Taxa de requisições por serviço
sum(rate(traces_spanmetrics_calls_total[5m])) by (service_name)

# Taxa de erros
sum(rate(traces_spanmetrics_calls_total{status_code="STATUS_CODE_ERROR"}[5m])) by (service_name)

# Latência P95 por rota
histogram_quantile(0.95, 
  sum(rate(traces_spanmetrics_latency_bucket[5m])) by (le, http_route)
)

# Latência P99 geral
histogram_quantile(0.99, 
  sum(rate(traces_spanmetrics_latency_bucket[5m])) by (le)
)
```

### 3. Logs (Loki)

1. Menu lateral → **Explore**
2. Selecione datasource **Loki**
3. Queries úteis:

```logql
# Todos os logs
{}

# Logs por serviço
{service_name="meu-app"}

# Logs com erro
{service_name="meu-app"} |= "error"

# Logs com JSON parseado
{service_name="meu-app"} | json | level="error"
```

---

## 🔗 Correlação entre Sinais

### Trace → Logs
1. Abra um trace no Tempo
2. Clique em um span
3. Clique em **"Logs for this span"**

### Trace → Métricas
1. Abra um trace no Tempo
2. Clique em um span
3. Veja as métricas relacionadas no painel lateral

### Logs → Trace
1. Se o log contém `traceId`, clique no link **"View Trace"**

---

## 📈 Dashboards Recomendados

### Importar dashboards da comunidade:

1. Menu lateral → **Dashboards** → **Import**
2. Cole o ID do dashboard:

| Dashboard | ID | Descrição |
|-----------|-----|-----------|
| RED Metrics | `17175` | Métricas Rate/Error/Duration |
| Tempo / Traces | `16698` | Overview de traces |
| Loki / Logs | `13639` | Overview de logs |

---

## ⚙️ Configuração do Next.js

### Instalar dependências

```bash
npm install @vercel/otel @opentelemetry/sdk-logs @opentelemetry/api-logs
```

### Criar `instrumentation.ts` na raiz do projeto

```typescript
import { registerOTel } from '@vercel/otel'

export function register() {
  registerOTel({
    serviceName: 'meu-app-nextjs',
  })
}
```

### Configurar `next.config.js`

```javascript
/** @type {import('next').NextConfig} */
const nextConfig = {
  experimental: {
    instrumentationHook: true,
  },
}

module.exports = nextConfig
```

### Variáveis de ambiente (`.env.local`)

```env
OTEL_EXPORTER_OTLP_ENDPOINT=http://localhost:4318
OTEL_EXPORTER_OTLP_PROTOCOL=http/protobuf
```

---

## 🏗️ Arquitetura

```
┌─────────────────┐
│   Next.js App   │
│  (@vercel/otel) │
└────────┬────────┘
         │ OTLP (4317/4318)
         ▼
┌─────────────────────────────────────────────────┐
│           OpenTelemetry Collector               │
│  ┌─────────────────────────────────────────┐    │
│  │         spanmetrics connector           │    │
│  │   (gera métricas RED dos traces)        │    │
│  └─────────────────────────────────────────┘    │
└──────┬──────────────┬──────────────┬────────────┘
       │              │              │
       ▼              ▼              ▼
┌──────────┐   ┌──────────┐   ┌──────────┐
│  Tempo   │   │  Mimir   │   │   Loki   │
│ (traces) │   │(métricas)│   │  (logs)  │
└────┬─────┘   └────┬─────┘   └────┬─────┘
     │              │              │
     └──────────────┼──────────────┘
                    ▼
            ┌──────────────┐
            │   Grafana    │
            │ :3000        │
            └──────────────┘
```

---

## 📦 Portas

| Porta | Serviço | Protocolo |
|-------|---------|-----------|
| `3000` | Grafana | HTTP |
| `3100` | Loki | HTTP |
| `3200` | Tempo | HTTP |
| `4317` | Collector | gRPC (OTLP) |
| `4318` | Collector | HTTP (OTLP) |
| `9009` | Mimir | HTTP |

---

## 🛠️ Comandos Úteis

```bash
# Iniciar
docker compose up -d

# Parar
docker compose down

# Ver logs
docker compose logs -f

# Logs de um serviço específico
docker compose logs -f otel-collector

# Reiniciar collector após mudanças
docker compose restart otel-collector

# Limpar volumes (reset total)
docker compose down -v
```

---

## 🔍 Troubleshooting

### Traces não aparecem no Tempo

1. Verifique se o Collector está recebendo dados:
   ```bash
   docker compose logs otel-collector | grep -i trace
   ```

2. Verifique se o endpoint está correto no app:
   ```
   OTEL_EXPORTER_OTLP_ENDPOINT=http://localhost:4318
   ```

### Métricas spanmetrics não aparecem

1. Aguarde alguns segundos após gerar traces
2. Verifique no Mimir:
   ```promql
   {__name__=~"traces_spanmetrics.*"}
   ```

### Logs não aparecem no Loki

1. Certifique-se que seu app está enviando logs via OTLP
2. Verifique os logs do Collector:
   ```bash
   docker compose logs otel-collector | grep -i log
   ```

---

## 📚 Referências

- [OpenTelemetry Collector](https://opentelemetry.io/docs/collector/)
- [Grafana Tempo](https://grafana.com/docs/tempo/latest/)
- [Grafana Mimir](https://grafana.com/docs/mimir/latest/)
- [Grafana Loki](https://grafana.com/docs/loki/latest/)
- [Vercel OTel](https://nextjs.org/docs/app/building-your-application/optimizing/open-telemetry)

