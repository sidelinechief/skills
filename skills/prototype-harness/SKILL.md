---
name: prototype-harness
description: Build an interactive, in-app prototype of a redesigned or new page, running in the real app against real production data. Use when asked to prototype, workshop, or mock up a redesign of an existing page (product page, search, checkout, profile, listing flow…) or a new page or flow built from the app's existing patterns and design system, compare design iterations side by side, or preview a page against real production data with toggles for viewer/user type, feature flags, item/entity state, and a desktop↔mobile switch. Produces a self-contained `/prototype/*` route with a control panel, a captured analytics event stream, a stubbed backend, and a decision log for the engineer who builds it — no live services required.
---

# Prototype Harness

A pattern for standing up a throwaway-but-faithful prototype **inside the app**, either a redesign of an existing page or a new page built from the app's existing components and design system, so design decisions can be made against real production data and real analytics without touching production code or needing a live backend.

Written for a React app on the Next.js App Router. The pattern ports to other React setups; adjust the server-component parts.

**Where the ground truth is:**

- If the repo already has a prototype built with this harness (look under `app/prototype/` or search for `usePrototypeBackend`), read it and copy its structure. That code is the ground truth for this repo; this skill explains the pattern.
- If not, you're building the first one. **[references/building-blocks.md](references/building-blocks.md)** is the spec, with sketches of the parts that are easy to get wrong. Start with "Map the app" below.

The examples throughout use a marketplace product page (an *item* with a seller, a price, and Add to Cart / Make Offer actions). Substitute your own entity.

## When to use this

- "Let's prototype / redesign / workshop the `<page>` page."
- "Show me a few design iterations of `<page>` and let me toggle between them."
- "Preview this against a real production item / listing / order."
- "I want to see the seller view vs. the buyer view / logged-out view."
- "How does this look on mobile vs. desktop?"
- "Design a new `<page>` page or flow that fits the rest of the app."

If the ask is a production feature change (not an exploratory redesign or new design), this is **not** the right tool — build it in the real page.

## Map the app (first prototype only)

Before writing harness code, find the app's equivalents of each of these and note them at the top of the mock module. Every later step depends on them.

| Need | What to find |
|------|--------------|
| Production data | The fetch helper the page uses, the production API base URL, and whether the record's read endpoint works without auth. If the page loads data in several calls, list them. |
| Session | The context/provider components read to know who's logged in and whether they own the record. |
| Analytics | The function(s) every event passes through on the way to each provider (your own events endpoint, GA4, ad pixels…). The debug tap goes here. |
| Backend | The path prefixes of the app's own APIs, which HTTP client makes the calls (`fetch` or an XHR-based client like axios), and the call each action awaits right before navigating away (e.g. creating a checkout session). |
| Providers | Anything live components need to render: modal hosts, query clients, theme, toasts. |
| Design system | The component library, color tokens, and any style docs in the repo. |
| Access | An admin/staff check, if the prototype should be gated. |
| Validation | The repo's lint, type-check and test commands. |

If the analytics layer has no single choke point, stop and tell the requester. Adding the tap means touching every provider call, and that change ships to production (it's inert there, but it should be reviewed).

## The five pillars

Every prototype built with this harness gives the reviewer these five levers. They are the whole point — don't drop one.

### 1. Branch off the production version (real data)

The prototype loads a real record from **production** regardless of deploy environment, so it renders with real content (staging data is usually sparse and unrepresentative). The page is a server component that fetches by ID and falls back to a built-in mock when none is given.

- Pin the fetch to the production base URL, not the environment's. See `page.tsx` in building-blocks.
- Some payloads embed slim sub-records (e.g. an item's embedded `model` omits fields the full model endpoint has). Fetch the full record separately and **merge only non-null fields**, so the richer endpoint can't clobber values the first payload already had. A plain `{...base, ...full}` spread overwrites good values with `null`; this is a recurring footgun.
- Always ship a hand-built mock (`buildMockItem`) so the prototype renders with zero setup and in tests.
- **New pages still load real records.** A new page usually shows things that already exist (items, orders, users), so fetch them from production the same way. Mock only what doesn't exist yet (a new field, a new entity type), shaped like what the backend would plausibly return, and list those shapes under Notes for the build in `DECISIONS.md`.
- Key the client component on the record id (`key={itemId ?? 'mock'}`) so switching records remounts cleanly.

### 2. Toggles for different views (viewer, state, feature flags)

All view state lives in the client harness (`<Page>Prototype.tsx`) and flows through a left **`ControlPanel`**. Three independent axes:

- **Design iteration** — a `Density`-style union type selects which body renders (`'current'` = built from live components, plus one or more redesign bodies). Adding an iteration = add a union member + a `DENSITIES` entry + a branch in the body selector. Keep the "current" body around as the before/after baseline. A new page has no "current" body; its union is just the design options.
- **Viewer / user type** — `PrototypeViewer` (e.g. `'logged-out' | 'buyer' | 'owner'`). `MockSessionProvider` supplies a static session for that viewer (logged in/out, record ownership) with no real cookies or tokens. The body branches on ownership to show buyer vs. seller UI.
- **Feature flags / experiences** — a flat `PrototypeVariants` bag of booleans (on sale, price drop, free shipping, reviews, social proof…), grouped for display via `VARIANT_OPTIONS`. `applyOverrides(base, { state, variants, viewer })` layers them onto the base record so each toggle mutates the record the body sees. Expose entity **state** (available/sold/removed/draft…) the same way.

`variantsFromItem(realItem)` seeds the toggles from a loaded real record, so what you see matches reality before you start flipping switches.

### 3. Prominent desktop ↔ mobile toggle

A segmented **Desktop / Mobile** control sits above the preview. Mobile is not a narrowed `div` — it's the **same route rendered inside an iframe** (`?embed=1`) placed in a fixed-width `PhoneFrame` (390px). The narrow iframe viewport makes real CSS breakpoints (`md:` etc.) fire exactly as they would on a phone, which a scaled container can't do.

- The iframe copy renders only the page body — no control panel, no event stream.
- Control state is mirrored into the iframe over `postMessage` with a handshake (child posts `ready` on mount → parent posts current state), so toggling controls updates the phone **without reloading or re-fetching**.
- If a redesign branches on a `mobile` **prop** instead of CSS breakpoints, you can pass the prop directly and skip the iframe. The iframe is for bodies that rely on viewport breakpoints.

### 4. Events panel (captured analytics)

A right-hand **`EventStream`** shows every analytics event the page dispatches, live, **captured only — nothing reaches real providers**. This lets you verify instrumentation while clicking around a fake page.

- It needs a **debug tap** in the analytics layer: every provider call reports to the tap first and skips the real send when capture is on. The tap is **inert in production** — with no subscriber it captures nothing and suppresses nothing. Sketch in building-blocks.
- Turn capture on for the prototype's lifetime (`setAnalyticsDebugCaptureOnly(true)` in an effect, declared before any effect that fires events).
- The iframe copy has no stream of its own, so it subscribes to its tap and forwards each event to the parent. That keeps capture engaged on the phone and puts mobile events in the same stream.
- The stream filters by channel/search and lets you expand an event to inspect and copy its JSON payload.
- Re-fire the page-view event whenever record/state/viewer changes, so the stream shows how properties differ across states.

### 5. Sandboxed backend (stub mutations, block navigation)

`usePrototypeBackend({ blockNavigation })` lets real action hooks (Add to Cart, Favorite, Make Offer, Buy Now…) run their **success paths** — which is where events fire — with no live backend:

- Non-GET calls to the app's own APIs are stubbed to resolve `200` with an empty success body; GETs pass through so data still loads.
- With `blockNavigation` on (the default), the call that precedes each hard navigation never resolves, `window.open` is neutralized, and same-tab link clicks are cancelled — so clicking an action just shows its events and stays on the page. An "Allow navigation off this page" toggle flips it for end-to-end walk-throughs.
- Everything is restored on unmount.

## The decision log (runs alongside the build)

The five pillars make the prototype reviewable. The **decision log** makes it buildable: an engineer, or
their agent, rebuilding the page in production needs to know which parts of the prototype were decided on
purpose and which were filled in along the way. AI is good at doing and only okay at deciding, so
anything the log doesn't mark as settled will get re-decided at build time.

Keep `app/prototype/<page>/DECISIONS.md` next to the code, and write it **during** the build:

- Tag every entry **Locked** (the requester's call, or agreed in review), **Data-backed** (research drove
  it; cite the source), **Flexible** (you chose it; nobody weighed in) or **Open** (undecided).
- Log a Locked entry whenever the requester redirects you, using their reason. Log research-driven choices
  as Data-backed without asking. Log your own product-visible choices as Flexible without asking.
- **Never invent a reason.** Write "No reason recorded" when none was given.
- **Call out removals (redesigns).** Anything on today's production page that the prototype drops goes under
  `## Removed` in the log, with where it went if it moved. New and moved things are easy to spot in the
  prototype; missing things aren't. Build the list by inventorying the production page, not from memory.
- **New pages: call out entry points.** A new page has nothing to remove, but existing pages change to
  link into it (a nav item, a button, a banner). List those under `## Entry points`.
- **Call out new patterns.** Any component or pattern the prototype invents rather than reuses goes under
  `## New patterns`, so the build adds it to the design system instead of making a one-off. This matters
  most on a new page.
- **Before each push**, ask one batched question about significant entries that have no reason recorded
  ("locked, and why, or a starting point?"). Don't block the push on the answer.
- Leave out harness mechanics and instrumentation details unless they reflect a product decision.
- Add a short `## Decisions` section to the PR description that links to the file.

Format, tags, the checkpoint question and the handoff ticket template:
**[references/decision-log.md](references/decision-log.md)**.

## Copy: the UI explains itself

Don't put copy on the screen when the UI already makes it clear. The goal is a page so easy to use that it
needs no explanation. **Help text is the last resort.** Before you add any, try to improve the UI until it's
self-explanatory. For every sentence you're about to add, work down this ladder and stop at the first rung
that works:

1. **Cut it.** If the label, control, value or state already says it, don't add text. No subtitles that
   restate a heading, no "Tap X to do X", no marketing lines on a settings card, no bullet lists that
   explain a rule the flow enforces anyway.
2. **Change the UI so it doesn't need saying.** Name the button after its outcome ("Start a return",
   "Accept and close returns"). Put the number or date in the control or heading ("Ship by Oct 21",
   "$174 pending · unlocks Nov 9"). Show the actual value instead of describing it (the saved address,
   not "falls back to your ship-from address"). Use a state the component already has: disable the
   control and say why in the control's summary instead of in a warning. Confirm what a save did in the
   save feedback ("Updated 23 listings") instead of predicting it in advance.
3. **Only then, help text.** Keep it for consequences the UI can't show: money, irreversible actions, and
   hidden timers or holds. One short line, next to the control it's about. Never a paragraph, and never
   a list.

Test each surviving line: cover it up. If a first-time user would still choose correctly and wouldn't be
surprised later, delete it.

- **Prototype notes stay off the page.** Reviewer hints ("viewing as a non-US buyer, so the badge is
  hidden", "try a price under $50") go in the control panel's option hints or in `DECISIONS.md`, never in
  the body. The body should look like the product would ship.
- **Copy from the plan is a requirement for what users must be able to learn, not wording to paste.**
  When the UI carries a rule without text (for example, a line that only appears on eligible items doesn't
  need to state the eligibility rule), log the dropped text in `DECISIONS.md` so the plan's owner can
  disagree.
- **Log the help text that survives** as a decision-log entry naming what the UI couldn't carry, so the
  build doesn't add more back "for consistency".

## Building blocks

| File | Role |
|------|------|
| `page.tsx` | Server component: fetch the real record from production (+ enrich/merge), fall back to mock, render the harness. `robots: noindex`. |
| `<Page>Prototype.tsx` | Client harness: owns all toggle state, the device switch + iframe sync, capture-only wiring, and the body selector. |
| `ControlPanel.tsx` | Left sidebar: design / viewer / state / experiences / record-id loader. Collapsible. |
| `EventStream.tsx` | Right sidebar: live captured analytics with channel filters + payload inspector. Collapsible. |
| `MockSessionProvider.tsx` | Static session per viewer (logged in/out, ownership) — no real auth. |
| `usePrototypeBackend.ts` | Stubs mutations + blocks navigation so actions fire events without a backend. |
| `mock<Entity>.ts` | The control vocabulary (`Density`, `PrototypeViewer`, `PrototypeVariants`, option lists) + `buildMock<Entity>`, `variantsFrom<Entity>`, `applyOverrides`. |
| `*Body.tsx` | One component per design iteration. The only files you write a lot of; everything above is harness. |
| debug tap (in the analytics layer) | `subscribeAnalyticsDebug` + `setAnalyticsDebugCaptureOnly`. The one change outside the prototype route. |

File-by-file spec with code sketches: **[references/building-blocks.md](references/building-blocks.md)**.

## Building a new prototype — steps

1. **Map the app** (first prototype only; see above). Add the debug tap to the analytics layer if it isn't there yet.
2. **Scaffold the route** at `app/prototype/<page>/`. If another prototype exists, copy its harness files; `EventStream`, `MockSessionProvider` and `usePrototypeBackend` are usually reusable almost verbatim. Otherwise build them from building-blocks.
3. **Define the control vocabulary** in the mock module: the entity type, the `Density` union of iterations, the `PrototypeVariants` you want to toggle (+ `VARIANT_OPTIONS` groups), `buildMock<Entity>()`, `variantsFrom<Entity>()`, and `applyOverrides()`.
4. **Wire production loading** in `page.tsx` for the entity's real endpoint; merge-enrich non-null if the payload embeds slim sub-records; keep the mock fallback.
5. **Build the bodies.** For a redesign, build the "current" body first from the live components as the baseline, then add redesign bodies. For a new page, build from the app's existing components and patterns; when nothing fits, make the new component and log it under New patterns. Select between bodies in the harness by the `Density` value.
6. **Run a copy pass** on every body (see [Copy](#copy-the-ui-explains-itself)): read each line of text on the page and cut it, or change the UI so it isn't needed. Keep only help text that names a consequence the UI can't show. Move any prototype notes to the control panel.
7. **Confirm all five pillars work**, in a browser: real-id load + mock fallback; viewer/state/variant toggles mutate the page; desktop↔mobile switch; events stream on interaction (desktop and phone) and don't reach real providers; actions stay on the page and fire events.
8. **Keep `DECISIONS.md` current** as you go (see the decision log above). Before each push, ask the checkpoint question about significant entries that have no reason recorded.
9. **Validate** (see below), commit, push, and open or update the draft PR with its `## Decisions` section.
10. **Hand off with one ticket per prototype**, using the handoff template in [references/decision-log.md](references/decision-log.md). Don't split the work into an epic; the engineer decides whether and how it breaks up.

## Conventions & gotchas

- **Design system only.** Prototypes ship in the app and must obey its design system: its color tokens, spacing and type scale. No generic grays or arbitrary hex. Read the repo's style docs before writing JSX/CSS. When adapting a mockup from an AI design tool, translate its palette to your tokens first — carrying the mockup's colors over is the most common miss.
- **Use the app's component library**, not raw form/UI elements, and follow the repo's code style.
- **Wrap real action components in an `ErrorBoundary`** — live components pulled into a mocked context can throw on data they don't expect.
- **The tap is inert in prod, but keep it that way**: only the prototype calls `setAnalyticsDebugCaptureOnly(true)`; never enable capture in real pages.
- **`noindex` the route** (`robots: { index: false, follow: false }`) and keep it self-contained (no marketing chrome) so it renders on a bare preview deploy. Ungated is fine for an internal prototype; to restrict to staff, gate on the app's admin check before rendering the harness.
- **Validate before reporting done:** run the repo's lint and type-check (expect 0 errors) and the prototype's colocated `__tests__/`. Cover the pure harness logic (`applyOverrides`, `variantsFrom<Entity>`, the backend stub predicate) — not rendering.
