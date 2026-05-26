# AURIG

**Open-source FPGA pipeline tooling for on-prem labs.**

Three layers, one manifest. Built by FPGA designers, used by FPGA designers.

## What is AURIG?

AURIG is a suite of open-source tools that work together to orchestrate the FPGA design lifecycle — from RTL parsing and linting to multi-vendor synthesis and nightly regression — without leaving your lab.

## The stack

- **Sentinel** — pipeline orchestrator (scheduling, regression, artifact bundling)
- **Build** — multi-vendor build engine (Vivado, Quartus, Diamond, Radiant)
- **Lint** — VHDL linter with configurable rule sets
- **Doc** — automatic documentation generator
- **Core** — shared library: VHDL parser, YAML manifest, utilities

All five tools share a single canonical project manifest in YAML.

## Status

AURIG is in active development. Public repositories will be published progressively in the coming weeks.

For early information, partnership inquiries, or professional services (setup, migration, support), contact us at [LogiMentor](https://logimentor.com).

## License

All AURIG tools are released under the Apache License 2.0.

Copyright 2026 LogiMentor S.r.l.

---

*AURIG is maintained by [LogiMentor S.r.l.](https://logimentor.com), an FPGA design consultancy based in Italy.*
