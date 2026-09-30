---
description: "Level 4 of Vue.js: SSR by hand and with Nuxt, production builds, component libraries, architecture, migration, i18n, deployment, observability and a capstone."
---

# Level 4 · Master <span class="level-badge">Production</span>

Goal: take a Vue app from "works on my machine" to something you can run for real users —
rendered on the server where it matters, built and deployed predictably, organised so a
team can grow it, available in more than one language, and observable when it breaks.

Level 3 explained how Vue works inside. Level 4 is about everything *around* the
components: the server, the build, the hosting, the codebase's shape over time, and the
feedback loop from production back to you.

## Modules

1. [Server-Side Rendering from Scratch](01-ssr-from-scratch.md) — app factories, `renderToString`, state transfer, preload links, status codes, hydration mismatches
2. [Nuxt Overview](02-nuxt-overview.md) — file routes, server API routes, `useFetch` and the payload format, `useState`, SEO meta, route rules
3. [Vite & Production Builds](03-vite-production-builds.md) — env files and modes, what reaches the client, base paths, source maps, chunking
4. [Component Libraries & Design Systems](04-component-libraries.md) — component APIs, library mode, typed exports, consuming a library
5. [Large-App Architecture](05-large-app-architecture.md) — features over file types, public APIs between features, enforced boundaries
6. [Migrating from Vue 2 & Staying Current](06-migration-staying-current.md) — seven observed breaking changes, an incremental plan, evaluating Vapor Mode
7. [Internationalization](07-i18n.md) — vue-i18n, correct plurals with `Intl.PluralRules`, formats, lazy locales, RTL, escaping
8. [Deployment](08-deployment.md) — history fallback, cache headers, stale chunks after releases, runtime config, containers, CI
9. [Observability & Production Readiness](09-production-readiness.md) — a tested error reporter, source maps, web vitals, a launch checklist
10. [Capstone — A Production Vue App](10-capstone.md) — a Nuxt meetup app with a validated API, accessible RSVP form, error reporting, unit and smoke tests

## Before you start

You should be comfortable with everything in Level 3, especially the reactivity model
(Level 3 · 01) and testing (Level 3 · 07). Several lessons use Node.js on the command line
directly — writing a small HTTP server, running `node --test` — so be ready to leave the
Vite dev server behind.

Lessons 01, 02 and 10 involve a server. You'll need Node.js locally; nothing requires a
cloud account.

## How this level was verified

Builds and servers were run locally: Vite 8.3.1 for the SPA builds, Nuxt 4.5.2 (with Vue
3.5.43) for the Nuxt lessons and the capstone, vue-i18n 11.4.12 for internationalization,
and Vitest 4.1.11 with jsdom for the error-reporting tests. HTTP behaviour was checked
with `curl`, and the SSR and capstone pages were used in a real browser. Vue 3.6 compiler
output in Lesson 06 came from the `3.6.0-rc.9` release candidate.

Some things couldn't be run on the machine used to write this level: no Docker or nginx
was available, and nothing was deployed to a hosting provider or sent to a hosted error
service. Those configurations are clearly marked as unexecuted examples — check them
against your provider's current documentation. Where a run failed first (a brittle smoke
test, a prerendered page ignoring runtime config, a test that didn't mount what we
thought), the lesson shows the failure and the fix.
