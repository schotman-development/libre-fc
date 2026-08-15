# Architecture

Decisions that are expensive to reverse, and the tree they produce. Everything not
recorded here is a `git mv` away from being different, and doesn't need a document.

## Topology

One repository. Laravel at the root, React under `resources/js`, built by Vite via
`laravel-vite-plugin`. A single Blade view serves the SPA shell; everything else is JSON.

Same origin, so there is no CORS layer, no `VITE_API_URL`, and cookie-based auth would
work without configuration if accounts are ever added. One build, one deploy.

This is not Inertia. The frontend talks to the API through React Router and TanStack
Query against JSON API Resources. Monolith-vs-split-repo and Inertia-vs-JSON-API are
independent axes; this project takes the monolith on one and the explicit API contract
on the other. The contract friction is the point — it is where the learning is.

## Local environment

Containers, three services: `php` (built from `Containerfile`), `node` and `db` (stock
images). Nothing is installed on the host.

**Vite runs in its own service, not inside the PHP container.** One image carrying both
runtimes rebuilds whenever either moves, and PHP 8.5 and Node 24 have nothing to say to
each other. Two containers, two lifecycles, one bind mount.

**`artisan serve`, not php-fpm behind nginx.** One process, one port, no config files. A
production-shaped web tier is a deploy concern and there is no deploy yet.

**Image names are fully qualified** — `docker.io/library/postgres:17-alpine`, not
`postgres:17-alpine`. Podman has no implicit registry and prompts for a choice without the
prefix; Docker ignores it. The prefix costs nothing and makes the file portable between
both.

**The whole repo is bind-mounted at `/app`.** Which means `vendor/` and `node_modules/`
are written into the working tree rather than into anonymous volumes — both gitignored,
and a host editor can read them for completion and static analysis without a second
install.

**Postgres publishes on host 5433.** 5432 is usually already taken by a host Postgres, and
a port collision at `compose up` is a worse first-run experience than a non-default number.

## Tree

The target layout. What exists today is the stock skeleton plus the container files —
see *Scaffold deltas*. Unmarked directories are Laravel's default install.

```
libre-fc/
├── .github/workflows/ci.yml          # Pest, Vitest, tsc --noEmit, Pint, ESLint
├── app/
│   ├── Actions/
│   │   └── CalculateStandings.php    # the only real logic in the app
│   ├── Console/Commands/
│   │   └── ImportEredivisie.php
│   ├── Http/
│   │   ├── Controllers/              # flat, no Api/ subfolder
│   │   └── Resources/
│   ├── Models/
│   └── Providers/
├── bootstrap/
├── config/
├── database/
│   ├── factories/
│   ├── migrations/
│   └── seeders/
├── docs/
│   └── ARCHITECTURE.md
├── public/
├── resources/
│   ├── css/
│   │   ├── tokens.css                # plain .css, NOT a module
│   │   └── app.css                   # reset + base elements, @imports tokens
│   ├── views/
│   │   └── app.blade.php             # SPA shell
│   └── js/
│       ├── main.tsx                  # imports '../css/app.css'
│       ├── router.tsx
│       ├── vite-env.d.ts
│       ├── lib/
│       │   ├── http.ts               # fetch wrapper, base URL, error shape
│       │   └── queryClient.ts
│       ├── types/
│       │   └── api.ts                # mirrors the API Resources
│       ├── ui/                       # design system. domain-free. folder per component.
│       │   ├── Button/
│       │   │   ├── Button.tsx
│       │   │   ├── Button.module.css
│       │   │   ├── Button.test.tsx
│       │   │   └── index.ts
│       │   └── Table/
│       ├── features/                 # flat inside each feature
│       │   ├── fixtures/
│       │   │   ├── FixtureList.tsx
│       │   │   ├── FixtureList.module.css
│       │   │   └── useFixtures.ts
│       │   ├── table/
│       │   └── match/
│       └── test/
│           └── setup.ts
├── routes/
│   ├── api.php
│   ├── console.php
│   └── web.php                       # catch-all, excludes api/*
├── storage/
├── tests/
│   ├── Feature/                      # flat, they're all API tests
│   └── Unit/
├── Containerfile                     # php 8.5 + pdo_pgsql + composer
├── compose.yaml                      # php :8000, db :5433, node (see deltas)
├── composer.json
├── package.json
├── eslint.config.js
├── tsconfig.json
├── vite.config.ts
├── phpunit.xml
└── pint.json
```

## Placement rules

**`ui/` may not import `features/` or `types/api.ts`.** The dependency arrow points one
way only. Enforced by ESLint `no-restricted-imports`, scoped to `resources/js/ui/**`.
Without the rule the boundary erodes invisibly — one convenient `fixture` prop on a Card
and the design system can no longer be extracted.

**Controllers stay flat.** Every controller here is an API controller; the only web route
is a closure returning the Blade shell. Add `Api/` the day a non-API controller exists.

**`app/Actions/` holds one class, and that is correct.** No `Services/` sibling — two
identically-shaped folders turn every future placement into an argument.

**Query hooks colocate inside their feature.** The usual reason to centralise them is
invalidation, and this app has no mutations. Nothing ever invalidates anything, so there
is no cross-feature key coordination to solve and no query-key factory.

**Shared domain components: rule of three.** A component needing a domain type cannot live
in `ui/`. It lives in the feature that needs it first, and is promoted to a top-level
`components/` when a *third* consumer appears. Two consumers is a coincidence.

**`ui/` is folder-per-component, `features/` is flat.** Forced by CSS Modules: every
primitive has three files immediately, so a flat `ui/` would be unusable. Feature folders
already provide the grouping.

## Path alias

`@/` maps to `resources/js/`, declared in **both** `tsconfig.json` (`compilerOptions.paths`)
and `vite.config.ts` (`resolve.alias`) — TypeScript resolves for the editor, Vite for the
bundle, and neither reads the other's config.

This is what makes the tree above cheap. Moving a folder rewrites no imports, which is why
the layout can start minimal instead of pre-building the general case.

## Data model

**One `matches` table.** Nullable `home_score` / `away_score`, plus `status`. A fixture is
a match with null scores; a result is the same row after ingest. The null-score branch is
also a UI state — the fixture card and result card are one component, not two.

**`status` is `->string()` with a PHP backed enum cast, not `->enum()`.** Laravel's `enum()`
on Postgres produces a varchar with a CHECK constraint; adding `abandoned` later would need
a migration to drop and recreate it. A string column makes new statuses a code-only change.

**Standings are stored, not derived.** Which means the table stores `position`. Tiebreak
rules are competition-specific — Eredivisie orders by points, goal difference, then goals
scored, while La Liga and Serie A break ties on head-to-head — so the ordering logic runs
once at ingest rather than in every query or, worse, in a frontend sort. The API Resource
and the `<Table>` primitive both just render rows in the order they arrive.

**Recompute is always total, never incremental.** 18 teams, 306 matches, one aggregate
query. Incremental updates require reversing a match's old contribution before applying a
correction, and any bug there is silent and permanent. A total recompute is idempotent, so
"did the import run twice?" stops being a question. Only `finished` matches count — omit
that filter and every unplayed match reads as a 0-0 draw.

**`external_id` on `teams` and `matches` from the first migration.** Nullable, unique. Data
comes from an external source; without a stable key, re-importing or changing provider means
matching on names, and names are not stable (`PSV` / `PSV Eindhoven` / `P.S.V.`). This is
the single most painful thing to retrofit onto a populated database.

**`competition` and `season` columns on `matches` and `standings`.** Plain strings, no
`competitions` or `seasons` tables yet. More competitions are planned, and this is the
unique key on standings — `unique(competition, season, team_id)` — which is expensive to
change once populated. Promoting a string to a foreign key later is mechanical because the
values are already distinct.

## Styling

CSS Modules with a token file. `tokens.css` holds `:root` custom properties; component
styles live in colocated `*.module.css`.

`tokens.css` is deliberately not a module. Custom properties are global by definition —
scoping them would defeat their purpose.

Custom properties are runtime and inherit through the cascade, unlike Sass variables, so a
module writes `var(--color-pitch)` with no import at all. That is what makes theming free:
`[data-theme="dark"] { --color-pitch: … }` re-themes every component without touching a
single component file. The token file is the design system's real API.

Class names are camelCase (`.homeScore`, not `.home-score`) so they read as `styles.homeScore`
without needing `css.modules.localsConvention`. Inside a module, names are already scoped —
`.root`, `.label`, `.score` are correct, and BEM buys nothing.

`tsconfig.json` needs `"types": ["vite/client"]`, or every `*.module.css` import is an
unresolved-module error under `tsc --noEmit` and CI is red from the first component.

One Vite input, not two: `main.tsx` imports `app.css`. `laravel-vite-plugin` reads the build
manifest and emits a real `<link rel="stylesheet">` for CSS reachable from the JS entry, so
a separate entry point buys nothing and is one more thing to keep in sync.

## Testing

Backend tests in `tests/`, flat under `Feature/`, because Pest expects them there. Frontend
tests colocate as `Component.test.tsx` — no mirror tree to keep in sync.

`CalculateStandings` is the only genuine logic in the codebase; everything else is Eloquent
and rendering. The case worth testing is not the 3-1-0 arithmetic but two teams level on
points where goal difference decides, with one postponed match in the set that must not
count.

## Scaffold deltas

The skeleton shipped defaults that contradict decisions above. Unlike *Deferred*, these are
not choices — they are drift, and this section should shrink to nothing.

**The Vite dev server is unreachable from the host.** `compose.yaml` publishes `5174:5174`,
and nothing binds it. The skeleton's `vite.config.js` sets no `server.port` and no
`server.host`, so Vite takes its default 5173 and listens on loopback — which inside a
container is the container's own loopback, not the host's. Two separate fixes: a port both
sides agree on, and `0.0.0.0` so the bind is on an interface the forward can reach, the same
reason the `php` service already passes `artisan serve --host=0.0.0.0`. Asset URLs are a
third piece — `laravel-vite-plugin` writes them into the page and the browser resolves them
in its own address space, so `server.hmr` / `server.origin` have to match what the host
publishes. Not yet verified against a running dev server.

**Postgres is running and unused.** `.env` and `phpunit.xml` both say `DB_CONNECTION=sqlite`,
and `database/database.sqlite` exists. The `db` service and `pdo_pgsql` are in place, so this
is env configuration only — but the data model above leans on Postgres, and every migration
written against SQLite is a migration whose column types were never really tested. Switch
before the first migration, not after.

**Tailwind is installed.** `resources/css/app.css` is `@import 'tailwindcss'`, and
`@tailwindcss/vite` is in the Vite config. Utility classes and CSS Modules can coexist, but
`tokens.css` is described above as the design system's real API, and a second styling system
means every component has two right answers. Remove Tailwind, or supersede the *Styling*
section deliberately — not by leaving both in place.

**Pest is not installed.** `composer.json` has PHPUnit 12; `tests/Feature/ExampleTest.php`
is a PHPUnit class. The `pestphp/pest-plugin` entry under `allow-plugins` is skeleton
boilerplate, not an install.

**Entry points are the skeleton's.** `vite.config.js` not `.ts`, `resources/js/app.js` not
`main.tsx`, `welcome.blade.php` not `app.blade.php`, no `routes/api.php`. No `tsconfig.json`,
`eslint.config.js`, `pint.json`, or CI workflow yet.

**Fonts are self-hosted, which is fine.** `vite.config.js` uses `bunny('Instrument Sans')`
from `laravel-vite-plugin/fonts`. The provider name is a source, not a runtime dependency —
the plugin resolves the files at build time and emits them as Vite assets, so no request
leaves the browser for a third party. No conflict with "no trackers"; noted because the
config reads like a CDN link and someone will eventually flag it.

## Deferred

**Sentry.** Two dependencies, and a real tension with "privacy-respecting" — Sentry receives
request context by default. If added: self-host or configure scrubbing, and be able to say
why. Add after the first feature works end to end.

**Playwright.** One test on one critical path is the right amount, but the path has to exist
first. Add when fixtures → match detail works.

**Queues / Horizon.** Ingest and recompute run synchronously inside the artisan command, in
one transaction. Add a queue when 306 matches is slow, which is never.

**`resources/views/app.blade.php` stays a Blade view, not a static `index.html`.** Costs
nothing and preserves a seam: fixtures, tables and results are SEO-relevant content, so
per-route meta tags or a server-rendered initial payload may be wanted later. A static entry
point would make that a restructure.

## Known ceilings

`vite/client` types CSS Modules as `Record<string, string>`, so `styles.buton` is `undefined`
at runtime rather than a type error. Accepted tradeoff for zero config; upgrade to
`typescript-plugin-css-modules` only if it actually bites.

The `web.php` catch-all must exclude `api/*` via a `where` constraint, or the SPA shell
swallows API 404s and returns HTML to a `fetch()` expecting JSON.
