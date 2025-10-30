### Noise Correction Flow (Mermaid)

```mermaid
flowchart LR
  %% Layout
  classDef box fill:#fff,stroke:#333,stroke-width:1px,rx:4px,ry:4px;
  classDef note fill:#f9f9f9,stroke:#bbb,stroke-width:1px,rx:4px,ry:4px,color:#333;
  classDef comp fill:#eef7ff,stroke:#2b6cb0,stroke-width:1px,rx:4px,ry:4px;
  classDef proc fill:#fdf6ec,stroke:#b7791f,stroke-width:1px,rx:4px,ry:4px;

  subgraph Inputs
    direction TB
    A1["Weekly cumulative reach\n(by EDP combination)\n(value, sigma, name)"]:::box
    A2["Whole campaign: reach, k-reach (k=1..F), impressions\n(value, sigma, name)"]:::box
    A3["Weekly non-cumulative: reach, k-reach, impressions\n(value, sigma, name)"]:::box
  end

  MR["MetricReport\n(getters for all measurements)"]:::comp
  RP["Report\n(aggregates metrics, relationships, population)"]:::comp

  A1 --> MR
  A2 --> MR
  A3 --> MR
  MR --> RP

  subgraph Spec_Construction["Spec construction: to_set_measurement_spec"]
    direction TB
    S1["_add_*_measurements_to_spec\n(normalize sigma; preserve 0 as fixed)"]:::proc
    S2["_add_set_relations_to_spec (constraints)"]:::proc
    S2a["subset relations\n(child <= parent; reach <= impressions)"]:::box
    S2b["cover relations\n(sum(children) >= union)"]:::box
    S2c["overlap constraints\n(ordered sets inequalities)"]:::box
    S2d["cumulative monotonicity\n(non-decreasing over time)"]:::box
    S2e["final cumulative = whole campaign reach"]:::box
    S2f["reach = sum over k of k-reach"]:::box
    S2g["impressions >= sum over k of k*(k-reach)"]:::box
    S2h["cross-metric ordering\n(child metric <= parent metric)"]:::box
    S1 --> S2
    S2 --> S2a
    S2 --> S2b
    S2 --> S2c
    S2 --> S2d
    S2 --> S2e
    S2 --> S2f
    S2 --> S2g
    S2 --> S2h
  end

  RP --> S1

  SV["noiseninja.Solver(spec).\nsolve_and_translate()\n-> solution + status"]:::box
  S2h --> SV

  subgraph Reconstruction["Reconstruction: report_from_solution"]
    direction TB
    R1["map solution indices -> measurement names"]:::proc
    R2["rebuild MetricReport(s)\n(cumulative, whole, weekly non-cumulative)"]:::proc
    RC["Corrected Report"]:::comp
    R1 --> R2 --> RC
  end

  SV --> R1

  L["Large correction check\n|delta| > max(7*sigma, 1) -> log + record"]:::note
  Qpre["Pre-correction quality\n(zero-variance consistency; union CI)"]:::note
  Qpost["Post-correction quality\n(zero-variance consistency; union CI)"]:::note

  RP --> Qpre
  RC --> Qpost
  RP --> L

  RES["ReportPostProcessorResult\nstatus, pre/post quality, large_corrections"]:::box
  RC --> RES
  SV --> RES
  Qpre --> RES
  Qpost --> RES
  L --> RES
```

#### Render locally (optional)

- With VS Code or Cursor: many Markdown preview plugins render Mermaid blocks natively.
- With Mermaid CLI (mmdc):
  - npm i -g @mermaid-js/mermaid-cli
  - mmdc -i docs/noise-correction-flow-mermaid.md -o docs/noise-correction-flow.png


