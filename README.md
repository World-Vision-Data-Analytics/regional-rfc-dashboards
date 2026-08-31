# AIM Regional Dashboard Project

This repository is a holding structure for the AIM / AIM+ FY26 regional indicator dashboard workflow described in [AIM_Regional_Dashboard_Agent_Instructions.md](AIM_Regional_Dashboard_Agent_Instructions.md).

The goal is to preserve the audit → build → verify process in a reusable, region-agnostic project layout while keeping the actual workbook parsing logic and HTML generation as future implementation work.

## Project intent

- Reproduce a regional dashboard from the FY26 AIM / AIM+ workbook
- Keep the build region-agnostic and workbook-driven
- Preserve the core governance rules: audit before build, verify before publish
- Output a single self-contained HTML file named index.html

## Repository layout

- [audit/](audit/) — workbook discovery, sheet detection, section detection, and NOTE parsing
- [build/](build/) — data assembly, reverse-code handling, trend extraction, and template injection
- [verify/](verify/) — validation, reconciliation, and headless render checks
- [templates/](templates/) — HTML template and any derived patch notes
- [data/](data/) — workbook inputs, roster metadata, and source material
- [output/](output/) — generated dashboard output and deployment notes
- [docs/](docs/) — summary architecture and implementation guidance

## Working model

This project follows the same sequence prescribed by the agent instructions:

1. Audit the workbook and detect the region-specific sheet layout
2. Build the data model and inject it into the reusable dashboard template
3. Verify every assertion before publishing the output

## Important operational notes

- The workbook is the single source of truth.
- Country columns must be detected per sheet rather than hardcoded.
- Additional indicators are shipped as an opt-in filter and flagged explicitly.
- The rendered page must be self-contained and hostable on a static or intranet server.
- SharePoint / OneDrive raw HTML should not be treated as a render target.

## Source reference

- [AIM_Regional_Dashboard_Agent_Instructions.md](AIM_Regional_Dashboard_Agent_Instructions.md)

## Planned future work

This scaffold intentionally does not yet contain the final Python extraction scripts. It is organized so that the next implementation step can add:

- workbook loader scripts
- section and code parsers
- reverse-coding logic
- template patching and data injection
- verification harnesses
- HTML output generation
