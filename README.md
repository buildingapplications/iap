# `@biltme/iap`

Expo client for communicating with Bilt payments service.

`@biltme/iap` gives a React app a single provider plus a couple of hooks that
cover the full iOS in-app purchase lifecycle:

- Bootstrap the current user with the backend (authenticated or anonymous).
- Fetch App Store product metadata through `expo-iap` in production, or build
  mock product cards from configured product IDs in development.
- Drive native purchases and forward receipts to `POST /v1/iap/purchases/ingest`.
- Expose a backend-authoritative `entitlements` map to the UI.
- Restore purchases via `expo-iap` + `POST /v1/iap/restore`.
- Open the native Manage Subscriptions sheet.
- Transparently retry ingest calls that failed with transient errors, persisted
  across app restarts.
- Support purchase-before-signup flows via anonymous identities that are later
  linked to the authenticated user.

Only iOS is wired up today. The provider is safe to mount on Android — it will
bootstrap entitlements and skip native store calls. On web, purchases and
restore are unavailable but entitlement reads still work.

## Contents

- [Installation](#installation)
- [Quick start](#quick-start)
- [Auth modes](#auth-modes)
- [Configuration (`BiltIapConfig`)](#configuration-biltiapconfig)
- [`BiltIapProvider`](#biltiapprovider)
- [Hooks](#hooks)
  - [`useBiltIAP()`](#usebiltiap)
  - [`useEntitlement(code)`](#useentitlementcode)
- [Entitlements](#entitlements)
- [Products](#products)
- [Anonymous identities](#anonymous-identities)
- [Error handling (`BiltIapError`)](#error-handling-biltiaperror)
- [Offline retry queue](#offline-retry-queue)
- [Lifecycle behavior](#lifecycle-behavior)
- [Backend endpoints used](#backend-endpoints-used)
- [Metro / workspace resolution](#metro--workspace-resolution)

## Installation

```sh
npm install @biltme/iap
# or
bun add @biltme/iap
```

Install the native peer dependencies in your app:

- `react` >= 18
- `react-native` >= 0.72
- `expo-iap` >= 3
- `@react-native-async-storage/async-storage` >= 2

They are peers because both the host app and `@biltme/iap` must see the exact
same copy. Duplicating `react` or `react-native` produces runtime errors like
`Invalid hook call` or `Cannot read property 'useRef' of null`.

> `expo-iap` requires a custom dev build. Expo Go cannot load it.

## Quick start

```tsx
import React from 'react';
import { BiltIapProvider, useBiltIAP, useEntitlement } from '@biltme/iap';
import type { BiltIapConfig } from '@biltme/iap';

const config: BiltIapConfig = {
  tenantAppId: '11111111-1111-1111-1111-111111111111',
  getAccessToken: async () => auth.getToken(),
  productIds: [{ id: 'com.example.pro.monthly' }],
  onError: (err) =>
    console.warn('[BiltIAP]', {
      code: err.code,
      message: err.message,
      retryable: err.retryable,
      requestId: err.requestId,
      cause: err.cause,
    }),
};

export default function App() {
  return (
    <BiltIapProvider config={config}>
      <PaywallScreen />
    </BiltIapProvider>
  );
}

function PaywallScreen() {
  const { initialized, products, purchaseProduct, restorePurchases } = useBiltIAP();
  const pro = useEntitlement('pro');

  if (!initialized) return null;
  if (pro.active) return <ProFeatures />;

  return (
    <>
      {products.map((p) => (
        <Button
          key={p.id}
          title={`Buy ${p.title} — ${p.displayPrice}`}
          onPress={() => purchaseProduct(p.id)}
        />
      ))}
      <Button title="Restore Purchases" onPress={() => restorePurchases()} />
    </>
  );
}
```

> The `config` object is read once on mount. If it needs to change across
> renders, memoize it with `useMemo` to avoid accidentally re-reading on
> every render. The `getAccessToken` callback is re-invoked on every
> request, so token rotation works without re-creating the provider.

## Auth modes

Each tenant app on the `bilt-billing` backend is configured with one of two
auth providers. The SDK config you pass **must match the tenant's configured
provider**, otherwise every authenticated request fails with
`unauthorized`.

### 1. Supabase-auth tenant (`auth_provider = supabase`)

The backend verifies incoming requests by validating a Supabase JWT against
the tenant's configured `jwt_issuer` (which encodes the Supabase project
ref). You **must** provide `getAccessToken` and return the current Supabase
session's access token:

```tsx
import { supabase } from './supabaseClient';

const config: BiltIapConfig = {
  tenantAppId: '...',
  getAccessToken: async () => {
    const { data } = await supabase.auth.getSession();
    return data.session?.access_token ?? null;
  },
  productIds: [{ id: 'com.example.pro.monthly' }],
};
```

Behavior:

- When the user is signed in, the SDK sends `Authorization: Bearer <jwt>`.
  The backend derives the stable app user id from the JWT `sub` claim.
- When the user is signed out, `getAccessToken` should return `null` /
  `undefined`. The SDK falls back to an anonymous billing identity
  (enabled by default) so the user can still purchase before signing in.
- After the user signs in, call `linkAnonymousPurchasesToCurrentUser()` to
  merge anonymous purchases into the authenticated account. See
  [Anonymous identities](#anonymous-identities).

### 2. No-auth tenant (`auth_provider = none`)

The backend does not accept bearer tokens for this tenant. **Omit
`getAccessToken` entirely.** The SDK uses anonymous billing identities only:

```tsx
const config: BiltIapConfig = {
  tenantAppId: '...',
  productIds: [{ id: 'com.example.pro.monthly' }],
};
```

Behavior:

- The SDK generates a stable `anon:<uuid>`, persists it in `AsyncStorage`,
  and sends it as `X-Bilt-Anonymous-App-User-Id` on every request.
- All entitlements are scoped to that anonymous id. The same device keeps
  the same id across launches and purchases.
- `linkAnonymousPurchasesToCurrentUser()` is not applicable.

### Common mistakes

- **Providing `getAccessToken` on a no-auth tenant**: the token is sent but
  the backend rejects it (wrong provider). All requests fail.
- **Omitting `getAccessToken` on a Supabase tenant**: the SDK runs in
  anonymous-only mode, which is fine for signed-out users but means signed-in
  users never become authenticated billing principals — they keep buying as
  anonymous identities forever.
- **Sending a custom user id header**: the SDK deliberately does not
  accept or forward any client-controlled user id. Identity is always
  derived by the backend from the verified bearer or the anonymous header.

## Configuration (`BiltIapConfig`)

| Field                | Type                                           | Required | Description                                                                                                          |
| -------------------- | ---------------------------------------------- | -------- | -------------------------------------------------------------------------------------------------------------------- |
| `backendUrl`         | `string`                                       | no       | Base URL of the `bilt-billing` backend. Defaults to `https://billing.bilt.me`. Trailing slashes are stripped.        |
| `headers`            | `Record<string, string>`                       | no       | Extra headers to include on every backend request. Useful for local tunnels such as ngrok.                          |
| `tenantAppId`        | `string`                                       | yes      | Sent as `X-Bilt-Tenant-App-Id` on every request. Scopes the call to the correct tenant app.                          |
| `getAccessToken`     | `() => Promise<string \| null \| undefined>`   | see [Auth modes](#auth-modes) | Required for Supabase-auth tenants (return the Supabase session access token). Omit entirely for no-auth tenants. Return `null`/`undefined` when the user is signed out to fall back to anonymous billing. |
| `anonymousIdentity`  | `{ enabled?: boolean; storageKey?: string }`   | no       | Enabled by default. Generates a stable anonymous billing principal for purchase-before-signup flows.                 |
| `billingEnvironment` | `'mock' \| 'production'`                       | no       | Explicit runtime environment. If omitted, `__DEV__ === true` uses `mock`; otherwise `production`.                    |
| `productIds`         | `ConfiguredProduct[]`                          | yes      | Product config to fetch from the store. In mock mode, entries can also provide explicit `type`, `title`, `description`, `displayPrice`, `price`, and `currency`. |
| `onError`            | `(err: BiltIapError) => void`                  | no       | Called whenever the provider surfaces an error. Useful for Sentry / Datadog / structured logging.                    |

Pointing at a local or staging billing backend:

```tsx
const config: BiltIapConfig = {
  backendUrl: 'http://127.0.0.1:8099',
  tenantAppId: '11111111-1111-1111-1111-111111111111',
  getAccessToken: async () => auth.getToken(),
  productIds: [{ id: 'com.example.pro.monthly' }],
};
```

Using explicit product metadata for mock mode:

```tsx
const config: BiltIapConfig = {
  tenantAppId: '11111111-1111-1111-1111-111111111111',
  billingEnvironment: 'mock',
  productIds: [
    {
      id: 'com.buildingpp.subtrackr.aaa',
      type: 'subs',
      title: 'AAA',
      description: 'Mock subscription product',
    },
    { id: 'com.buildingpp.subtrackr.pro_monthly_2' },
  ],
};
```

This is especially useful when a subscription SKU does not contain words like
`monthly`, `annual`, or `yearly`. In production, `expo-iap` still provides the
authoritative store metadata.

## `BiltIapProvider`

```tsx
<BiltIapProvider config={config}>{children}</BiltIapProvider>
```

On mount the provider:

1. Resolves the billing principal: authenticated if `getAccessToken` returns
   a token, otherwise anonymous (if enabled).
2. Calls `GET /v1/iap/bootstrap` to obtain the `appAccountToken` and the
   initial `entitlements` map.
3. In mock mode, builds product cards from `productIds` and skips native
   StoreKit. When a `productIds` entry is an object, its explicit mock metadata
   is used instead of guessing from the SKU text.
4. In production on iOS, probes native store availability via
   `ExpoIap.initConnection()`. If the probe fails (web, Expo Go, missing
   native module) the provider still loads entitlements but disables
   purchase / restore.
5. When the store is available, calls `ExpoIap.fetchProducts({ skus, type: 'all' })`
   so mixed subscription + in-app catalogs load.
6. Loads the persisted retry queue from `AsyncStorage` and flushes it.
7. Subscribes to `ExpoIap.purchaseUpdatedListener` and
   `ExpoIap.purchaseErrorListener`.
8. Subscribes to `AppState` `"change"`. On foreground it re-fetches
   entitlements and flushes the retry queue.

On unmount it removes listeners, clears any pending flush timer, and calls
`ExpoIap.endConnection()` if a native connection was opened.

## Hooks

### `useBiltIAP()`

Returns `BiltIapState & BiltIapApi`. Must be called inside a `BiltIapProvider`,
otherwise it throws `useBiltIAP must be used inside <BiltIapProvider>`.

State:

| Field                    | Type                                  | Notes                                                                             |
| ------------------------ | ------------------------------------- | --------------------------------------------------------------------------------- |
| `initialized`            | `boolean`                             | `true` once bootstrap + store init have resolved.                                 |
| `loading`                | `boolean`                             | `true` until the initial bootstrap resolves (success or failure).                 |
| `principalType`          | `'authenticated' \| 'anonymous'`      | Current billing principal.                                                        |
| `currentAppUserId`       | `string \| undefined`                 | Stable backend user id for the active principal.                                  |
| `billingEnvironment`     | `'mock' \| 'production'`              | Runtime environment sent as `X-Bilt-Billing-Environment`.                         |
| `purchasing`             | `boolean`                             | `true` while a purchase flow is in progress.                                      |
| `restoring`              | `boolean`                             | `true` while `restorePurchases` is running.                                       |
| `entitlements`           | `EntitlementMap`                      | Backend-authoritative map keyed by entitlement code (e.g. `"pro"`).               |
| `appAccountToken`        | `string \| undefined`                 | Stable per-user token the backend tags transactions with. Set after bootstrap.    |
| `products`               | `StoreProduct[]`                      | App Store products for `config.productIds`. Empty on Android or if fetch failed.  |
| `storeAvailable`         | `boolean`                             | `true` when native StoreKit/Billing is usable. `false` on web, Expo Go, failures. |
| `storeUnavailableReason` | `string \| undefined`                 | Human-readable reason the store is unavailable, if known.                         |
| `lastError`              | `BiltIapError \| undefined`           | Last error the provider surfaced.                                                 |
| `pendingRetries`         | `number`                              | Number of ingest payloads sitting in the offline retry queue.                     |

API methods:

| Method                                                 | Description                                                                                                                                                                                                                     |
| ------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `purchaseProduct(productId: string): Promise<void>`    | Production: kicks off the native purchase sheet. Mock: sends synthetic proof through the normal ingest route. Throws if the provider has not initialized, the store is unavailable, or the product is not in `products`.        |
| `restorePurchases(): Promise<void>`                    | Production: calls `ExpoIap.restorePurchases()`, re-ingests every active purchase it finds, then calls `POST /v1/iap/restore`. Mock: calls backend restore directly. Updates `entitlements`.                                     |
| `linkAnonymousPurchasesToCurrentUser(): Promise<void>` | Merge the current anonymous purchase history into the signed-in user. See [Anonymous identities](#anonymous-identities). Throws if not anonymous or no access token is available.                                               |
| `refreshEntitlements(): Promise<void>`                 | Calls `GET /v1/iap/entitlements` and updates state. Never throws — errors are piped to `onError`.                                                                                                                               |
| `hasEntitlement(code: string): boolean`                | Shortcut for `entitlements[code]?.active === true`.                                                                                                                                                                             |
| `openManageSubscriptions(): Promise<void>`             | iOS: opens the native Manage Subscriptions sheet via `ExpoIap.deepLinkToSubscriptions`. Android: no-op without extra args.                                                                                                      |
| `flushRetryQueue(): Promise<void>`                     | Force-flush the retry queue now (e.g. when the app regains connectivity).                                                                                                                                                       |

### `useEntitlement(code)`

Convenience hook for simple gate UIs.

```tsx
const pro = useEntitlement('pro');

if (pro.loading) return <Spinner />;
if (pro.active) return <ProFeatures />;
return <Paywall />;
```

Returns:

| Field         | Type                                 |
| ------------- | ------------------------------------ |
| `active`      | `boolean`                            |
| `status`      | `EntitlementStatus \| undefined`     |
| `entitlement` | `BiltEntitlement \| undefined`       |
| `loading`     | `boolean` (true until `initialized`) |

## Entitlements

```ts
type EntitlementStatus =
  | 'active'
  | 'grace_period'
  | 'billing_retry'
  | 'expired'
  | 'revoked'
  | 'refunded'
  | 'purchased';

type BiltEntitlement = {
  active: boolean;
  status: EntitlementStatus;
  platform: 'IOS' | 'Android';
  productId?: string;
  currentPlanId?: string;
  expirationDate?: number;   // epoch ms
  isAutoRenewing?: boolean;
  gracePeriod?: boolean;
  billingRetry?: boolean;
  updatedAt?: number;        // epoch ms
};

type EntitlementMap = Record<string, BiltEntitlement>;
```

Entitlements are always authored by the backend. The client never derives
entitlement state from a local purchase — it only forwards receipts and
re-reads the map.

## Products

```ts
type StoreProduct = {
  id: string;
  title: string;
  description: string;
  displayPrice: string;      // localized, e.g. "$4.99"
  price: number;             // numeric price in `currency`
  currency: string;
  store: 'apple' | 'google' | 'unknown';
  type: 'subs' | 'in-app';
};
```

`type` is inferred from `expo-iap`'s `typeIOS`. `auto-renewable-subscription`
and `non-renewing-subscription` are treated as `'subs'`, everything
else as `'in-app'`. `purchaseProduct` uses this to pick between `type: 'subs'`
and `type: 'in-app'` when calling `ExpoIap.requestPurchase`.

In mock mode, `type` can be provided explicitly in `config.productIds` object
entries. If omitted, the SDK falls back to the legacy SKU-name heuristic.

## Anonymous identities

The provider supports purchase-before-signup. When `getAccessToken` returns
`null`/`undefined` and `anonymousIdentity.enabled` is not `false` (the
default), a stable anonymous billing principal is generated, stored in
`AsyncStorage`, and sent to the backend as `X-Bilt-Anonymous-App-User-Id`.
The user can buy and hold entitlements while signed out.

After the user signs in, call `linkAnonymousPurchasesToCurrentUser()` to
merge the anonymous history into the authenticated account:

```tsx
const { linkAnonymousPurchasesToCurrentUser } = useBiltIAP();

async function onSignInComplete() {
  try {
    await linkAnonymousPurchasesToCurrentUser();
  } catch (err) {
    // Already authenticated, no anonymous id, or missing access token
  }
}
```

Linking is also attempted automatically at the start of `purchaseProduct` and
`restorePurchases` when an access token becomes available, so in most apps
you do not need to call it explicitly.

Customize the storage key if you need to isolate multiple apps in the same
bundle:

```tsx
const config: BiltIapConfig = {
  // ...
  anonymousIdentity: { storageKey: '@myapp:bilt-anon-id' },
};
```

## Error handling (`BiltIapError`)

Every error surfaced by the provider is a `BiltIapError`.

```ts
class BiltIapError extends Error {
  readonly code: BiltIapErrorCode;
  readonly retryable: boolean;
  readonly requestId?: string;
}
```

Codes:

| Code                     | Source  | Meaning                                                |
| ------------------------ | ------- | ------------------------------------------------------ |
| `unauthorized`           | backend | Caller is not allowed.                                 |
| `invalid_request`        | backend | Validation failed.                                     |
| `ownership_mismatch`     | backend | Transaction belongs to another `appAccountToken`.      |
| `product_not_configured` | backend | Product id is not mapped in the tenant catalog.        |
| `store_unavailable`      | backend | Upstream App Store call failed.                        |
| `internal_error`         | backend | Fallback for unexpected backend failures.              |
| `billing_user_not_found` | backend | The backend has no record of this user.                |
| `notification_invalid`   | backend | Apple notification was rejected.                       |
| `not_initialized`        | client  | Provider has not finished init / token missing.        |
| `purchase_cancelled`     | client  | User dismissed the purchase sheet.                     |
| `purchase_pending`       | client  | Ask-to-buy / deferred purchase.                        |
| `purchase_failed`        | client  | Native purchase flow failed.                           |
| `network_error`          | client  | `fetch` rejected. Always `retryable: true`.            |
| `store_not_available`    | client  | Store connection never came up.                        |
| `product_not_found`      | client  | `purchaseProduct` called with an id not in `products`. |

Only `refreshEntitlements` swallows its own errors (into `lastError` /
`onError`). `purchaseProduct`, `restorePurchases`, and
`linkAnonymousPurchasesToCurrentUser` route errors to `onError` and also
rethrow so callers can show per-action UI.

## Offline retry queue

Ingest calls (`POST /v1/iap/purchases/ingest`) are critical — they turn a
real StoreKit receipt into an entitlement. If one fails with a
`retryable: true` error, the payload is enqueued in the retry queue and
persisted to `AsyncStorage` under the key `@biltme/iap:retry-queue`.

Retry behavior:

- Exponential backoff: `min(2s * 2^attempts, 5min)` with 50–100% jitter.
- Up to `8` attempts per payload. After that the item is dropped (dead
  letter) rather than poisoning the queue.
- Flushed on: enqueue, app foreground, and when `flushRetryQueue()` is
  called explicitly.
- Non-retryable errors (`invalid_request`, `ownership_mismatch`, etc.) drop
  the item immediately instead of retrying.

`pendingRetries` is exposed so the UI can surface a banner:

```tsx
{pendingRetries > 0 && (
  <Text>{pendingRetries} pending retries in queue</Text>
)}
```

## Lifecycle behavior

- **`purchase-updated` listener**: ingests the receipt, then calls
  `ExpoIap.finishTransaction({ purchase })` only if the backend returns
  `finishTransaction: true`. Pending (`ask-to-buy`) purchases are ignored
  until they resolve — the user does not get access yet.
- **`purchase-error` listener**: `user-cancelled` is silently dropped; all
  other codes surface via `onError`.
- **Foreground refresh**: when `AppState` flips to `"active"` and the
  provider has initialized, it re-fetches entitlements and flushes the
  retry queue. This is how server-side lifecycle events (renew, expire,
  refund) reach the client without an explicit pull.
- **Restore**: calls `ExpoIap.restorePurchases()` first (so the native
  receipt refresh happens), then `ExpoIap.getAvailablePurchases` with
  `onlyIncludeActiveItemsIOS: true`, re-ingests each unique transaction,
  then hits `POST /v1/iap/restore` to let the backend reconcile.
- **Mock runtime**: skips native store calls, builds mock product cards
  from `productIds`, and sends synthetic purchase proof through
  `POST /v1/iap/purchases/ingest`. Entitlements still come only from the
  backend. Mock product metadata can be supplied explicitly so the UI does not
  have to infer subscription-vs-in-app from the product id string.

## Backend endpoints used

All requests are JSON and carry these headers:

- `Content-Type: application/json`
- `X-Bilt-Tenant-App-Id: <config.tenantAppId>`
- `X-Bilt-Billing-Environment: mock | production`
- `Authorization: Bearer <await config.getAccessToken()>` when authenticated,
  or `X-Bilt-Anonymous-App-User-Id: <id>` when anonymous

The backend uses the bearer token or anonymous header to derive the stable
billing principal. `tenantAppId` is tenant routing only. Missing billing
environment headers default to `production`, but the SDK always sends one.

| Method | Path                       | When                                                  |
| ------ | -------------------------- | ----------------------------------------------------- |
| GET    | `/v1/iap/bootstrap`        | Mount.                                                |
| GET    | `/v1/iap/entitlements`     | Foreground, `refreshEntitlements()`.                  |
| POST   | `/v1/iap/purchases/ingest` | After every `purchase-updated` and during restore.    |
| POST   | `/v1/iap/restore`          | `restorePurchases()`.                                 |
| POST   | `/v1/iap/link`             | `linkAnonymousPurchasesToCurrentUser()`.              |

Responses follow the shape `{ ok: true, data: T }` on success and
`{ ok: false, error: { code, message, retryable?, requestId? } }` on
failure. The client reads `retryable` from the body when present and falls
back to "HTTP 5xx => retryable" otherwise.

## Metro / workspace resolution

When consuming `@biltme/iap` from a local workspace path inside an Expo app,
Metro must resolve `react`, `react-native`, and the native peers from the
**host app's** `node_modules`, not from `packages/iap/node_modules`.
Otherwise you will see:

- `Invalid hook call`
- `Cannot read property 'useRef' of null`

The fix is to add the package's path to Metro's `watchFolders` and set
`resolver.nodeModulesPaths` to the host app's `node_modules` only.
