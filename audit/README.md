# Audit stage

The audit stage validates workbook structure and region-specific assumptions before data extraction begins.

## Responsibilities

- detect the correct region country columns in each global dashboard sheet
- locate section boundaries in column A
- extract the NOTE row and reverse-code roster
- confirm the presence or absence of RC / sponsorship data
- capture workbook anomalies for later reporting

## Planned scripts

- `detect_country_columns.py`
- `detect_sections.py`
- `parse_note_row.py`
- `validate_roster.py`
- `audit_summary.py`

## Output contract

The audit stage should emit summary data that downstream build steps can consume without embedding workbook assumptions directly in the render code.
