# Astronomy Shop architecture

Architecture of the [OpenTelemetry Demo](https://github.com/open-telemetry/opentelemetry-demo) (Astronomy Shop), derived from its Compose files, Envoy routes, and Collector configs rather than from the upstream docs.

![Astronomy Shop architecture](architecture.svg)

- `architecture.mmd`: Mermaid source (edit this)
- `architecture.svg` / `architecture.png`: rendered output for slides and video

Mermaid also renders inline on GitHub:

```mermaid
%%{init: {"flowchart": {"nodeSpacing": 40, "rankSpacing": 70, "curve": "basis"}}}%%
flowchart TB
  user([Browser / Load Generator<br/>Python + Locust])
  envoy[frontend-proxy<br/>Envoy]

  subgraph app[Application services · gRPC unless noted]
    direction TB
    frontend[frontend<br/>TypeScript / Next.js]
    subgraph leaf[Backend services]
      direction LR
      ad[ad<br/>Java]
      cart[cart<br/>.NET]
      currency[currency<br/>C++]
      reco[recommendation<br/>Python]
      catalog[product-catalog<br/>Go]
      shipping[shipping<br/>Rust]
      quote[quote<br/>PHP]
      payment[payment<br/>Node.js]
      email[email<br/>Ruby]
    end
    checkout[checkout<br/>Go]
    images[image-provider<br/>nginx]
  end

  subgraph data[Data stores and messaging]
    direction LR
    valkey[(valkey-cart)]
    pg[(PostgreSQL)]
    kafka{{Kafka}}
  end

  subgraph async[Async consumers]
    direction LR
    accounting[accounting<br/>.NET]
    fraud[fraud-detection<br/>Kotlin]
  end

  subgraph flags[Feature flags]
    direction LR
    flagd[flagd]
    flagdui[flagd-ui<br/>Elixir]
  end

  subgraph obs[Observability]
    direction LR
    otelcol[[OTel Collector]]
    jaeger[Jaeger]
    prom[Prometheus]
    opensearch[OpenSearch]
    grafana[Grafana]
    opamp[OpAMP server]
  end

  user -->|HTTP| envoy
  envoy -->|"/"| frontend
  envoy -->|"/images/"| images
  envoy -->|"/feature"| flagdui
  envoy -->|"/otlp-http/"| otelcol
  envoy -.->|"/jaeger/ /grafana/ /opamp/"| obs

  frontend --> ad & cart & currency & reco & catalog & shipping
  frontend --> checkout
  checkout --> cart & currency & catalog & payment & shipping
  checkout -->|HTTP| email
  reco --> catalog
  shipping -->|HTTP| quote

  cart --> valkey
  catalog --> pg
  checkout -->|produce| kafka
  kafka -->|consume| accounting & fraud
  accounting --> pg
  flagdui -.->|writes flag file| flagd

  flagd -.->|flag evaluation| app
  flagd -.-> async

  app ==>|OTLP| otelcol
  async ==>|OTLP| otelcol
  envoy ==>|OTLP| otelcol
  data -.->|kafka / postgresql / redis receivers| otelcol
  otelcol -->|traces| jaeger
  otelcol -->|metrics| prom
  otelcol -->|logs| opensearch
  otelcol <-.->|OpAMP| opamp
  grafana -.->|query| jaeger & prom & opensearch
```

## Reading the diagram

| Line style | Meaning |
|---|---|
| Solid | Request path (gRPC unless labeled HTTP) |
| Thick `==>` | OTLP telemetry export to the Collector |
| Dotted | Control plane or scrape: feature-flag evaluation, Collector receivers, Grafana queries, OpAMP |

## Telemetry pipeline

All instrumented services send OTLP to a single Collector (contrib distribution, gateway role). With `compose.observability.yaml` layered on, it fans out to:

| Signal | Exporter | Backend |
|---|---|---|
| Traces | `otlp_grpc/jaeger` (and `span_metrics` connector) | Jaeger |
| Metrics | `otlp_http/prometheus` | Prometheus |
| Logs | `opensearch` | OpenSearch |

The Collector also scrapes Kafka, PostgreSQL, Valkey (redis receiver), and Docker stats. Browser telemetry reaches it through Envoy at `/otlp-http/`.

## Caveats

- **Compose layering:** the base `compose.yaml` does not include Kafka, `accounting`, or `fraud-detection`. They come from `compose.full.yaml`. Jaeger, Prometheus, Grafana, OpenSearch, and OpAMP come from `compose.observability.yaml`. The diagram shows the fully layered stack.
- **Simplified edges:** flagd is consumed by nearly every service and the load generator via OpenFeature. Drawing each edge produced unreadable output, so it is shown as one edge to the app and async groups.
- **Snapshot:** generated from upstream `main` on 2026-09-28. Upstream changes often (for example `chatbot`, `mcp`, and `agent` services exist under `src/` but are not drawn). Re-verify before recording a video against a pinned demo version.

## Regenerating

```bash
npx -y @mermaid-js/mermaid-cli -i architecture.mmd -o architecture.svg -b white
npx -y @mermaid-js/mermaid-cli -i architecture.mmd -o architecture.png -b white -s 2
```

If Puppeteer cannot find a browser, pass `-p` with a config JSON containing `executablePath` for a local Chrome.
