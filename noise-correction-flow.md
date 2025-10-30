### Render the noise-correction flow diagram as PNG

Use Graphviz to render `noise-correction-flow.dot` to PNG:

```bash
brew install graphviz  # macOS (if needed)
dot -Tpng docs/noise-correction-flow.dot -o docs/noise-correction-flow.png
```

If you prefer SVG:

```bash
dot -Tsvg docs/noise-correction-flow.dot -o docs/noise-correction-flow.svg
```

The DOT file models the flow in `src/main/python/wfa/measurement/reporting/postprocessing/report/report.py` from inputs, spec construction and constraints, solving with `noiseninja.Solver`, rebuilding the corrected report, through to large-correction detection and pre/post quality summaries.


