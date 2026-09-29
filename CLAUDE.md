# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

Companion resources for the KubeSkills OpenTelemetry videos. It **builds on** the upstream [OpenTelemetry Demo (Astronomy Shop)](https://opentelemetry.io/docs/demo/) for learning purposes; it is not a fork or repackaging, and the README says so explicitly. Keep that framing in any new docs. There is no application code, build, or test suite: the contents are docs, a diagram, and Helm values extracted from upstream.

## Layout

- `docs/architecture/`: architecture diagram of the demo. `architecture.mmd` is the source of truth; `architecture.svg` and `architecture.png` are rendered from it.
- `helm/values-components.yaml`: partial extract of the upstream Helm values (`components`, `jaeger`, `prometheus`, `grafana`, `opensearch` keys only). Not a complete values file.
- `NOTICE`, `third_party/opentelemetry/LICENSE`: Apache-2.0 attribution for material derived from upstream.

## Working with the diagram

Regenerate the SVG and PNG after any edit to `architecture.mmd` (run from `docs/architecture/`):

```bash
npx -y @mermaid-js/mermaid-cli -i architecture.mmd -o architecture.svg -b white
npx -y @mermaid-js/mermaid-cli -i architecture.mmd -o architecture.png -b white -s 2
```

- mermaid-cli needs a browser. If Puppeteer can't find one, pass `-p <config.json>` containing `{"executablePath": "/Applications/Google Chrome.app/Contents/MacOS/Google Chrome", "args": ["--no-sandbox"]}`.
- `-w`/`--width` are not accepted by the installed mermaid-cli; use `-s` for PNG scale.
- **`docs/architecture/README.md` embeds a second copy of the Mermaid source** in a fenced block for GitHub inline rendering. Update it together with `architecture.mmd`, or the two diverge.
- The diagram was derived from upstream `compose*.yaml`, `src/frontend-proxy/envoy.tmpl.yaml`, and `src/otel-collector/otelcol-config*.yml`, not from prose docs. Re-derive from those if upstream changes. Some services (`chatbot`, `mcp`, `agent`) exist upstream but are intentionally not drawn.

## Upstream-derived files

- `helm/values-components.yaml` comes from `charts/opentelemetry-demo/values.yaml` in `open-telemetry/opentelemetry-helm-charts` (not from `opentelemetry-demo`). The block contents must stay **verbatim**; only the header comment is ours. Do not hand-edit values in it. Pull changes from upstream instead.
- When adding or changing a copied or derived file: keep its Apache-2.0 header (copyright, SPDX id, source, list of changes) and update the matching entry in `NOTICE`, including the upstream version and date.
- `helm/values-default.yaml` (full upstream copy) is deliberately untracked. Don't commit it without also adding a header and a `NOTICE` entry.

## Conventions

- YAML should pass `yamllint`; it is not installed locally, so at minimum confirm files parse (e.g. `ruby -ryaml -e 'YAML.load_file("helm/values-components.yaml")'`).
- The upstream values contain known demo-grade issues (hardcoded passwords, `busybox:latest` init containers, memory limits without requests, no NetworkPolicies, open Grafana/OpenSearch). They are left as-is in the verbatim extract. Flag them, but fix them in new manifests rather than in the extract.
