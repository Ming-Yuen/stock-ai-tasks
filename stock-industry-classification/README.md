# Stock Industry Classification AI Task

CSV schema (exact order):

```csv
figi,symbol,industry,sub_industry,valuation_industry
```

Rules:
- Keep the CSV file name, header, and row count unchanged. Do not split or merge files.
- `figi` and `symbol` are identifiers and must never be changed.
- AI may update only `industry`, `sub_industry`, and `valuation_industry`.
- Existing classification values are hints only and must be re-checked.
- If a classification cannot be determined reliably, leave that field blank. Do not guess.
- Move a file from `pending/` to `processing/` before editing.
- Move a fully processed file to `finished/`.
- `imported/` is owned by Spring Boot after successful database import.
- Use `error/` for files that cannot be processed safely.
