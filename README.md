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

Documentation only. The application is not scaffolded yet.

## Docs

[`docs/ARCHITECTURE.md`](docs/ARCHITECTURE.md) — the decisions that are expensive to
reverse: topology, data model, placement rules, styling, and what has been deliberately
deferred.
