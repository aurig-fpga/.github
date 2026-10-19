# AURIG

**Open-source FPGA pipeline tooling for on-prem labs.**

Built by FPGA designers, used by FPGA designers.

## What is AURIG?

AURIG is a suite of open-source tools that work together to orchestrate the FPGA design lifecycle — from RTL parsing and linting to multi-vendor synthesis and nightly regression — without leaving your lab.

## The stack

- **AURIG Sentinel** — pipeline orchestrator (scheduling, regression, artifact bundling)
- **Build** — multi-vendor build engine (Vivado, Quartus, Diamond, Radiant)
- **AURIG Lint** — VHDL linter with configurable rule sets
- **AURIG Doc** — automatic documentation generator
- **AURIG Core** — shared library: VHDL parser, YAML manifest, utilities

Projects are described by one canonical YAML manifest, defined by a shared JSON Schema. The contract is set out in [decision 0001](https://github.com/aurig-fpga/.github/blob/main/decisions/0001-manifest-contract.md); its implementation across the tools is in progress.

## Status

AURIG is in early access and under active development. All five tools listed above are published as public repositories in this organization.

For early information, partnership inquiries, or professional services (setup, migration, support), contact us at [LogiMentor](https://logimentor.com).

## License

All AURIG tools are released under the Apache License 2.0.

Copyright 2026 LogiMentor S.r.l.

---

*AURIG is maintained by [LogiMentor S.r.l.](https://logimentor.com), an FPGA design consultancy based in Italy.*
