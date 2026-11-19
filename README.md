# skills

Claude Code skills from how we build at SidelineSwap. Each skill is a plain folder under `skills/`.

| Skill | What it does |
|-------|--------------|
| [prototype-harness](skills/prototype-harness/SKILL.md) | Prototype a redesign of an existing page inside your real app, against real production data |

## Install

In Claude Code:

```
/plugin marketplace add sidelinechief/skills
/plugin install prototype-harness@sidelinechief-skills
```

Or copy a skill folder (for example `skills/prototype-harness/`) into your project's `.claude/skills/`.

## prototype-harness

Prototypes a redesign of a page that already exists in your app. The prototype lives at its own
`/prototype/<page>` route, loads real production records, and runs inside your real app, so it uses your
real components, design system and analytics. It never touches the production page or needs a live
backend.

Every prototype gets:

- **Real data.** Load any production record by id, or fall back to a built-in mock.
- **Toggles.** Switch design iteration, viewer type (logged out, buyer, owner…), record state and
  feature flags from a control panel.
- **Desktop ↔ mobile.** Mobile renders the same route in a 390px iframe, so real CSS breakpoints fire.
- **Live events.** Every analytics event the page fires shows up in a side panel, and none reach your
  real providers.
- **A stubbed backend.** Buttons like Add to Cart run their success path (and fire their events)
  without writing anything or navigating away.
- **A decision log.** A `DECISIONS.md` that tells the engineer who builds it which parts were decided
  on purpose, which are theirs to call, and what the prototype removed.

**Requirements**

- A React app. The skill is written for the Next.js App Router; the pattern ports to other React setups.
- Production read endpoints the prototype can call (or a way to proxy them).
- One small change to your analytics layer: a debug tap that the prototype subscribes to. It does
  nothing unless a prototype is open. The skill explains how to add it.

Then ask Claude something like "prototype a redesign of the product page". The first prototype takes
longest, because Claude maps your app (fetch helper, session context, analytics layer, API prefixes,
design system) and builds the harness. After that, new prototypes copy the first one.
