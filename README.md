# gha-biorxiv-stats-action

Weekly stats of papers submitted to [bioRxiv](https://www.biorxiv.org/) and
[medRxiv](https://www.medrxiv.org/), filtered to a configurable category set
and written as deduplicated weekly CSV files.

![Version](https://img.shields.io/badge/version-0.1.0-8A2BE2)
[![License](https://img.shields.io/badge/license-Apache--2.0-blue)](LICENSE)
[![Update rxiv stats](https://github.com/Lambda-Biolab/gha-biorxiv-stats-action/actions/workflows/write-rxiv-stats.yml/badge.svg)](https://github.com/Lambda-Biolab/gha-biorxiv-stats-action/actions/workflows/write-rxiv-stats.yml)
[![CodeQL](https://github.com/Lambda-Biolab/gha-biorxiv-stats-action/actions/workflows/codeql.yml/badge.svg)](https://github.com/Lambda-Biolab/gha-biorxiv-stats-action/actions/workflows/codeql.yml)
[![Lint](https://github.com/Lambda-Biolab/gha-biorxiv-stats-action/actions/workflows/ruff.yml/badge.svg)](https://github.com/Lambda-Biolab/gha-biorxiv-stats-action/actions/workflows/ruff.yml)
[![Tests](https://github.com/Lambda-Biolab/gha-biorxiv-stats-action/actions/workflows/test.yml/badge.svg)](https://github.com/Lambda-Biolab/gha-biorxiv-stats-action/actions/workflows/test.yml)

## What it does

1. Fetches paper records from the bioRxiv / medRxiv `/details/` API for the
   configured date range.
2. Filters rows client-side to the configured `CATEGORIES` set (the API has
   no server-side category filter — see [`docs/categories.md`](docs/categories.md)
   for the full per-server taxonomy).
3. Prunes existing CSVs so historical rows outside the current category set
   are dropped on each run (idempotent).
4. Writes deduplicated `[Date, ISOWeek, DOI, Version, Category, Title, Authors]`
   rows to weekly CSV files at `<OUT_DIR>/<year>/<week>.csv`.
5. Commits the diff via the GitHub API (server-signed) and squash-merges the
   PR back to the base branch.

The included workflow `write-rxiv-stats.yml` runs the action as a 2-leg
matrix over `biorxiv` and `medrxiv`, each with its own category set and
output directory under `data/<server>/`.

## Usage

```yaml
- uses: Lambda-Biolab/gha-biorxiv-stats-action@v0
  with:
    OUT_DIR: "./data/biorxiv"
    DAYS: "7"
    CATEGORIES: "bioinformatics,microbiology"
    SERVER: "biorxiv"
    TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

Multi-server matrix:

```yaml
strategy:
  max-parallel: 1   # serialize to avoid concurrent auto-PR merges
  matrix:
    include:
      - server: biorxiv
        categories: "bioinformatics,microbiology"
      - server: medrxiv
        categories: "infectious diseases,genetic and genomic medicine"
steps:
  - uses: Lambda-Biolab/gha-biorxiv-stats-action@v0
    with:
      OUT_DIR: "./data/${{ matrix.server }}"
      SERVER: ${{ matrix.server }}
      CATEGORIES: ${{ matrix.categories }}
      TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

## Inputs

| Name | Required | Default | Description |
| ---- | -------- | ------- | ----------- |
| `OUT_DIR` | No | `./data/biorxiv` | Directory to write CSV output files. Convention: `./data/<server>` so multiple servers can coexist. |
| `DAYS` | No | `1` | Number of days back to fetch. |
| `CATEGORIES` | No | _(8 Lambda-Biolab categories)_ | Comma-separated categories to keep (case-insensitive). Empty keeps all. Filter applied client-side. Default for bioRxiv: `bioinformatics,bioengineering,microbiology,biochemistry,biophysics,pharmacology and toxicology,genomics,synthetic biology`. See [`docs/categories.md`](docs/categories.md) for medRxiv defaults and per-server taxonomies. |
| `SERVER` | No | `biorxiv` | API server: `biorxiv` or `medrxiv`. Same `/details/` endpoint, different `server` path segment. |
| `TOKEN` | No | `${{ github.token }}` | GitHub token for pushing changes. |

## Output layout

```text
data/
├── biorxiv/
│   └── <YYYY>/<ISOweek>.csv
└── medrxiv/
    └── <YYYY>/<ISOweek>.csv
```

Row shape: `Date, ISOWeek, DOI, Version, Category, Title, Authors` (UTF-8 CSV).

## API

Data sourced from
`https://api.biorxiv.org/details/{server}/{date1}/{date2}/{cursor}/json`.
chemRxiv and psyArXiv use different APIs and are tracked separately
([upstream issue #69](https://github.com/qte77/gha-biorxiv-stats-action/issues/69),
[upstream issue #70](https://github.com/qte77/gha-biorxiv-stats-action/issues/70)).

## License

Apache-2.0 — see [LICENSE](LICENSE).
