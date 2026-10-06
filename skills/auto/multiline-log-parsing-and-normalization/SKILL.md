---
name: multiline-log-parsing-and-normalization
description: Use when parsing multiline log files to extract structured error entries.
---
- Parse timestamps and convert all to UTC ISO 8601 format.
- Normalize service names to lower-case with dashes replaced by underscores.
- Extract log level in uppercase and filter for error levels only.
- Extract the main message and the last line of any traceback as the exception.
- Detect and sum repeated message counts from lines like '-- last message repeated N times --'.
- Sort errors by service name and timestamp ascending.
- Output a top-level JSON object with schema_version and generated_by fields.
