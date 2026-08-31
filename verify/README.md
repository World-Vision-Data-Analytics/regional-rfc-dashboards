# Verify stage

The verify stage ensures the dashboard is faithful to the workbook and free from structural or logic defects before publication.

## Responsibilities

- run independent re-reads of workbook values
- reconcile descriptive statistics, trends, and RC gaps
- verify reverse-coding logic against the workbook NOTE row
- exercise the render in a headless DOM context
- confirm self-containment and governance requirements

## Planned scripts

- `reconcile_descriptive.py`
- `reconcile_trends.py`
- `reconcile_rc.py`
- `render_test.js`
- `final_audit.py`

## Output contract

This stage should produce a pass/fail summary and block publication if any checks fail.
