# libre-fc

A privacy-respecting football app: fixtures, results, tables, match detail. No news, no
trackers, no ads. Starts with the Eredivisie and grows to more competitions later.

The domain is deliberately small. The design system is the point.

## Stack

Laravel serving a JSON API, a React + TypeScript SPA, PostgreSQL. Pest on the backend,
Vitest + React Testing Library on the frontend.

Not Inertia — API Resources on one side, React Router and TanStack Query on the other.
The contract friction is deliberate; it is where the learning is.

No component libraries. Primitives are built by hand against a CSS custom property token
file.

## Status

Scaffolded, no features. A stock Laravel 13 skeleton on PHP 8.5, plus a container stack.
Nothing in `## Stack` above is wired up yet: no React, no TypeScript, no Vitest, no Pest,
and the app still runs on the skeleton's SQLite file rather than the Postgres service.

## Running it

`compose up` starts three services: `php` on :8000 (`artisan serve`), `db` on :5433, and
`node` running `npm run dev`. Images are fully qualified (`docker.io/library/…`) so Podman
resolves them without prompting.

The `node` service publishes 5174, but nothing is listening there — the Vite dev server is
not reachable from the host yet. See *Scaffold deltas* in the architecture doc.

`Containerfile` builds PHP 8.5 with `pdo_pgsql` and Composer. Node 24 and Postgres 17 run
as stock images — no build. Host port 5433 keeps a local Postgres out of the way.

First run needs `composer install` in `php` and `npm install` in `node`; `composer setup`
does both plus key generation and migrations, but only where both runtimes exist.

## Docs

[`docs/ARCHITECTURE.md`](docs/ARCHITECTURE.md) — the decisions that are expensive to
reverse: topology, data model, placement rules, styling, and what has been deliberately
deferred.
