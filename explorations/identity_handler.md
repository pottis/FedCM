# FedCM Identity Handler — Explainer

**Authors:**
Suresh Potti (sureshpotti@microsoft.com)

## Participate

- https://github.com/w3c-fedid/FedCM/issues/80
- https://github.com/w3c-fedid/FedCM/pull/815 (spec PR)

## Summary

This document describes the **FedCM Identity Handler**, a proposed feature that would allow Identity Providers (IDPs) to register a Service Worker capable of intercepting credentialed FedCM requests (accounts, id_assertion, disconnect).

## Status

This explainer documents a **proposed addition** to the FedCM specification. The proposal would let an IDP-controlled Service Worker intercept FedCM's credentialed fetches (accounts, id_assertion, disconnect).

The underlying request was discussed in the WG on [23 September 2025](https://github.com/w3c-fedid/meetings/blob/main/2025/2025-09-23-FedCM-notes.md) and editors agreed to pursue it; the credentialed-endpoints-only scoping cleared privacy review. Spec mechanics — registration model, internal-field preservation, header allowlist, and the dispatch shape (re-using `FetchEvent` vs. a new dedicated event) — are still being worked out. See [Open Design Discussions](#open-design-discussions).

## Problem Statement

The credentialed FedCM endpoints (accounts, id_assertion, disconnect) authenticate the user to the IDP using cookies. Today these requests have `service-workers mode: "none"`, so the IDP has no programmable seam between FedCM and its own network stack. The discussion on [Issue #80](https://github.com/w3c-fedid/FedCM/issues/80) surfaced four concrete problems this creates.

### 1. Bearer-token theft on the IDP cookie

**Challenge.** The cookies sent on the accounts and id_assertion calls are **bearer credentials** — anyone in possession of the bytes can replay them, often for the lifetime of the session (months). Token theft is a primary cause of identity-related security incidents, and modern enterprise IDPs increasingly require the cookie to be accompanied by a fresh, device-bound, proof-of-possession assertion before they will honor it. Microsoft Entra, for example, only accepts its `ESTSAUTH*` cookies if an `x-ms-RefreshTokenCredential` JWT — containing the device-wide SSO token and a server-issued nonce, signed by a TPM-backed device key — is attached to the request. Today the only way to get that header onto a FedCM-style call is via a browser extension (Chrome) or a non-standard browser API (Edge); see the [Platform Authentication explainer](https://github.com/MicrosoftEdge/MSEdgeExplainers/blob/main/PlatformAuthentication/explainer.md). FedCM as specified gives no place for that proof to be attached.

**How SW interception helps.** If FedCM's credentialed calls were routed through an IDP-controlled Service Worker — via a `fetch` event, a dedicated identity event, or another dispatch shape still under discussion — the SW could compute its proof (DPoP, mTLS-style signature, Entra's device-key JWT, WebAuthn-derived assertion, etc.) and attach it as a request header before forwarding the request to the network. The bearer cookie alone would no longer be sufficient to replay the call — the attacker would also need the device-resident private key.

> **Why not just "OAuth profile + DPoP at code redemption"?** [aaronpk's OAuth profile for FedCM](https://github.com/aaronpk/oauth-fedcm-profile) shows that DPoP-bound *RP* tokens can already be obtained without browser changes by treating the FedCM response as an authorization code and DPoP-binding the subsequent code→token exchange. That solves RP-side token binding but does **not** address the IDP-cookie bearer problem (the cookie sent on the accounts call is still a plain bearer), and the extra round trip is costly for bundle apps that talk to many IDP-issued resources (e.g., Teams). SW interception is targeted at the gap the OAuth-profile path leaves open.

### 2. Hard failures during IDP outages

**Challenge.** When the IDP backend is degraded — a regional incident, a partial accounts-service outage, a brief maintenance window — the FedCM call fails outright and the user sees a sign-in failure. The IDP has no opportunity to apply the resiliency patterns (regional failover, last-known-good cached account data) it routinely uses on its own surface.

**How SW interception helps.** The SW is a first-class place to implement these patterns: catch the network error, retry against a backup region, or fall back to a cached account list that's recent enough to render the FedCM account chooser. The FedCM contract with the RP is unchanged; the user sees a working flow instead of a hard failure.

### 3. Geographic / sovereignty-aware routing

**Challenge.** Some IDPs are required (by data-residency regulations or by tenant configuration) to route a given user's authentication traffic to a specific regional service. The FedCM `config.json` declares a single set of endpoint URLs per IDP origin and is fetched without user context, so there is no point at which the IDP can pick a region based on the actual signed-in user before the accounts call goes out.

**How SW interception helps.** The SW runs in the IDP origin and has access to IDP-side state (cookies, IndexedDB) that identifies the user. It can rewrite the outbound request URL to a regional endpoint at the moment the FedCM call happens, without changing the public `config.json` or requiring per-tenant config files.

### 4. Bridging non-FedCM identity backends

**Challenge.** An IDP with an existing identity backend whose wire format does not exactly match FedCM's accounts / id_assertion shape would otherwise have to deploy a server-side translation layer in front of every FedCM endpoint. This is operationally heavy and slows adoption — especially for IDPs experimenting with FedCM alongside an existing OIDC / SAML stack.

**How SW interception helps.** The SW can act as the translation layer at the edge: receive the FedCM-shaped request, call the IDP's existing backend in its native format, and synthesize a FedCM-shaped `Response` to return via `respondWith()`. No backend rewrite is required to participate in FedCM.

## Proposed Solution

### Candidate Approaches Considered

Three approaches to opening the SW seam were evaluated before converging on the current direction.

#### Approach A — Flip `service-workers mode` to `"all"` and let standard SW dispatch handle it

The most conservative spec change: replace `service-workers mode: "none"` with `"all"` on FedCM's credentialed requests. Fetch's *HTTP fetch* algorithm would then call the Service Worker spec's [*Handle Fetch*](https://www.w3.org/TR/service-workers/#handle-fetch), which fires a `FetchEvent` at the controlling SW.

**Why this cannot work as a spec-only change.** *Handle Fetch* dispatches via the request's **client** — but per the [FedCM spec](https://fedidcg.github.io/FedCM/), every endpoint (disconnect, well-known, config, accounts, account picture, id_assertion, client_metadata) sets `client = null` in its request shape. That choice is structural — FedCM requests are initiated by the UA, not by any document — so *Handle Fetch* has nothing to dispatch through and the request goes straight to the network.

#### Approach B — Browser-side dispatch layer that calls the SW directly, bypassing standard `Handle Fetch`

An alternative would be for the browser to ship a FedCM-specific dispatch layer that bypasses standard *Handle Fetch* entirely: look up the IDP's SW registration by URL **scope** (not by a controlling client, which FedCM doesn't have), construct an internal request from the FedCM call, invoke the SW's `fetch` event directly, and fall back to the network if no SW is registered or the SW does not respond. The intent is to provide the missing dispatch primitive purely on the browser side — and this has been prototyped, demonstrating it is implementable today without spec changes to Fetch or Service Worker.

However, the corresponding **spec direction** — extending Fetch / Service Worker so the *spec* can dispatch a `FetchEvent` to a SW with no controlling client — was not endorsed by reviewers on the [Chromium service-worker-discuss thread](https://groups.google.com/a/chromium.org/g/service-worker-discuss/c/t9d33x6l718):

- **No initiating client** (Ben Kelly). Standard SW semantics dispatch `FetchEvent` to the SW controlling the *initiating client*. FedCM has no initiating client, and the dispatch target is the *destination* origin's SW — the same dispatch shape used by the deprecated [foreign fetch](https://github.com/whatwg/fetch/issues/506), which the SW WG removed for similar reasons.
- **Risk to deployed SWs.** Any IDP SW with a generic `fetch` handler would suddenly receive FedCM requests it was not written to handle. The opt-in is implicit (the SW exists), not explicit.
- **Procedural asks** (Dominic Farolino). A formal spec design section, Chrome security / privacy review, cross-vendor review, and TAG review were called out as prerequisites before reusing `FetchEvent` for this dispatch shape.

The browser-side dispatch is implementable today, but the spec path of "reuse `FetchEvent` via *Handle Fetch*" was not pursued on the basis of the thread feedback.

#### Approach C — Dedicated event, modeled on Payment Handler (current direction)

The shape suggested by reviewers in the SW WG thread, introduce a purpose-built event (`IdentityRequestEvent`) that the SW must explicitly listen for. The dispatch model is borrowed directly from the [Payment Handler API](https://w3c.github.io/web-based-payment-handler/), which faces the same shape of problem (UA-initiated request, destination-origin SW, no controlling client) and resolves it the same way — Payment Handler dispatches `"paymentrequest"` (and `"canmakepayment"`) using the Service Worker spec's [*Fire Functional Event ... on registration*](https://www.w3.org/TR/service-workers/#fire-functional-event) primitive, which targets a `ServiceWorkerRegistration`'s active worker directly **without** going through *Handle Fetch* or `request's client`. That is exactly the gap that blocks Approach A, and Approach C borrows the same primitive for FedCM. (The corresponding registration surface — how the SW gets associated with a specific IDP config in the first place — is a separate question, captured in [Open Design Discussions §b](#b-service-worker-registration).)

The dedicated-event design makes the unusual dispatch model **explicit at the API surface** rather than overloading `FetchEvent`. It addresses each of the Approach B concerns:

- The new event has no client-resolution dependency — dispatch is anchored to whichever IDP-config-to-SW-registration binding the registration surface (see [§b](#b-service-worker-registration)) defines, and fired directly on that registration.
- Pre-existing IDP SWs that only listen for `fetch` are not affected; only SWs that explicitly register the new event type participate.
- Being a different event entirely sidesteps the "is this foreign fetch in disguise?" critique against reusing `FetchEvent`.

This is the approach described in *How It Works* below.

### How It Works

This section describes **Approach C** in detail.

#### 1. IDP Registers a Service Worker for FedCM

The IDP makes its Service Worker available to the FedCM dispatch path. The Service Worker is hosted at the IDP origin and only sees `identityrequest` events when it has explicitly indicated that it is intended to handle them.

The precise registration surface — how the IDP signals that intent to the user agent — is an open design question. See [Open Design Discussions §b](#b-service-worker-registration) for the options under consideration.

#### 2. Browser Dispatches `IdentityRequestEvent` to the SW

When the browser needs to fetch an IDP endpoint (accounts, token, disconnect), it first dispatches an `IdentityRequestEvent` to the registered SW:

```webidl
enum IdentityRequestEndpoint {
    "accounts",
    "id_assertion",
    "disconnect"
};

[Exposed=ServiceWorker]
interface IdentityRequestEvent : ExtendableEvent {
    constructor(DOMString type, IdentityRequestEventInit eventInitDict);

    readonly attribute IdentityRequestEndpoint endpoint;
    readonly attribute Request request;

    [RaisesException] undefined respondWith(Promise<Response> r);
};

dictionary IdentityRequestEventInit : ExtendableEventInit {
    required IdentityRequestEndpoint endpoint;
    required Request request;
};
```

**Key properties:**
- `endpoint` — An `IdentityRequestEndpoint` enum value: `"accounts"`, `"id_assertion"`, or `"disconnect"`
- `request` — A `Request` object with the original URL, method, body (for POST), and
- `destination: "webidentity"` (which sets `Sec-Fetch-Dest: webidentity`)
- `respondWith(promise)` — The SW provides a `Response` to the browser, same pattern as `FetchEvent.respondWith()`

#### 3. SW Handles the Event

```javascript
// /idp-sw.js — IDP's Identity Handler Service Worker

self.addEventListener('identityrequest', (event) => {
  switch (event.endpoint) {
    case 'accounts':
      // Outage resiliency: try the network, fall back to a recent cache
      // snapshot so the FedCM account chooser still renders during a
      // partial backend outage. (See Problem 2.)
      event.respondWith(
        fetch(event.request)
          .then(response => {
            if (response.ok) {
              caches.open('fedcm').then(c => c.put(event.request, response.clone()));
            }
            return response;
          })
          .catch(() => caches.match(event.request))
      );
      break;

    case 'id_assertion':
      // Token binding: attach a fresh DPoP proof so the IDP only honors
      // the session cookie when accompanied by the device-resident key.
      // (See Problem 1.)
      event.respondWith((async () => {
        const proof = await generateDPoPProof(event.request.url, 'POST');
        const augmented = new Request(event.request, {
          headers: { ...event.request.headers, 'DPoP': proof }
        });
        return fetch(augmented);
      })());
      break;

    case 'disconnect':
      // Bridging a non-FedCM backend: forward to the IDP's existing
      // revocation endpoint (which doesn't return the FedCM-required
      // shape) and synthesize a {account_id} response so the UA can
      // remove the account from its connected accounts set.
      event.respondWith((async () => {
        const body = await event.request.clone().text();
        const accountId = new URLSearchParams(body).get('account_hint');
        const upstream = await fetch('/internal/revoke', {
          method: 'POST',
          body
        });
        if (!upstream.ok) {
          // Surface the failure; the UA will remove all accounts for
          // this (RP, IDP) pair, per the spec's failure handling.
          return upstream;
        }
        return Response.json({ account_id: accountId });
      })());
      break;
  }
});
```

#### 4. Fallback to Network

If the SW does not call `respondWith()`, rejects the promise, returns a non-OK response, returns a response of a disallowed type, or times out (default 10 seconds), the browser **transparently falls back** to a normal network fetch — as if the SW did not exist. This ensures the feature is purely additive: IDPs that opt in get SW capabilities, but failures never break the FedCM flow.

### Selective Endpoint Policy

Only credentialed authentication endpoints are dispatched to the SW:

| Endpoint | SW Dispatch | Reason |
|----------|-----------|--------|
| `/accounts` | ✅ Yes | User account data — benefits from caching and augmentation |
| `/token` (id_assertion) | ✅ Yes | Token generation — can use DPoP, custom logic |
| `/disconnect` | ✅ Yes | Account management — graceful error handling |
| `/.well-known/web-identity` | ❌ No | IDP discovery — privacy protection |
| `/config.json` | ❌ No | Configuration — privacy protection |
| `/client_metadata` | ❌ No | RP metadata — privacy protection |

### Augmenting Headers

When the SW forwards a FedCM request, it may attach additional headers (e.g., a `DPoP` proof).
Headers UA-set on the FedCM internal request (`Accept`, `Sec-Fetch-*`, `Cookie`, `Origin`,
`Referer`) must **not** be overridden by the SW.

The exact set of header names the SW is permitted to add, and the process for growing that set, is still being worked out (called out in [Status](#status) alongside the other open spec mechanics).

### Response Type Restriction

The `Response` returned to `respondWith` must satisfy: its [URL](https://fetch.spec.whatwg.org/#concept-response-url) is same-origin to the IDP, **or** its [type](https://fetch.spec.whatwg.org/#concept-response-type) is `"default"` (SW-synthesized via `new Response(...)`). `"opaque"`, `"opaqueredirect"`, and `"error"` are rejected outright. `"cors"` is accepted only when the URL is same-origin to the IDP — the legitimate shape for `id_assertion` / `disconnect`, where `mode: "cors"` plus an RP-origin request makes Fetch tag the response `"cors"` even though it came from the IDP itself. Any other origin is rejected, blocking the SW from laundering cross-origin data through the FedCM channel. 

## Benefits

### DPoP (Demonstration of Proof-of-Possession)

The primary motivating use case. DPoP binds tokens to the specific client that requested them, preventing token theft:

```javascript
self.addEventListener('identityrequest', (event) => {
  if (event.endpoint === 'id_assertion') {
    const dpopProof = await generateDPoPProof(event.request.url, 'POST');
    const augmented = new Request(event.request, {
      headers: { ...event.request.headers, 'DPoP': dpopProof }
    });
    event.respondWith(fetch(augmented));
  }
});
```

### Graceful Degradation During Outages

```javascript
self.addEventListener('identityrequest', (event) => {
  if (event.endpoint === 'accounts') {
    event.respondWith(
      fetch(event.request)
        .then(response => {
          if (response.ok) {
            // Cache successful response
            caches.open('fedcm-cache').then(c => c.put(event.request, response.clone()));
          }
          return response;
        })
        .catch(async () => {
          // Network failed — serve stale cache
          const cached = await caches.match(event.request);
          if (cached) return cached;
          throw new Error('No cached accounts available');
        })
    );
  }
});
```

### RP Transparency

RPs require **zero changes**. The FedCM API contract is unchanged:

```javascript
// Standard FedCM usage — works identically with or without Identity Handler
const credential = await navigator.credentials.get({
  identity: {
    providers: [{
      configURL: "https://idp.example/config.json",
      clientId: "rp_client_123",
      nonce: "random_nonce_value"
    }]
  }
});
console.log('Token:', credential.token);
```

## Privacy Considerations

### Configuration Endpoints Are Protected

Configuration endpoints (`/.well-known`, `/config.json`, `/client_metadata`) are **not** dispatched to the SW for privacy reasons:

- These endpoints are fetched with privacy-preserving properties (`credentials: "omit"`, opaque origin, no referrer)
- If intercepted, the SW could correlate user identity (via cookies from its own origin) with RP identity (from `client_metadata` URL parameters)
- Keeping these endpoints out of SW scope preserves the privacy boundary

### IDP Trust Model

The Identity Handler SW runs at the IDP's origin and sees the same information that IDP servers already see:
- `/accounts` (GET) — no RP identity visible (opaque origin)
- `/token` (POST) — RP client_id in the POST body (already visible to IDP server)
- `/disconnect` (POST) — RP client_id in the POST body (already visible to IDP server)

No new information is exposed beyond what the IDP server already receives. The SW has the same visibility as the IDP backend.

### User Consent

User consent is still required before any tokens are issued. The SW cannot bypass the FedCM consent UI — it only augments the network layer between the browser and the IDP server.

### Discussion: Privacy Attacks Considered

The credentialed-endpoints-only scoping in [Selective Endpoint Policy](#selective-endpoint-policy) is the result of explicit attack analysis on Issue #80. Two attacks were considered:

**Attack A — RP iframes IDP, then invokes FedCM.** The RP embeds the IDP in an iframe so the IDP's SW (or storage) sees the RP's identity, then invokes FedCM; the IDP SW correlates the RP with the user on the accounts call. **Conclusion:** this attack does not introduce new surface. Browsers already partition IndexedDB (and other SW-accessible storage) by top-frame origin, so an iframed IDP cannot stash RP context that a top-frame FedCM SW can read. To the extent IDP↔RP correlation is possible today via unpartitioned cookies or pop-ups, that correlation channel exists with or without SW interception.

**Attack B — uncredentialed endpoint logs the RP, credentialed endpoint reads it.** If SW interception were enabled on uncredentialed endpoints (e.g., `client_metadata`), the IDP SW could record the RP identity from that call (which legitimately includes the RP's `client_id`) and read it back during the subsequent accounts call — combining user identity (via cookies on the accounts call) and RP identity in the same SW context. **Conclusion:** real attack. Resolved by restricting SW interception to credentialed endpoints only (accounts, id_assertion, disconnect) and continuing to fetch configuration / metadata / well-known endpoints UA-direct with the existing privacy properties.

The privacy reviewer for the WG cleared the credentialed-only proposal on this basis.

## Security Considerations

### Origin Isolation

The Identity Handler SW is registered at the IDP origin and can only intercept requests destined for that origin:

```
✅ idp.example's SW intercepts: requests TO idp.example/accounts
❌ idp.example's SW cannot intercept: requests TO other-idp.com/accounts
❌ rp.com's SW cannot intercept: requests TO idp.example/accounts
```

Cross-origin Service Worker interception is architecturally impossible.

### Sec-Fetch-Dest: webidentity

On the UA-direct path, the browser stamps `Sec-Fetch-Dest: webidentity` — a forbidden header JavaScript cannot set — so the IDP can confirm the request came from the FedCM API, not from page-initiated script. In the SW path, `event.request.destination` is read-only and equal to `"webidentity"`. Whether the header reaches the IDP unchanged when the SW forwards via `fetch(event.request)` — and whether the spec should mechanically enforce that it does — is an open question, captured in [Open Design Discussions §c](#c-should-the-spec-enforce-lock-down-properties-when-fetch-is-called-from-the-sw).

### Redirect Prevention

FedCM requests use `redirect mode: "error"` — the SW cannot redirect requests to different URLs. In addition, the response-type filter rejects any response whose origin differs from the IDP's, so the SW cannot launder a cross-origin response back into the FedCM flow either.

### Robust Fallback

The implementation uses a "skip flag" pattern for fallback:
1. If SW dispatch fails → emit console warning
2. Set `skip_identity_handler` flag → bypass SW on retry
3. Re-issue the same request via normal network fetch
4. Clear the flag for future requests

This ensures that SW failures never block authentication — the flow degrades gracefully to the non-SW path.

## Developer Experience

**IDP developers** opt in by:
1. Registering a Service Worker that handles FedCM (mechanism under discussion — see [Open Design Discussions §b](#b-service-worker-registration))
2. Implementing an `identityrequest` event handler in their SW
3. Using standard web APIs (Fetch, Cache, Crypto) within the handler, subject to the augmenting header allowlist

**RP developers**: No changes required. Completely transparent.

**Debugging**: Console warnings are emitted when the SW fails:
- "FedCM: No identity handler service worker registration found for the provider. Falling back to network."
- "FedCM: The identity handler service worker has no active version. Falling back to network."
- "FedCM: The identity handler service worker failed to start. Falling back to network."
- "FedCM: The identity handler response had a non-ok status (NNN). Falling back to network."

## Open Design Discussions

### a) Dispatch Shape — `FetchEvent` vs. a Dedicated Event

How should the FedCM credentialed call surface inside the SW? The two shapes under consideration are:

- **Reuse `FetchEvent`.** The browser routes the FedCM internal request through the standard SW dispatch path; the IDP's existing `fetch` handler sees the request (distinguished by `Sec-Fetch-Dest: webidentity` and `request.destination === "webidentity"`) and decides whether to handle it.
- **Introduce a dedicated event** (e.g., `IdentityRequestEvent`). The browser dispatches a purpose-built event the SW must explicitly listen for; SWs that only have a `fetch` handler are not invoked.

The tradeoffs were discussed on the [Chromium service-worker-discuss thread](https://groups.google.com/a/chromium.org/g/service-worker-discuss/c/t9d33x6l718). The service-worker editors' core architectural objection to reusing `FetchEvent` was stated directly on that thread:

> FedCM's request — which has no controlled client and is dispatched to the *destination* origin's SW rather than an *initiating* origin's SW — does not fit standard subresource-fetch semantics, and resembles the deprecated [foreign fetch](https://github.com/whatwg/fetch/issues/506).

In standard `FetchEvent` semantics, the controlling SW is the one belonging to the **page that initiated the request** — the request's *client*. FedCM inverts both halves of that contract:

- **No client.** FedCM sets `request's client = null` on every credentialed endpoint, because the request is initiated by the UA in response to the FedCM API call, not by a document-bound subresource fetch. Standard *Handle Fetch* has nothing to dispatch through.
- **Destination-origin dispatch, not initiator-origin.** The SW we want to invoke belongs to the **IDP** (the request's destination), not to the RP page that initiated the FedCM call. This is the exact inversion that the [foreign fetch](https://github.com/whatwg/fetch/issues/506) proposal tried to standardise — and that was removed from the platform for being too dangerous a primitive (cross-origin SWs intercepting requests destined for their own origin, with all the trust-boundary questions that opens).

In plain terms: reusing `FetchEvent` for FedCM is foreign fetch in a smaller box — same shape (one origin's SW intercepting requests aimed at it from elsewhere), just narrower scope. That is what the editors pushed back on, and what the dedicated-event option avoids by giving this unusual dispatch its own event instead of overloading `FetchEvent`.

The remaining tradeoff is spec cost vs. IDP-deployment risk:

| Aspect | `FetchEvent` reuse | Dedicated event |
|---|---|---|
| **Spec surface** | Smallest — a flag flip on `service-workers mode` plus the augmenting-header rules | Larger — new IDL, new dispatch algorithm |
| **Risk to deployed SWs** | Pre-existing IDP SWs (caching, push, offline) would silently start receiving FedCM requests; every existing `fetch` handler needs a `destination === "webidentity"` early-exit retrofit | None — the new event is invisible to existing handlers |

### b) Service Worker Registration

How does an IDP register a Service Worker for FedCM, and how does the UA know which registration to dispatch `identityrequest` events to?

#### 1. SW path/scope declared in well-known + IDP-page registration

**How it works.**

1. The IDP's `/.well-known/web-identity` declares an `identity_handler` field — either a literal SW path or a scope pattern:

   ```json
   {
     "provider_urls": ["https://idp.example/fedcm/config.json"],
     "identity_handler": "/fedcm/sw.js"      // literal path
     // -or-
     "identity_handler": "/fedcm/*"          // scope pattern
   }
   ```

2. The IDP **page** calls the standard `navigator.serviceWorker.register("/fedcm/sw.js")` — Document-owned registration, standard SW install / activate / update lifecycle, SW lands in the IDP's first-party storage partition.

3. When a FedCM call happens, the browser will only forward it to a service worker whose URL was pre-listed in `identity_handler` in the well-known file — any other service worker on the IDP origin is ignored. The eligible registration receives `identityrequest` via [*Fire Functional Event ... on registration*](https://www.w3.org/TR/service-workers/#fire-functional-event).

Since the well-known file is served by the IDP's own server, an XSS on any IDP page cannot tamper with the allowlist.

**Literal path vs. scope pattern — IDP operational trade-off.**

| Aspect | Literal path (`/fedcm/sw.js`) | Scope pattern (`/fedcm/*`) |
| --- | --- | --- |
| Filename change | Edit well-known each time | Free under the prefix |
| Content-hashed builds (`sw.a3f7b2.js`) | Forces a shim or breaks the convention | Native fit |
| Gradual rollout / canary / A/B | Server-side cohort variation on one URL | Per-cohort URLs under the prefix |
| Multi-tenant / multi-env | One SW for all tenants | Sub-scope per tenant/env |
| Spec & IDP complexity | Trivial (string equality) | Requires URL-matching grammar |

#### 2. Dedicated manager extending `ServiceWorkerRegistration` (`registration.identity`)

**How it works.**

1. The IDP page calls standard `register()` plus a new IDL surface borrowed from the [Payment Handler API](https://w3c.github.io/web-based-payment-handler/) (which uses `registration.paymentManager`):

   ```js
   const reg = await navigator.serviceWorker.register("/fedcm/sw.js");
   await reg.identity.register("https://idp.example/fedcm/config.json");
   ```

2. `registration.identity` is an `IdentityProviderManager`. Calling `identity.register(configURL)` declares a one-to-one binding between this SW registration and the named IDP config. Same-origin enforcement applies: `configURL` must be same-origin to the SW script. Lifecycle: `identity.unregister(configURL)`, or implicit removal when the parent registration is removed.

3. The UA records `(configURL → registration)` and dispatches `identityrequest` on FedCM calls matching that configURL via the same *Fire Functional Event ... on registration* primitive.

Brings the cleanest per-configURL binding — one SW per IDP config, mirroring the Payment Handler architectural precedent and giving multi-tenant IDPs a dedicated SW per config out of the box.

**IDP operational constraints.**

| Constraint | What it means for the IDP |
| --- | --- |
| Login-flow coupling | SW only registers when an IDP page runs `identity.register()`. No IDP page open → no registration → FedCM falls back to UA-direct. |
| XSS exposure | Any IDP-origin JS that can call `register()` + `identity.register()` claims the slot. No server-side allowlist; defense = strict XSS hygiene on every IDP page. |
| Per-configURL bookkeeping | One `register()` / `unregister()` call per supported configURL. |
| Feature detection | New IDL — IDP must check `'identity' in reg` and fall back on older browsers. |
| No declarative discovery | Nothing in well-known or `config.json` advertises the SW; only observable after `identity.register()` runs. |

#### Cross-cutting reviewer concerns (apply to all three)

- Pre-existing SWs (deployed for caching, push, offline) must not accidentally be invoked for FedCM if they add an `identityrequest` listener.
- FedCM-purposed SWs should not be reachable by unrelated platform features that look up the same registration.
- The unregistration story should be clear (logout, `Clear-Site-Data`, IDP removal of opt-in).

#### Backed-out shape

- **SW URL declared in the IDP `config.json`** (e.g., `"identity_handler": { "service_worker": "/sw.js" }` inside `config.json`). **Fails privacy** — the IDP can encode RP-specific values into the URL (`/sw-for-rp-A.js` vs. `/sw-for-rp-B.js`), so the SW (and the IDP server serving the SW script) would learn which RP the user is interacting with.

### c) Should the Spec Enforce Lock-Down Properties When `fetch` Is Called From the SW?

On the UA-direct path, the FedCM internal request has a deliberately hardened shape on the wire:

| Property | Value | Why |
|---|---|---|
| `client` | `null` | No ambient authority leaks |
| `origin` | accounts: fresh **opaque origin**<br>id_assertion / disconnect: **RP document's origin** | Cookie layer treats the request as cross-site to the IDP → only `SameSite=None` cookies are sent. For accounts the opaque origin also keeps the IDP from learning the RP. For id_assertion / disconnect the IDP **needs** the RP origin — to mint a token bound to it and to authorize sharing the response via CORS — so the RP origin is on the wire instead. |
| `destination` | `"webidentity"` | Stamps `Sec-Fetch-Dest: webidentity` on the wire (the FedCM CSRF marker) |
| `redirect mode` | `"error"` | Prevents IDP-controlled redirects |
| `referrer policy` | `"no-referrer"` | Suppresses the `Referer` header (the `Origin` header still carries RP identity on id_assertion / disconnect by design) |
| `mode` | accounts: `"no-cors"`<br>id_assertion / disconnect: `"cors"` | accounts response is consumed by the UA only — no CORS handshake needed. id_assertion / disconnect responses flow to the RP — CORS is the IDP's explicit opt-in (premise S4). |
| `credentials mode` | `"include"` | Sends the IDP session cookie |
| `Accept` | UA-set per endpoint | Forces request / response shape |

When the SW forwards via `fetch(event.request)`, the internal request fields `client`, `origin`, and `destination` are not exposed (or are only read-only) through the public `Request` interface today, so they can be reset as the request flows through the [Fetch](https://fetch.spec.whatwg.org/) `Request` constructor. If they are reset, the wire diverges from the UA-direct shape:

| Wire | UA-direct | Naive `fetch(event.request)` |
| --- | --- | --- |
| `Sec-Fetch-Dest` | `webidentity` | `empty` ❌ |
| `Sec-Fetch-Site` | `cross-site` | `same-origin` ❌ |
| `Origin` | `null` (accounts) / `https://rp.example` (id_assertion, disconnect) | `https://idp.example` ❌ |
| `Cookie` scope | `SameSite=None` only | All cookies, including `SameSite=Lax` / `Strict` ❌ |

In that downgraded form, the IDP server could no longer tell the SW-mediated request apart from a page-initiated XHR, the FedCM CSRF marker (`Sec-Fetch-Dest: webidentity`) would be missing, and `SameSite=Lax` / `Strict` cookies would be exposed.

**Does the IDP's own PoP proof make the lock-down redundant?**

A reasonable counter-argument: if an SW already attaches a DPoP / Entra-JWT / mTLS-style proof, does the UA-stamped `Sec-Fetch-Dest: webidentity` (and the rest of the lock-down) still matter? Three reasons it does:

1. **Not every IDP attaches PoP.** SW interception is opt-in, and even an IDP that ships a SW may not attach a proof to every request. The spec must work for IDPs that rely only on cookies — for them, `Sec-Fetch-Dest: webidentity` is the universal CSRF marker that lets the IDP server tell a real FedCM call apart from a page-initiated XHR.
2. **PoP and `Sec-Fetch-Dest` answer different questions.** PoP says "this request came from a holder of this device's private key." `Sec-Fetch-Dest: webidentity` says "this request came through the FedCM API surface, with the UA's mediation and consent UI." An IDP may legitimately want both: a credential honored only if *both* device-bound *and* arriving on the FedCM path. PoP alone does not vouch for the path the request took.
3. **The cookie-scope invariant is unrelated to PoP.** Preserving the request `origin` set by the UA (opaque for accounts; RP origin for id_assertion / disconnect) is what keeps the cookie layer from sending `SameSite=Lax` / `Strict` cookies. PoP does not undo a cookie exposure that already happened on the wire.

The lock-down is a **UA-vouched** set of properties about how the request was made; PoP is an **IDP-defined** assertion about the client. They are layered defenses, not substitutes — which makes the enforcement question non-trivial even in a PoP world.

**Open question.** Should the spec mechanically enforce that the lock-down shape survives the SW forwarding round-trip, or document it as an IDP-side contract and rely on each SW author to forward correctly?

If enforced, the probable options are:

- **Dedicated forwarding helper.** Introduce a purpose-built method on `IdentityRequestEvent` that invokes [=fetching=] directly on the UA's stored internal request, bypassing the public `Request` constructor entirely. The helper would also be the natural place to enforce the augmenting-header allowlist. Most FedCM-local solution, but adds new IDL and asks SW authors to learn a non-`fetch` forwarding API.
- **Fetch / Request spec changes.** Extend the public `Request` constructor and Fetch algorithm so the locked-down properties round-trip through the standard SW forwarding path for any caller, not just FedCM. Broadest reach, smallest FedCM-specific surface, but a much larger spec change touching general Fetch semantics.


## References

- [FedCM Specification](https://fedidcg.github.io/FedCM/)
- [Spec PR #815 — Enabling IDP Interception in FedCM Request](https://github.com/w3c-fedid/FedCM/pull/815)
- [GitHub Issue #80 — SW interception for FedCM](https://github.com/w3c-fedid/FedCM/issues/80)
- [WG meeting notes (23 Sept 2025) — agreement to pursue SW interception](https://github.com/w3c-fedid/meetings/blob/main/2025/2025-09-23-FedCM-notes.md)
- [Chromium service-worker-discuss thread — FedCM SW interception](https://groups.google.com/a/chromium.org/g/service-worker-discuss/c/t9d33x6l718)
- [Service Worker Specification](https://w3c.github.io/ServiceWorker/)
- [Fetch Specification](https://fetch.spec.whatwg.org/)
- [DPoP (RFC 9449)](https://www.rfc-editor.org/rfc/rfc9449)
- [HTTP Message Signatures (RFC 9421)](https://www.rfc-editor.org/rfc/rfc9421)
- [HTTP Integrity Fields (RFC 9530)](https://www.rfc-editor.org/rfc/rfc9530)
