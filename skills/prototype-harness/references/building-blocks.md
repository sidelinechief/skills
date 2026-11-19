# Prototype harness — building blocks

File-by-file spec for the harness. Read alongside `SKILL.md`. If the repo already has a prototype built with this harness, its code wins over these notes.

The sketches use a marketplace product page (`Item`) and Next.js App Router. They show the shape, not drop-in code: names like `PRODUCTION_API_URL`, `Item` and the API prefixes come from the "Map the app" step.

## The debug tap — in the analytics layer

The only change outside the prototype route. Every provider call (your own events endpoint, GA4, ad pixels…) reports to the tap first, and skips the real send when the tap says so.

```ts
// analyticsDebugTap.ts
export type DebugEvent = { id: number; at: number; channel: string; name: string; payload: unknown };
type Listener = (event: DebugEvent) => void;

const listeners = new Set<Listener>();
let captureOnly = false;
let nextId = 0;

export function subscribeAnalyticsDebug(listener: Listener) {
  listeners.add(listener);
  return () => {
    listeners.delete(listener);
  };
}

export function setAnalyticsDebugCaptureOnly(on: boolean) {
  captureOnly = on;
}

// Returns true when the caller should skip the real send.
export function tapAnalytics(channel: string, name: string, payload: unknown) {
  if (listeners.size === 0) return false; // inert: nothing captured, nothing suppressed
  const event = { id: nextId++, at: Date.now(), channel, name, payload };
  listeners.forEach((listener) => listener(event));
  return captureOnly;
}
```

```ts
// in each provider
export function fireGa4(name: string, params: Record<string, unknown>) {
  if (tapAnalytics('ga4', name, params)) return;
  window.gtag?.('event', name, params);
}
```

Suppression needs a subscriber **and** capture-only on. A real page has neither, so the tap costs production a `Set` size check.

## `page.tsx` — server component, production data

Read the record id from `searchParams`, fetch the real record **from production**, enrich it, fall back to a mock, render the client harness.

```tsx
export const metadata = { robots: { index: false, follow: false } };

// Pin the base URL to production so any real id renders with real data,
// whichever environment this prototype is deployed to.
async function fetchProductionItem(id: string) {
  const res = await fetch(`${PRODUCTION_API_URL}/items/${id}`, { cache: 'no-store' });
  if (!res.ok) return undefined;
  return ((await res.json()) as { data: Item }).data;
}

// Merge a fuller sub-record over the slim embedded one, non-null fields only —
// a plain spread would let the fuller endpoint's nulls clobber good values.
function mergeNonNull<T extends object>(base: T, full: Partial<T>): T {
  const merged = { ...base };
  for (const [key, value] of Object.entries(full)) {
    if (value != null) (merged as Record<string, unknown>)[key] = value;
  }
  return merged;
}

export default async function Page({
  searchParams,
}: {
  searchParams: Promise<{ id?: string; embed?: string }>;
}) {
  const { id, embed } = await searchParams;
  const realItem = id ? await fetchProductionItem(id) : undefined;
  return (
    <ItemPrototype
      key={id ?? 'mock'}
      baseItem={realItem ?? buildMockItem()}
      isReal={!!realItem}
      loadError={!!id && !realItem}
      embed={embed === '1'}
    />
  );
}
```

Notes:
- `loadError` lets the panel show "couldn't load, using mock".
- `embed` lets the same route render as the mobile iframe body.
- Wrap the harness in whatever providers real components need (modal hosts, query client, toasts) so pulled-in live components have their context.

## `<Page>Prototype.tsx` — client harness

Owns every toggle as `useState`: `viewer`, `state`, `variants`, `density` (iteration), `navigationAllowed`, panel collapse flags, and `device`.

- **Derived record:** `const item = useMemo(() => applyOverrides(baseItem, { state, variants, viewer }), [baseItem, state, variants, viewer])`. The body only ever sees the overridden record.
- **Capture-only wiring** runs first: `useEffect(() => { setAnalyticsDebugCaptureOnly(true); return () => setAnalyticsDebugCaptureOnly(false); }, [])`. Declare it before the page-view effect so it engages before any event fires.
- **Body selector:** branch on `density` to pick the iteration body, wrap it in `MockSessionProvider` and an `ErrorBoundary`.
- **Embed short-circuit:** `if (embed) return <main>{body}</main>` — no panels.
- Small `Segmented` (pill toggle) and `PhoneFrame` (fixed 390px wide, ~820px tall, status bar) components can live at the bottom of this file.

**Device sync.** Envelope every message with a tag and check the origin.

```ts
const MESSAGE_TAG = 'prototype-harness';
const syncRef = useRef({ density, viewer, state, variants });
syncRef.current = { density, viewer, state, variants };

// Parent: send state on the child's handshake and on every control change.
useEffect(() => {
  if (embed || device !== 'mobile') return;
  const send = () =>
    iframeRef.current?.contentWindow?.postMessage(
      { tag: MESSAGE_TAG, state: syncRef.current },
      window.location.origin,
    );
  send();
  function onMessage(e: MessageEvent) {
    if (e.origin !== window.location.origin || e.data?.tag !== MESSAGE_TAG) return;
    if (e.data.ready) send();
    if (e.data.event) addPhoneEvent(e.data.event); // forwarded analytics, shown in the stream
  }
  window.addEventListener('message', onMessage);
  return () => window.removeEventListener('message', onMessage);
}, [embed, device, density, viewer, state, variants]);

// Embed: apply incoming state, forward captured events, then say ready.
useEffect(() => {
  if (!embed) return;
  function onMessage(e: MessageEvent) {
    if (e.origin !== window.location.origin || e.data?.tag !== MESSAGE_TAG || !e.data.state) return;
    applySyncedState(e.data.state);
  }
  window.addEventListener('message', onMessage);
  const unsubscribe = subscribeAnalyticsDebug((event) =>
    window.parent.postMessage({ tag: MESSAGE_TAG, event }, window.location.origin),
  );
  window.parent.postMessage({ tag: MESSAGE_TAG, ready: true }, window.location.origin);
  return () => {
    window.removeEventListener('message', onMessage);
    unsubscribe();
  };
}, [embed]);
```

The embed's subscription matters for more than the stream: without a subscriber the tap is inert, so events fired inside the phone would reach real providers.

## `ControlPanel.tsx` — left sidebar

Collapsible `<aside>` (sticky, full height on large screens). Sections: **Design** (iteration segmented control), **Behavior** (allow-navigation checkbox), **Record data** (real-id form + mock/real badge + load error), **Viewer**, **State**, **Experiences** (variant checkboxes grouped by `VARIANT_OPTIONS[].group`, with a reset). Options come from the mock module's export lists — add a toggle there and it appears here automatically.

## `EventStream.tsx` — right sidebar

Subscribes with `subscribeAnalyticsDebug`, prepends events (cap ~300), filters by channel chips + free-text search, pause/clear, and per-row expand to view/copy the JSON payload. Highlight the app's own event channel against third-party channels (GA4, ad pixels…), and mark events forwarded from the phone.

## `MockSessionProvider.tsx`

Given a `viewer`, provides a static value for the app's session context: logged out (no user), or logged in as a synthetic user whose id/username is the record owner's (owner view) or a fixed buyer's. No real cookies or tokens. All session methods (log in, log out, refresh) are no-ops.

## `usePrototypeBackend.ts`

Two effects.

**1. Request stub (always on).** Non-GET calls to the app's own APIs resolve `200` with an empty success body in the shape the app's client expects; GETs pass through. When `blockNavigation` is on, calls that precede a hard navigation never resolve (the caller awaits them before leaving the page).

```ts
const API_PREFIXES = ['/api/']; // the app's own API path prefixes
const NAVIGATING_CALLS = [/^\/api\/checkout\/sessions$/]; // calls awaited right before a hard navigation
const STUB_BODY = { data: {} };

function pathOf(url: string) {
  return new URL(url, 'http://localhost').pathname;
}

export function isStubbableApiRequest(url: string, method = 'GET') {
  if (method.toUpperCase() === 'GET') return false;
  const path = pathOf(url);
  return API_PREFIXES.some((prefix) => path.startsWith(prefix));
}

export function isNavigatingCall(url: string, method = 'GET') {
  return method.toUpperCase() !== 'GET' && NAVIGATING_CALLS.some((re) => re.test(pathOf(url)));
}

useEffect(() => {
  const realFetch = window.fetch;
  window.fetch = (input, init) => {
    const url = typeof input === 'string' ? input : input instanceof URL ? input.href : input.url;
    const method = init?.method ?? (input instanceof Request ? input.method : 'GET');
    if (blockNavigation && isNavigatingCall(url, method)) return new Promise<Response>(() => {});
    if (isStubbableApiRequest(url, method)) {
      return Promise.resolve(
        new Response(JSON.stringify(STUB_BODY), {
          status: 200,
          headers: { 'Content-Type': 'application/json' },
        }),
      );
    }
    return realFetch(input, init);
  };
  return () => {
    window.fetch = realFetch;
  };
}, [blockNavigation]);
```

If the app's HTTP client uses XHR (axios and similar), a `fetch` shim won't see its calls. Stub at the client instead, with a request interceptor or adapter that applies the same two predicates.

**2. Navigation trap (when `blockNavigation`).** Override `window.open`, and `preventDefault` same-tab `<a href>` clicks in the capture phase (leave `#` links and `target="_blank"` alone). Restore on cleanup.

`location.assign` and `location.replace` can't be reliably overridden: browsers define them as non-writable. Find where the app calls them and make sure each is preceded by an awaited call listed in `NAVIGATING_CALLS`, or routed through a helper the harness can intercept. Check in a browser that every action actually stays on the page.

Keep both predicates pure and exported so they can be unit-tested.

## `mock<Entity>.ts` — control vocabulary + builders

The single source for what the panel can toggle:
- `PrototypeViewer`, `Density` (union of iterations) + `DENSITIES`, `VIEWERS`, `STATES` display lists.
- `PrototypeVariants` (flat boolean bag) + `DEFAULT_VARIANTS` (default the redesign's richer signals **on** so a new iteration shows its full surface) + `VARIANT_OPTIONS` (label + group for the panel).
- `buildMock<Entity>()` — a fully populated fake record so the prototype renders with no setup and in tests.
- `variantsFrom<Entity>(record)` — infer toggle state from a loaded real record.
- `applyOverrides(base, { state, variants, viewer })` — return a shallow clone with each variant/state layered on. This is the heart of the "feature flag" behavior; keep it pure and unit-tested.

## Testing

Colocated `__tests__/` covering the **pure harness logic**, not rendering: `applyOverrides` (each variant produces the expected record shape), `variantsFrom<Entity>` (inference), and `isStubbableApiRequest` / `isNavigatingCall` (GET passthrough vs. mutation stub vs. held navigation). Run with the repo's test command scoped to `app/prototype/<page>/__tests__/`.
