# CLAUDE.md

Guidance for AI assistants working in this repository. See `README.md` for user-facing setup docs and `CONTRIBUTING.md` for the contribution rules (including the project's **AI-assisted coding policy** — read it before opening a PR).

## What this project is

PokéAPI (`pokeapi.co`) — a read-only RESTful API for Pokémon data, served by Django + Django REST Framework. All data lives as CSV files in `data/v2/csv/` and is loaded into the database by a build script; there is no write path in the API. A Hasura GraphQL layer sits on top of the same Postgres database.

- Python `>=3.10,<3.15`; local dev pins `3.14` (`.python-version`). CI tests 3.10 → 3.14.
- Package/venv manager: **uv** (`uv.lock` is committed and CI installs with `--locked`).
- Version is duplicated in `pyproject.toml` (`version`) and `config/settings.py` (`SPECTACULAR_SETTINGS["VERSION"]`) — keep them in sync.

## Repository layout

| Path | Purpose |
| --- | --- |
| `pokemon_v2/models.py` | ~210 Django models: abstract `Has*` mixins first, then concrete models. |
| `pokemon_v2/serializers.py` | ~198 serializers (~230k chars). `*SummarySerializer` (name + url, used in lists/references) come first, then `*DetailSerializer`. |
| `pokemon_v2/api.py` | DRF viewsets (`*Resource`), plus `PokemonEncounterView` and `PokeapiMetaViewset`. |
| `pokemon_v2/urls.py` | Router registration for every resource under `/api/v2/`. |
| `pokemon_v2/tests.py` | API integration tests: `APIData` fixture factory + `APITests`. |
| `pokemon_v2/test_models.py` | Model tests and CSV data validation (identifier slug format). |
| `pokemon_v2/migrations/` | Django migrations (latest: `0029_pokemonformtrigger_pokemonformcondition`). |
| `data/v2/csv/` | ~180 CSV files — the canonical data source. |
| `data/v2/build.py` | Loader that wipes and repopulates every table from the CSVs. |
| `data/v2/sprites/`, `data/v2/cries/` | Git submodules (media assets). |
| `config/` | Django settings: `settings.py` (base/prod), `local.py` (SQLite + dummy cache + DEBUG), `docker-compose.py`, `docker.py`. |
| `graphql/v1beta/`, `graphql/v1beta2/` | Hasura metadata + `.gql` examples. |
| `Resources/` | Dockerfile, nginx, k8s kustomize overlays, kind config, helper scripts. |
| `openapi.yml` | Generated OpenAPI 3.1 schema — do not hand-edit. |

Settings are always selected explicitly, never by default: local commands pass `--settings=config.local`, container commands pass `--settings=config.docker-compose`. `manage.py` defaults to `config.settings` (production Postgres), so run commands through the Makefile rather than bare `manage.py`.

## Common commands

Use the Makefile (`make help` lists everything). Every target shells out through `uv run`.

```sh
make install          # uv sync --locked --all-extras --dev
make install-base     # runtime-only deps (no dev group)
make setup            # migrate against SQLite (config.local)
make build-db         # wipe + repopulate all tables from data/v2/csv
make serve            # runserver on :8000
make test             # Django test runner (55 tests, ~2s)
make format           # black (excludes '.+/scripts/.+')
make format-check     # black --check — CI gate
make make-migrations  # makemigrations
make migrate          # migrate
make shell            # Django shell
make openapi-generate # regenerate openapi.yml via drf-spectacular
make wipe-sqlite-db   # rm db.sqlite3
```

Docker/Compose (Postgres + Redis + nginx + Hasura, API on port 80): `make docker-setup` (= `docker-up` + `docker-migrate` + `docker-build-db`), then `make docker-test`, `make docker-shell`, `make docker-down`. Kubernetes: `make kustomize-apply`, `make k8s-migrate`, `make k8s-build-db`. Hasura: `make hasura-apply` / `make hasura-export` (needs `HASURA_GRAPHQL_ADMIN_SECRET` and the `hasura` CLI).

`make build-db` takes several minutes and requires the submodules for sprite/cry URLs. If `data/v2/sprites` and `data/v2/cries` are empty (they are unpopulated by default in a fresh clone/session), `file_path_or_none()` in `data/v2/build.py` returns `None` and sprite/cry fields come back null. Populate with `git submodule init && git submodule update`.

`make test` uses SQLite and does **not** need a built database — tests create their own fixtures.

## Conventions

**Data changes (CSV).** `data/v2/csv/` is the source of truth and is kept in sync with the veekun/pokedex project (`make sync-from-veekun` / `make sync-to-veekun`, expecting a `../pokedex` checkout). Rules:
- The `identifier` column is what appears in API URLs and must match `^[a-z0-9-]+$` — lowercase ASCII, digits, hyphens only. `test_models.py::CSVResourceNameValidationTestCase` enforces this across every CSV; the `Database / csv` GitHub workflow additionally runs `csvlint`.
- IDs are explicit in the CSVs; `build.py` resets the DB sequences per table, so don't rely on auto-increment.
- Files use `,` delimiters with a header row.

**Adding or changing a resource / field.** The canonical change touches, in order:
1. `data/v2/csv/*.csv` — the data.
2. `data/v2/build.py` — a `_build_*()` function with a nested `csv_record_to_objects(info)` generator, wired through `build_generic((Model,), "file.csv", csv_record_to_objects)`, and registered in `build_all()`. Column access is positional (`info[0]`, `info[1]`, …), so CSV column order and the builder must stay in lockstep.
3. `pokemon_v2/models.py` — reuse the abstract `Has*` mixins for foreign keys (they set `related_name="%(class)s"`, which serializers rely on as the `source=`).
4. `make make-migrations` — commit the generated migration.
5. `pokemon_v2/serializers.py` — a `*SummarySerializer` (declared in the top block) and/or fields on the `*DetailSerializer`. Computed fields use `SerializerMethodField` annotated with `@extend_schema_field({...})` so the OpenAPI schema stays accurate.
6. `pokemon_v2/api.py` + `pokemon_v2/urls.py` — for a brand-new endpoint: subclass `PokeapiCommonViewset`, set `queryset`, `serializer_class`, `list_serializer_class`, decorate with `@extend_schema(description=..., tags=[...], summary=...)` and `@extend_schema_view(list=extend_schema(summary=...))`, then `router.register(r"kebab-case-name", XResource)`.
7. `pokemon_v2/tests.py` — a `setup_*_data` classmethod on `APIData` plus a `test_*_api` method asserting `id`, `name`, nested `name`/`url` pairs (URLs built as `"{}{}/resource/{}/".format(TEST_HOST, API_V2, pk)`).
8. New tables also need Hasura metadata under `graphql/v1beta2/metadata/databases/default/tables/` (one YAML per table, listed in `tables.yaml`, plus relationship entries on the related tables). Historically done as a follow-up PR — see `332f2b9`.
9. `make openapi-generate` when the public API shape changes.

**API behavior.** `PokeapiCommonViewset` composes two mixins: `ListOrDetailSerialRelation` (swaps in `list_serializer_class` for list actions) and `NameOrIdRetrieval` (lookup by numeric pk or case-insensitive `name`, plus a `q=` `name__icontains` filter that the schema documents as local-only — it is not reachable through the production pokeapi.co deployment). IDs above `2147483647` and lookups that match neither the id nor name pattern raise `Http404`. Everything is `ReadOnlyModelViewSet`; CORS allows GET only, on `^/api/.*$`. Pagination is `LimitOffsetPagination` with `PAGE_SIZE = 20`.

**Style.** `black` with default settings, excluding `.+/scripts/.+`; run `make format` before committing. Existing files use `from .models import *` / `from pokemon_v2.api import *` star imports and banner comments (`###  SECTION  ###`) — match the surrounding style rather than modernizing. `# pylint: disable=...` pragmas at the top of files are intentional.

## CI

- **CircleCI** (`.circleci/config.yml`): `static-checks` (`make install`, `make format-check`, `make openapi-generate`), `run-tests` matrix over Python 3.10–3.14 (`make test`), `build` (`make install-base`, `make setup`, `make build-db`), and `deploy` (pushes regenerated data to `PokeAPI/api-data` — `master`/`staging` only). A weekly Monday cron runs the same pipeline on `staging`.
- **GitHub Actions**: `database.yml` (csvlint, OpenAPI generation, SQLite end-to-end across 3.10–3.14, Postgres/Compose dump→restore round-trip); `docker-k8s.yml` (multi-arch image build, yamllint + kind cluster apply); `docker-build-and-push.yml` and `release.yml` (publish images and attach `db.sqlite3` / `pokeapi.pgdump` to releases) on `master`/tags.

The bar for a PR: `make format-check` clean, `make test` green, new tests for new behavior.

## Gotchas

- Never hand-edit `openapi.yml` or `uv.lock`; regenerate them (`make openapi-generate`, `uv` commands).
- `db.sqlite3` is gitignored (`db.*`) — don't commit databases or dumps.
- `data/v2/build.py` executes `connection.cursor()` at import time, so importing it outside a Django context fails; run it via `make build-db` or the Django shell.
- `build.py` has both `DATA_LOCATION` (cwd-relative `data/v2/csv/`) and `DATA_LOCATION2` (`__file__`-relative). `load_data()` uses the cwd-relative one, so build commands must run from the repo root.
- `PokeapiMetaViewset` (`/api/v2/meta/`) shells out to `git`; that's why the Docker image installs `git`.
- The Hasura tables are prefixed `pokemon_v2_*` in Postgres but exposed under `custom_name` (e.g. `ability`) in GraphQL.
- `default.nix` still references a `requirements.txt` that no longer exists (the project moved to uv) — it's stale.
