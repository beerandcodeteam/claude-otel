# Claude Code OTel Stack

Stack local de observabilidade para coletar métricas e logs do **Claude Code** via OpenTelemetry, com armazenamento em Prometheus + Loki e visualização no Grafana.

## Arquitetura

```
Claude Code (OTLP/gRPC)
        │
        ▼
OTel Collector (4317)
   ├── metrics ──► Prometheus (9090)
   └── logs    ──► Loki (3100)
                     │
                     ▼
                 Grafana (3000)
```

Componentes (ver `docker-compose.yml`):

| Serviço          | Porta | Descrição                                          |
| ---------------- | ----- | -------------------------------------------------- |
| `otel-collector` | 4317 / 4318 / 9464 | Recebe OTLP do Claude Code e exporta para os backends |
| `prometheus`     | 9090  | Armazena métricas (retenção 90 dias)               |
| `loki`           | 3100  | Armazena logs (retenção 720h)                      |
| `grafana`        | 3000  | Dashboards e exploração                            |

## Como subir

```bash
docker compose up -d
```

Verifique:

```bash
docker compose ps
curl -s http://localhost:9464/metrics | head   # métricas saindo do collector
```

## Variáveis de ambiente do Claude Code

Exporte no shell onde o `claude` será executado (ou adicione ao seu `~/.bashrc` / `~/.zshrc`):

```bash
# Liga a telemetria do Claude Code
export CLAUDE_CODE_ENABLE_TELEMETRY=1

# Exportadores OTLP
export OTEL_METRICS_EXPORTER=otlp
export OTEL_LOGS_EXPORTER=otlp
export OTEL_EXPORTER_OTLP_PROTOCOL=grpc
export OTEL_EXPORTER_OTLP_ENDPOINT=http://localhost:4317

# CRÍTICO para Prometheus — sem isso, contadores ficam "delta"
# e o backend descarta silenciosamente
export OTEL_EXPORTER_OTLP_METRICS_TEMPORALITY_PREFERENCE=cumulative

# Habilita "título de sessão" via primeiro prompt nos logs
export OTEL_LOG_USER_PROMPTS=1

# Opcional — atributos de recurso para segmentar (multi-time, multi-projeto)
export OTEL_RESOURCE_ATTRIBUTES="team.id=beerandcode,project=civic-chatbot"

# Para debug inicial, intervalo de export menor (ms)
export OTEL_METRIC_EXPORT_INTERVAL=10000
```

### Explicação de cada variável

| Variável | O que faz |
| --- | --- |
| `CLAUDE_CODE_ENABLE_TELEMETRY` | Liga o pipeline de telemetria do Claude Code. Sem isso, nada é emitido. |
| `OTEL_METRICS_EXPORTER` / `OTEL_LOGS_EXPORTER` | Define que métricas e logs saem via OTLP. |
| `OTEL_EXPORTER_OTLP_PROTOCOL` | Usa gRPC (porta 4317) — mais eficiente que HTTP/protobuf. |
| `OTEL_EXPORTER_OTLP_ENDPOINT` | Endereço do collector. `localhost:4317` se o stack roda na mesma máquina. |
| `OTEL_EXPORTER_OTLP_METRICS_TEMPORALITY_PREFERENCE=cumulative` | **Obrigatório para Prometheus.** O default é `delta`, que o Prometheus descarta sem aviso, resultando em painéis vazios. |
| `OTEL_LOG_USER_PROMPTS=1` | Inclui o conteúdo do primeiro prompt do usuário nos logs — facilita identificar sessões no Grafana. |
| `OTEL_RESOURCE_ATTRIBUTES` | Atributos extras anexados a todas as métricas/logs (`team.id`, `project`, etc.). O collector os promove a labels Prometheus (`resource_to_telemetry_conversion: enabled`). |
| `OTEL_METRIC_EXPORT_INTERVAL` | Intervalo em ms entre envios. `10000` (10s) é útil em debug; em produção use o default (60s). |

### Testando

Em um terminal com as variáveis exportadas, rode o `claude` normalmente. Depois consulte:

```bash
# Métricas devem aparecer no scrape do Prometheus
curl -s http://localhost:9464/metrics | grep claude
```

## Grafana

### 1. Primeiro acesso e criação de usuário

1. Abra `http://localhost:3000`
2. Login inicial:
   - **usuário:** `admin`
   - **senha:** `admin`
   (definidos em `docker-compose.yml` via `GF_SECURITY_ADMIN_USER` / `GF_SECURITY_ADMIN_PASSWORD`)
3. O Grafana pedirá para trocar a senha no primeiro login — defina uma senha forte.
4. (Opcional) Para criar usuários adicionais:
   - Menu lateral → **Administration → Users and access → Users → New user**
   - Preencha nome, email, senha e atribua a função (`Admin`, `Editor` ou `Viewer`).

Os datasources **Prometheus** e **Loki** já vêm provisionados automaticamente via `grafana/provisioning/datasources/datasources.yml`.

### 2. Importando o dashboard oficial do Claude Code

A Anthropic publica um dashboard de referência. Para importá-lo:

1. No Grafana, vá em **Dashboards → New → Import**.
2. Você tem duas formas de importar:
   - **Via ID do Grafana.com:** cole o ID do dashboard publicado pela Anthropic (procure por *"Claude Code"* em https://grafana.com/grafana/dashboards/) e clique em **Load**.
   - **Via JSON:** baixe o JSON do dashboard (do repo oficial da Anthropic ou de https://grafana.com/grafana/dashboards/) e cole em **Import via panel json**.
3. Quando o Grafana pedir, selecione:
   - **Prometheus** → datasource `Prometheus` (já provisionado)
   - **Loki** → datasource `Loki` (já provisionado)
4. Clique em **Import**.

Se o dashboard usar variáveis como `team_id` ou `project`, elas serão populadas pelos labels gerados a partir do `OTEL_RESOURCE_ATTRIBUTES` configurado acima.

### Dashboard não mostra dados?

- Confirme que `OTEL_EXPORTER_OTLP_METRICS_TEMPORALITY_PREFERENCE=cumulative` está exportado **antes** de iniciar o `claude`.
- Verifique no Prometheus (`http://localhost:9090`) se existem séries começando com `claude_` — em **Status → Targets** o job `claude-code-metrics` deve estar `UP`.
- Confira logs do collector: `docker compose logs -f otel-collector`.

## Arquivos do projeto

```
.
├── docker-compose.yml                          # Stack completo
├── otel-collector-config.yaml                  # Pipeline OTLP → Prometheus/Loki
├── prometheus.yml                              # Scrape do collector
├── loki-config.yaml                            # Loki single-binary, filesystem
└── grafana/
    └── provisioning/
        └── datasources/
            └── datasources.yml                 # Prometheus + Loki pré-configurados
```

## Parar e limpar

```bash
docker compose down           # para os containers
docker compose down -v        # remove também os volumes (apaga métricas e logs)
```
