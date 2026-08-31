# Build stage

The build stage assembles the workbook data into the final dashboard-ready structure.

## Responsibilities

- read descriptive status data by region and office
- extract rich-text time-series change values and significance
- compute reverse-coded direction and comparable thresholds
- merge RC values where sponsorship applies
- inject data into a reusable dashboard template and generate the final HTML output

## Planned scripts

- `extract_descriptive.py`
- `extract_trends.py`
- `extract_rc.py`
- `assemble_data.py`
- `inject_template.py`
- `generate_output.py`

## Output contract

The build stage should produce a single self-contained dashboard payload written to `output/index.html` and supporting region metadata if needed.
