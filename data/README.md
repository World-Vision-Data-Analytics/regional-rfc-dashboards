# Data and workbook inputs

This directory is reserved for the source workbook and any region metadata required by the build.

## Expected inputs

- FY26 AIM / AIM+ executive dashboard workbook
- region roster metadata, including country order and short codes
- sponsorship / RC availability flag for the region
- sub-office rules when a region splits country offices

## Suggested structure

- `workbooks/` — raw workbook files
- `metadata/` — region roster and static config
- `notes/` — audit notes and anomaly logs

## Design intent

All data acquisition should happen from workbook contents and workbook-derived metadata. This directory is intentionally simple and acts as a controlled input layer for the build pipeline.
