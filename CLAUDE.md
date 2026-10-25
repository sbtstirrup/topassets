# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

TopSupport Assets: a single-page, front-end-only prototype of an internal IT asset management tool (laptops, phones, licences, network gear; assignment, warranties, audit trail, user admin). There is no backend, build step, package manager, linter or test suite. All data is seeded in-page and lives in component state, so it resets on reload; the only persistence is the remembered sign-in in `localStorage` (`ts-assets-auth`).

## Files

- `TopSupport Assets.dc.html` — the whole app: template markup plus the logic script. This is the only file you normally edit.
- `support.js` — the generated "dc-runtime" that parses and renders `.dc.html` files. **Do not edit** (header: generated from `dc-runtime/src/*.ts`, which is not in this repo). It loads React 18 UMD from unpkg at runtime, so viewing the page needs network access.

## Running

Open the `.dc.html` file in a browser, or serve the folder (e.g. `python -m http.server`) and open `TopSupport%20Assets.dc.html`. Serving over HTTP is preferable: the runtime re-fetches `location.href` to pick up the raw template, which silently no-ops under `file://`.

## How a `.dc.html` file works

The file has two parts that the runtime (`support.js`) binds together:

1. **Template** — everything inside `<x-dc>…</x-dc>`. Plain HTML with inline styles (no CSS classes or stylesheet beyond the small reset in `<helmet>`, which is hoisted into `<head>`). Dynamic parts use:
   - `{{ expr }}` interpolation in text and attribute values (including inside `style="…"`).
   - `<sc-if value="{{ cond }}">` and `<sc-for list="{{ items }}" as="x">` for control flow.
   - Event attributes such as `onClick="{{ handler }}"`, `onChange`, `onKeyDown` bound to functions.
   - `hint-placeholder-val` / `hint-placeholder-count` are only for the streaming/placeholder preview; keep them on new `sc-if` / `sc-for` elements.

   Expressions are deliberately minimal (see `resolve` in `support.js`): dotted/bracket paths, string/number/boolean literals, `!`, and `===`/`!==`/`==`/`!=`. No function calls, ternaries, arithmetic or `&&`/`||`. Any derived value, label, colour or click handler must be computed in JS and exposed by name.

2. **Logic** — `<script type="text/x-dc" data-dc-script data-props="…">` at the bottom. Defines `class Component extends DCLogic` (a React-class-like wrapper with `state`, `setState`, `componentDidMount`, `props`). Its `renderVals()` returns the flat object the template renders against, merged over props. `data-props` (HTML-entity-encoded JSON) declares the editable props: `requireSignIn`, `showWarrantyBanner`, `startPage`.

So adding a UI feature almost always means touching both halves: add the value or handler to `renderVals()` (or one of its helpers, `userVals()` / `authVals()`, which are spread into it), then reference it by name in the template.

## App structure

- Navigation is a single `state.page` string (`dash`, `assets`, `detail`, `form`, `expiring`, `audit`, `reports`, `users`, `userForm`, `userAudit`, `changes`). Each page is a `<sc-if>` block in the template, marked by an HTML comment (`<!-- DASHBOARD -->`, `<!-- INVENTORY -->`, …); `renderVals` exposes the matching `pgXxx` booleans. `this.go(page, extra)` returns a click handler that switches page.
- Sign-in is a mocked flow (Technician: email/password; Admin: plus a verification code) gated by `signedOut`. Roles are `Admin`, `Technician` and `Read only`, and drive `isAdmin`, `canEdit` and the admin nav.
- Seed data constants sit at the top of the script: `SEED` (assets), `SEED_LOG` (asset audit events), `SEED_USERS`, `SEED_CHANGES`, `PEOPLE` (assignees), `BY` (initials → name). "Today" is fixed as `TODAY = 2026-10-25`, and new log/change entries use a hard-coded timestamp, so warranty/expiry maths is deterministic.
- Mutations go through `update(tag, patch)` for assets, `logEvent()` for the asset audit log, `addChange()` for field-level change history, and `toast()` for notifications. Keep this pattern so audit and change views stay consistent.
- Brand palette used across inline styles: primary green `#1b8463` (dark `#0d5c43` / `#0d4a37`, tint `#f2f8f5` / `#d5e8de`), page background `#e9eaec`; fonts Montserrat (headings) and Open Sans (body).
