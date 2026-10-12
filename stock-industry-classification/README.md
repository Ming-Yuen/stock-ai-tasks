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
- Process `pending/` files from oldest to newest.
- Every ChatGPT run should continue until all processable CSV files in `pending/` have been handled, unless a transient GitHub/network/service error requires the run to stop safely.
- Move a file from `pending/` to `processing/` before editing and commit that move.
- Move a fully processed file to `finished/`, then commit and push before continuing with the next file.
- `imported/` is owned by Spring Boot after successful database import.
- Use `error/` only for files that cannot be processed safely because of data/schema/identity problems.
- For transient GitHub/network/service failures, keep the file in `processing/` for retry instead of moving it to `error/`.
