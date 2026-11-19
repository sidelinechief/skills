# Decision log

Every prototype keeps a `DECISIONS.md` next to its code
(`app/prototype/<page>/DECISIONS.md`). It records what was decided while building, **how firm
each decision is**, and why. The prototype shows *what*; the log tells the engineer who builds it which
parts are settled and which are theirs to interpret.

The log lives on the prototype branch rather than only in the PR description. PR descriptions get
rewritten on every iteration and a build that switches threads loses its context, but a committed file
is appended to, versioned, and readable at `origin/<branch>:<path>` by whoever builds it.

## Tags

| Tag | Meaning | At build time |
| --- | --- | --- |
| **Locked** | The requester chose it, or it was agreed in review. "No reason recorded" means nobody stated one | Build it as the prototype shows |
| **Data-backed** | Research drove it; the entry cites the source | Follow it unless the data has changed |
| **Flexible** | Claude chose it while building; nobody weighed in | Implementer's call |
| **Open** | Not decided yet | Needs an answer before that part is built |

Name the source in every entry: the requester, a review (with who and when), a research artifact or
chart, or the design being rebuilt (for example "the mockup from the design tool").

## Entry format

One bullet per decision, numbered so tickets and reviews can point at it:

```
- **N19. The bar shows logo, search icon and hamburger. Cart and Inbox appear only when they have a
  badge.** Locked (agreed with Sam, PM, Sep 24). Few checkouts start from the cart, and most people
  won't have both badges at once.
```

Open questions go in their own section and are numbered `O1`, `O2`…, each ending with what needs
deciding.

**On a redesign, removals get their own section.** Anything on today's production page that the prototype drops goes
under `## Removed`, numbered `R1`, `R2`…, with the same tag, source and reason as any other entry. A
builder working from the prototype sees what's new or moved, but nothing points at what's missing, so a
removal that isn't called out either survives into the build by accident or disappears without anyone
deciding it should. Say where the thing goes instead if it moved elsewhere ("moves into the account
menu", N10), or that it's gone entirely.

```
- **R3. Favorites and Sell leave the mobile bar.** Locked (agreed with Sam, PM, Sep 24). Favorites is in
  the menu's icon row; Sell is in the menu's Selling section (N15).
```

Build the Removed list by comparing against the production page, not from memory: inventory today's
page element by element (desktop and mobile, logged in and out, every menu and drawer) and check each
one against the prototype. An element with no counterpart is a removal, even if nobody discussed it; log
it as Flexible and raise it at the next checkpoint.

The same inventory catches the opposite case: things the prototype doesn't show but the build must keep
(banners, hidden SEO links, feature-flagged variants, gated items). List them under **Not in the
prototype, but kept**, so their absence from the prototype isn't read as a removal.

**On a new page, entry points replace removals.** There's no production page to compare against, but
existing pages change to link into the new one: a nav item, a button, a banner, an email link. List each
under `## Entry points`, numbered `E1`, `E2`…, with the same tag, source and reason as any other entry.
Find them by asking where people will come from, and check each existing page you'd change on desktop
and mobile.

**New patterns get their own section.** Any component or pattern the prototype invents rather than
reuses from the design system goes under `## New patterns`, numbered `P1`, `P2`…, saying what it is and
why nothing existing fit. Without it, the build copies the prototype's one-off instead of adding the
component to the design system. On a new page this list is often long; check it as carefully as a
redesign's Removed list.

Group entries by surface (both devices / desktop / mobile, or by page section). Add a short **Notes for
the build** section for engineering concerns raised in review that are build scope, not product
questions.

## What to log and what to leave out

Log a decision when it changes what gets built: layout, content, ordering, copy, which states exist,
what's in or out of scope, experiment design, anything a reviewer argued about.

Every line of help text that survives the copy pass (SKILL.md, "Copy: the UI explains itself") gets an
entry naming the consequence the UI couldn't carry. So does plan copy the UI made unnecessary, so the plan's
owner can disagree.

Leave out the harness itself (control panel, event stream, phone frame, stubs) and instrumentation
details, unless they reflect a product decision (for example "the header ships with click events"). A
log full of harness notes buries the decisions that matter.

**Never invent a reason.** If the requester made a call without saying why, write "No reason recorded".
That gap is information: it tells the implementer the call is firm but its purpose is unknown.

## When to write entries (during the build)

Write entries as the build goes, not at the end:

1. **When the requester redirects or corrects you**, log it as Locked. They usually give the reason while
   correcting ("no, what you're missing is…"); use their words.
2. **When research drives a choice**, log it as Data-backed with the number and a link. No need to ask.
3. **When you make a product-visible choice nobody asked for**, log it as Flexible. Don't ask about it.
4. **When something on today's page is dropped**, log it under Removed straight away, with where it went
   if it moved. Before the first push, run the production-page comparison above to catch removals
   nobody mentioned.
5. **When you build a component or pattern the design system doesn't have**, log it under New
   patterns.
6. **On a new page, when you add a way into it from an existing page**, log it under Entry points.
7. **When a review settles something** (a call with the engineer, PR comments), log it as Locked with the
   reviewer and date. When a review raises something and leaves it unsettled, log it under Open.

**Checkpoint question.** Before each push, look at the entries added since the last push. If any are
Locked with no reason recorded and look significant, ask once, batched:

> Three changes have no reason logged: the tagline removal, the collapsed account section, and the
> "99+" badge cap. For each: is it locked (and why), or a starting point?

Record the answer. An unanswered item stays Locked with "No reason recorded" if the requester asked for
it, or Flexible if Claude chose it. Never block the push on the answer, and don't ask more than once per
checkpoint.

## In the PR description

Add a short `## Decisions` section near the top of the prototype PR. It links to `DECISIONS.md` and lists
the Locked, Data-backed and Open items in one line each. Flexible items are left out of the summary, and
the file stays the source of truth.

## In the handoff ticket

One ticket per prototype, in whatever tracker the team uses. The ticket carries the summary; the log
carries the detail. Map the tags onto the ticket like this:

| Tag | Ticket section |
| --- | --- |
| Locked, Data-backed | **Constraints**: one line each, with the entry number |
| Flexible | Not listed. The ticket says Flexible entries are the implementer's call |
| Open | **Decisions needed**: the question, a recommendation if there is one, and who answers |
| Removed (`R#`) | **Removed**: every entry, one line each, including Flexible ones. Nothing on today's page disappears silently |
| Not in the prototype, but kept | **Kept**: one line each, so the build doesn't drop them |
| Entry points (`E#`) | **Entry points**: every entry, one line each |
| New patterns (`P#`) | **New patterns**: one line each, so the build adds them to the design system |

Template (leave out sections with nothing in them; a new page has no Removed or Kept):

```markdown
## Goal
<One or two sentences: what the redesign is for.>

## Prototype
- PR: <link> · Branch: <branch>
- Preview: <deploy URL>/prototype/<page>
- Decision log: <link to DECISIONS.md on the branch>

## Constraints
- N3. <Locked or Data-backed decision, one line.>

## Decisions needed
- O1. <Question.> Recommendation: <…>. Answers: <who>.

## Removed
- R1. <What leaves today's page, and where it went if it moved.>

## Kept
- <What the prototype doesn't show but the build must keep.>

## Entry points
- E1. <Existing page that changes to link into the new one, and how.>

## New patterns
- P1. <Component or pattern the design system doesn't have yet, and why nothing existing fit.>

## Notes
Entries tagged Flexible in the decision log are the implementer's call. The prototype's harness
(control panel, event stream, stubs) is not part of the build.
```
