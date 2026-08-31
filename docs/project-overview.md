# Project Overview

This repository is organized around the dashboard build playbook in [AIM_Regional_Dashboard_Agent_Instructions.md](../AIM_Regional_Dashboard_Agent_Instructions.md).

## Architecture summary

The intended implementation is a workbook-driven pipeline with three stages:

### 1. Audit

Responsible for:
- finding the region's country columns in each sheet
- identifying section boundaries by column A patterns
- locating the NOTE row and deriving reverse-coded indicators
- confirming sheet-level assumptions against the workbook instead of hardcoded values

### 2. Build

Responsible for:
- reading descriptive values
- extracting time-series deltas and significance from rich-text cells
- computing RC / sponsorship gaps where present
- merging the three layers by indicator key
- preserving duplicate and raw-code disambiguation rules
- patching the reusable dashboard template with region-specific data

### 3. Verify

Responsible for:
- independent re-read checks for extracted values
- comparison of trend and descriptive divergence
- reverse-code validation
- headless render tests
- governance checks on self-containment and output constraints

## Governance principles

- Audit before build
- Derive inputs from the workbook each time
- Flag low-n or non-comparable results
- Never present conflicting current values for a single indicator
- Use the NOTE row as the source of truth for reverse-coding
- Ensure a generated html file is valid, self-contained, and safe to host

## Future implementation note

This project intentionally leaves the actual scripts as a placeholder structure so the repo can expand without losing the workflow logic defined in the markdown instructions.
