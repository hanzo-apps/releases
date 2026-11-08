# Shipwright — Product Changelog & Blog

<!-- hanzo-badges -->
<p align="left">
  <a href="https://platform.hanzo.ai/new?repo=https%3A%2F%2Fgithub.com%2Fhanzo-apps%2Freleases&template=releases"><img src="https://gallery.hanzo.ai/deploy.svg" alt="Deploy to Hanzo" height="32" /></a>
  <a href="https://hanzo.ai/chat?template=releases"><img src="https://gallery.hanzo.ai/badges/edit-in-hanzo.svg" alt="Edit in Hanzo.ai" height="32" /></a>
  <a href="https://gallery.hanzo.ai/templates/releases"><img src="https://gallery.hanzo.ai/badges/live-preview.svg" alt="Live Preview" height="32" /></a>
  <a href="https://gallery.hanzo.ai/llms/releases/LLM.md"><img src="https://gallery.hanzo.ai/badges/llm-md.svg" alt="LLM.md Playbook" height="32" /></a>
</p>
<!-- /hanzo-badges -->

**Ship it, then tell everyone.** A real, buildable Hanzo app: a public product
changelog paired with a longer-form blog. Team members post dated release
entries tagged by type (feature / fix / breaking) alongside narrative posts;
visitors browse a reverse-chronological timeline and filter by type or tag.

Built on the canonical Hanzo stack — forked from
[`hanzo-starter`](https://github.com/hanzo-apps/hanzo-starter), so its provider
stack, Vite config, PKCE auth, and deploy contract are unchanged.

- **UI** — [`@hanzo/gui`](https://www.npmjs.com/package/@hanzo/gui) primitives
  only (no Tailwind, no second kit), under Vite + React 19. A monospace-accented
  terminal aesthetic: a connector line down the left gutter, JetBrains-style
  version chips, muted graphite with a single lime highlight for the newest
  release.
- **Auth** — [`@hanzo/iam`](https://www.npmjs.com/package/@hanzo/iam), OAuth2
  **PKCE** against [hanzo.id](https://hanzo.id). No local passwords.
- **Data** — [`@hanzo/base`](https://www.npmjs.com/package/@hanzo/base), the
  IAM-native, org-scoped data plane. Entries, tags, and their join are real Base
  collections provisioned from [`schema.sql`](./schema.sql).

## Views

| View | What it does |
| --- | --- |
| **Changelog feed** | Reverse-chronological timeline of dated entries with type pills; filter by type (feature/fix/breaking/post) or tag. The newest published release is highlighted lime. |
| **Post detail** | The full entry body (blank-line paragraphs) plus a "related" list of entries sharing a tag or type. |
| **Compose** | Signed-in editor: title, version, type, comma-separated tags, body — publish or save as a draft. |

The signed-out landing is itself a public changelog page (a preview timeline)
so visitors see the format before signing in.

## Stack (pinned)

| Package | Version |
| --- | --- |
| `react` / `react-dom` | `^19.2.4` |
| `@hanzo/gui` + `@hanzogui/config` | `7.3.0` |
| `@hanzo/iam` | `^0.13.1` |
| `@hanzo/base` | `^0.2.1` |
| `vite` | `^6` (`@vitejs/plugin-react`) |
| `typescript` | `5.9.3` |
| `react-native-web` | `^0.21.0` |

## Run it

```sh
npm install
npm run dev        # http://localhost:5173
npm run build      # tsc --noEmit && vite build  ->  dist/
npm run preview    # serve the production build (SPA fallback on)
```

Out of the box it runs against **live** Hanzo (hanzo.id + api.hanzo.ai) — no
config needed to see the sign-in flow and the landing. Copy `.env.example` to
`.env` to point at a different environment.

## Environment contract

Only `VITE_`-prefixed vars reach the browser (this is a static SPA — no server).
Defaults in parentheses.

| Var | Purpose |
| --- | --- |
| `VITE_HANZO_IAM_URL` (`https://hanzo.id`) | OIDC issuer. |
| `VITE_IAM_CLIENT_ID` (`hanzo-app`) | IAM application (`<org>-<app>`). A dedicated `hanzo-changelog-ship` client is provisioned at deploy; its redirect-URI list must allow this origin's `/auth/callback`. |
| `VITE_HANZO_REDIRECT_URI` (`${origin}/auth/callback`) | PKCE redirect. |
| `VITE_HANZO_BASE_URL` (`https://api.hanzo.ai`) | Browser-reachable Hanzo Base data plane. Deploy injects the provisioned URL. |

## How auth works — ambient IAM

`login()` starts an OAuth2 **PKCE S256** redirect to hanzo.id; hanzo.id returns
to `/auth/callback`, where `handleCallback()` exchanges the code for tokens
(stored in `localStorage`, refresh-aware via `offline_access`). There is **no
server token** — the SPA authenticates the user in the browser and carries the
resulting IAM JWT to Base. The client id is read from `VITE_IAM_CLIENT_ID`
(fallback `hanzo-app`).

## How data works — Base from `schema.sql`

[`schema.sql`](./schema.sql) is the app's `databaseSchema` (SQL DDL). On publish,
Hanzo Cloud translates each `CREATE TABLE` into a Hanzo Base collection
(`provisionBaseFromDDL`, additive + idempotent). Base manages
`id`/`created`/`updated`/`owner`/`org`, stamps `owner`+`org` from the verified
IAM principal, and scopes every row to the caller's org via the access rule
`@request.auth.org_id = org`. Three collections back Shipwright:

- **`entries`** — `title`, `body`, `kind`, `version`, `published_at`, `draft`.
- **`tags`** — `label`, `color` (deduped by label; created on demand in Compose).
- **`entry_tags`** — `entry_id`, `tag_id` (the many-to-many join).

At runtime the views read/write these through `@hanzo/base/react`
(`useQuery`/`useMutation`) and the imperative `useBase()` client, all carrying
the IAM token. Keep `schema.sql` in lockstep with what the app reads/writes.

## Deploy — Hanzo Cloud

[`hanzo.yml`](./hanzo.yml) declares a static build (`npm run build` → `dist/`,
served at `<slug>.hanzo.app`) plus the Base schema to provision and the env to
inject. Do **not** build a container image locally — Hanzo Cloud owns builds and
deploys. CI here only proves the template compiles green.

## Layout

```
src/
  main.tsx          entry
  providers.tsx     GuiProvider -> IamProvider -> BaseProvider(client=IAM-token)
  app.tsx           route (/auth/callback) + auth gate
  gui.config.ts     createGui(defaultConfig from @hanzogui/config/v5)
  iam.config.ts     IAM PKCE config
  env.ts            the VITE_ env contract, one place
  theme.ts          the terminal palette + mono stack + per-kind pill metadata
  ui.tsx            shared @hanzo/gui compositions (chips, pills, timeline row…)
  lib/base.ts       BaseClient carrying the IAM bearer token
  lib/entries.ts    Entry/Tag/EntryTag types + date helpers + landing sample
  auth/callback.tsx PKCE return leg
  views/            signed-out · home · feed · detail · compose
schema.sql          databaseSchema -> Base collections on publish
hanzo.yml           Hanzo Cloud build/deploy manifest
```
