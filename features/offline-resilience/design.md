# Offline resilience — Categories: design

*Feature slug: `offline-resilience` · Date: 2026-06-17 · Scope: Phase 1 + 2*

---

## 0. The problem (observed)

First launch, online → license-selection screen renders. User loses connectivity, taps
**PPL(A)** → the home screen shows the calm empty state **"Brak dostępnych kategorii"** with no
retry and no mention of the connection.

That message means *"this licence has no subjects"* — which is never true. It's a connectivity
failure wearing a content-empty costume, and it's a dead end: nothing on screen lets the user
recover.

### Why it happens — two independent gaps

**Gap A — an offline result is indistinguishable from a legitimately-empty one.**
`ObserveCategoriesUseCase` only routes a *thrown* exception to `Data.Error`:

```kotlin
try {
    val categories = categoryRepository.getCategories(license.id).filter { it.code != UNKNOWN }
    emit(Data.Content(CategoriesResult(license, language, categories)))   // empty list lands here
} catch (e: Exception) {
    e.rethrowIfCancellation(); emit(Data.Error(e))                        // only throws land here
}
```

Firestore's offline persistence resolves a collection `get()` **from the local cache and returns an
empty `QuerySnapshot` — it does not throw.** Cold first launch ⇒ empty cache ⇒ `[]` ⇒
`Data.Content(emptyList())` ⇒ the `categories.isEmpty()` branch in `CategoriesGrid` renders
`categories_text_empty`. The `Data.Error` path (which already shows `ErrorContent`: `CloudOff` icon +
"Spróbuj ponownie") is never reached.

The empty *content* state is itself fine — an empty list is a valid outcome and deserves its own
calm screen. The defect is that **the code can't tell an authoritative empty (the server returned
zero docs) from an offline empty (we never reached the server)** and paints both with the empty
state. The two are distinct outcomes and must render distinctly:

| outcome | state | screen |
|---|---|---|
| server returned 0 docs | **Content**(emptyList) | calm empty state (keep `categories_text_empty`) |
| couldn't reach server, nothing cached | **Error** (offline) | `ErrorContent` + retry |
| fetch threw | **Error** | `ErrorContent` + retry |

**Gap B — categories have no durable cache.**
Licences survive offline because `LicenseRepositoryImpl` persists the catalog to DataStore
(`LicenseCatalogCache`), with a *never-persist-empty* guard and a stale-while-revalidate boot.
Categories only have an in-memory `SuspendCache` in `CategoryRepositoryImpl` — empty on every cold
start. A returning user who once loaded categories online still gets nothing after process death
while offline. (`AppViewModel.prefetchStartContent` only runs when a licence is *already saved*; the
observed flow was a true first launch, so no prefetch ran.)

**Gap A is the bug the user hit. Gap B is why offline stays broken on later launches.**

---

## 1. Goals & non-goals

**Goals**
- Keep two distinct, correctly-rendered states: an **empty** content state (server says zero) and an
  **error** state (offline / failure) — never let one masquerade as the other.
- Make the error state recoverable (retry) and honest (you're offline, not "no data").
- Let a returning user keep a populated home screen offline (durable cache).
- Reuse existing patterns: the `Data` sealed type, `ErrorContent`/`CloudOff`, and the licence
  catalog's cache + SWR boot.
- Both platforms, from shared Compose — no per-platform UI.

**Non-goals (this phase)**
- Connectivity monitoring / auto-retry-on-reconnect / offline banner (deferred Phase 3).
- Offline support for the **exam/question** path (Firestore-backed, fails downstream offline) — the
  same gateway rule fixes it, tracked as a follow-up, not built here.
- Any data-model or Firestore-schema change.

---

## 2. Phase 1 — network vs non-network error distinction *(fixes the observed bug)*

**This phase ships on its own and fixes the reported bug. No caching.** Sort each read onto one axis
— *did we reach the backend or not?* — and make that call **at the data source**, where Firestore's
semantics already live, by throwing a domain exception. Everything above keeps its ordinary
`try/catch`; the connectivity signal never leaks into the domain.

- **Backend not reached** (offline with nothing to show, or a transport failure) → the source throws
  `BackendUnreachableException` → caught → **error** state with connection copy + retry.
- **Backend reached** → the source returns normally → a **content** state: the data, or — if the
  server authoritatively returned nothing — a calm **empty** state. (A *non-network* failure, e.g. a
  deserialization or permission error, propagates as itself → a *generic* error, distinct from the
  connection one.)

Why the source can decide: offline `get(Source.DEFAULT)` doesn't throw — it resolves from the local
cache and returns an empty `QuerySnapshot` with `metadata.isFromCache = true`. That flag is the
discriminator, and the source acts on it locally instead of forwarding it upward. (Verified present
in gitlive `firebase-firestore` 2.1.0 — `firestore.kt`.)

### 2.1 Data layer — throw on unreachable

The gateway is the single choke point for every Firestore read, so it owns the rule (framework
primitive — see `architecture.md` §1.2). Categories asserts `neverEmpty = true` because a categories
collection is never legitimately empty:

```kotlin
// FirestoreGateway.list(neverEmpty = true) { ... } — categories read
val snap = db.query().get()                  // Source.DEFAULT: server, cache-fallback offline
if (neverEmpty && snap.metadata.isFromCache && snap.documents.isEmpty()) throw BackendUnreachableException()
return snap.documents.map(map)
// + catch FirebaseFirestoreException(UNAVAILABLE / DEADLINE_EXCEEDED) → throw BackendUnreachableException(e)
```

For a one-shot `get(Source.DEFAULT)`, `isFromCache = true` means the server wasn't reached — online,
even a genuinely-empty collection returns `isFromCache = false`. So the rule does **not** throw on:
- `!isFromCache && empty` → server authoritatively returned zero (the *online* empty) → **return `[]`**
  → calm empty state.
- `isFromCache && non-empty` → offline but warm Firestore cache → **return the data** (free offline read).

> **Limitation:** `isFromCache` can't tell "offline, nothing cached" from "offline, genuinely empty"
> — the `neverEmpty` assertion resolves it (safe for categories, which are never empty). A can-be-empty
> collection would leave the flag off and show the empty state offline. See `architecture.md` §1.2.

`FirestoreCategoryDataSource` and `CategoryRepository` keep their existing `List<Category>`
signatures — nothing new flows through them.

### 2.2 Use case — unchanged

`ObserveCategoriesUseCase` needs **no change**: its existing `try { Content } catch { Error }` already
does the right thing — the source throws on unreachable → `Data.Error(BackendUnreachableException)`;
otherwise `Data.Content`, empty or not.

If we want the authoritative-empty alarm (a licence with zero subjects is a content/mapping problem),
it's now trivial and unambiguous — any empty list that *reaches* the use case is authoritative,
because the offline-empty already threw:

```kotlin
val categories = categoryRepository.getCategories(license.id).filter { it.code != CategoryCode.UNKNOWN }
if (categories.isEmpty()) Logger.w("Category catalog for ${license.id} resolved empty from the server")
emit(Data.Content(CategoriesResult(license, language, categories)))
```

### 2.3 Screen & ViewModel — keep the empty state, fix the error state

**Keep** the `content.categories.isEmpty()` branch in `CategoriesGrid` (`categories_text_empty`) — it
is now reached only for an authoritative empty. `CategoriesViewModel` widens `CategoriesUiState.Error`
to carry the `cause` (drives connection vs generic copy) and the licence header (so the title +
switcher stay usable on the error screen — the user can switch to a licence that *is* reachable/
cached); `CategoriesScreen` passes `state.cause` to `ErrorContent`. `retry()` already re-fires
`retryTrigger`.

### 2.4 Copy

Connection-aware `ErrorContent` (shared by 5 screens): when `cause is BackendUnreachableException` it
shows `CloudOff` + connection copy; otherwise `ErrorOutline` + the generic `common_text_generic_error`.
The connection strings are **app-level** (not categories-scoped, since `ErrorContent` is shared), both
locales:

| key | PL | EN |
|---|---|---|
| `error_offline_title` | Brak połączenia z internetem | No internet connection |
| `error_offline_message` | Sprawdź połączenie i spróbuj ponownie | Check your connection and try again |

Reuse `app_catalog_error_retry` ("Spróbuj ponownie" / "Retry") for the button (rename to
`common_action_retry` optional, since it's now app-wide). The empty state keeps `categories_text_empty`.

---

## 3. Phase 2 — durable category cache *(offline for returning users)*

Mirror the licence catalog pattern so categories boot offline the same way licences do.

### 3.1 Durable cache (framework `DataStoreDurableCache`)

Categories' catalog cache is one configuration of the framework's `DataStoreDurableCache`
(`architecture.md` §1.4): per-licence key `category_catalog_$licenseId`, JSON-serialized via a new
`CachedCategoryDto` (+ `toCacheDto`/`toDomain`) so the persisted form is decoupled from the Firestore
`CategoryDto` (as `CachedLicenseDto` is). Deserialize defensively — on failure return `null` and fall
through to the network.

### 3.2 `CategoryRepositoryImpl` — cache-first

Read the durable cache first; on a miss, hit the data source (which throws on unreachable) and persist
a non-empty result:

```kotlin
override suspend fun getCategories(licenseId: String): List<Category> =
    inMemory.getOrPut(licenseId) {                          // existing per-process SuspendCache
        catalogCache.read(licenseId)                        // durable hit → done (survives process death)
            ?: dataSource.getCategories(licenseId)          // miss → network; throws if offline-cold
                .also { if (it.isNotEmpty()) catalogCache.write(licenseId, it) }
    }
```

The durable read short-circuits **above** the data source, so the unreachable throw is only reached on
a durable miss — exactly when there's no offline copy to fall back on. The "don't persist garbage"
guard is free: an offline-empty can't reach the `write` because it threw, so we simply persist any
non-empty result. (`CachedRemoteResource`, `architecture.md` §1.5, packages this read-through; the
category repo is one configuration of it.) Background revalidate (`refresh`) wires in where the
licence catalog already does (see 3.3).

> **Interaction with Phase 1:** durable hit → Content (offline-capable). Durable miss + offline →
> data source throws → `Data.Error(BackendUnreachableException)` → recoverable screen. Durable miss +
> server-empty → `[]` → calm empty state. The three outcomes stay distinct with or without Phase 2 —
> Phase 2 only adds the durable-hit row.

### 3.3 Stale-while-revalidate boot

`AppViewModel` already prefetches categories for a saved licence (`prefetchStartContent`) and
revalidates the licence catalog in the background (`refreshCatalogInBackground`). Extend the same
choreography to categories: serve the cached categories immediately, kick a detached refresh. For a
licence the user *switches to*, the first `getCategories` call populates the durable cache for next
time. No new screen state needed — the existing Loading→Content/Error states cover it.

### 3.4 DI

`composeApp` Koin module: provide the `DataStoreDurableCache<List<Category>>` (same
`DataStore<Preferences>` the licence cache uses) and pass it into `CategoryRepositoryImpl`'s
constructor.

> ⚠️ Per project memory: `singleOf` ignores defaulted constructor params and host/VM tests bypass the
> Koin graph. Bind `CategoryRepositoryImpl` with an explicit lambda that passes the new cache, and
> **launch the app on the emulator** to confirm the graph resolves before calling this done.

---

## 4. UX states (categories screen, after Phase 1 + 2)

| Situation | State | Render |
|---|---|---|
| Categories present (network, Firestore cache, or durable cache) | `Content` (non-empty) | Grid (unchanged) |
| Fetching, nothing cached yet | `Loading` | Shimmer grid (unchanged) |
| Server authoritatively returned 0 docs | `Content` (empty) | Calm empty state — `categories_text_empty` (unchanged) |
| Offline, nothing cached → source throws | `Error` (`BackendUnreachableException`) | `ErrorContent`: `CloudOff` + connection copy + retry |
| Non-network failure (e.g. deserialization) | `Error` (other throwable) | `ErrorContent`: `ErrorOutline` + generic copy + retry |
| Retry tapped, network back | `Loading` → `Content` | Shimmer → grid |
| Retry tapped, still offline | `Error` | Same error state (idempotent) |

The licence header (title + switcher) stays visible on the error screen because
`CategoriesUiState.Error` carries `selectedLicense`/`availableLicenses` (§2.3) — so the user can
switch to a licence whose catalog *is* reachable instead of being stuck.

---

## 5. File-by-file

**Phase 1 — network vs non-network distinction**
*Framework primitives (new, shared) — `architecture.md` §1 Phase-1 set:* `BackendUnreachableException`,
the gateway throw-rule, connection-aware `ErrorContent`. Categories-specific wiring:
- `data/source/FirestoreGateway.kt` — throw `BackendUnreachableException` when a read is
  `isFromCache && empty`, and translate `UNAVAILABLE`/`DEADLINE_EXCEEDED` to it. Applies to every domain.
- `data/source/FirestoreCategoryDataSource.kt` + `domain/repository/CategoryRepository.kt` —
  **unchanged signatures** (`List<Category>`); nothing new threads through.
- `domain/usecase/category/ObserveCategoriesUseCase.kt` — **unchanged**; *(optional)* `Logger.w` on
  authoritative empty.
- `ui/categories/CategoriesViewModel.kt` — `CategoriesUiState.Error` carries `cause: Throwable` + the
  licence header; `toUiState`'s `Data.Error` branch passes them.
- `ui/categories/CategoriesScreen.kt` — pass `state.cause` to `ErrorContent`; **keep** the empty-list
  branch.
- `ui/components/CenteredContent.kt` — `ErrorContent(cause: Throwable? = null, …)` picks connection vs
  generic copy/icon off `is BackendUnreachableException`.
- `composeResources/values*/strings.xml` — add app-level `error_offline_title`/`error_offline_message`
  (both locales); `categories_text_empty` stays.

**Phase 2 — durable cache & cache-first**
*Framework primitives (new, shared) — `architecture.md` §1 Phase-2 set:* `DurableCache` +
`DataStoreDurableCache`, `CachedRemoteResource`. Categories-specific wiring:
- `data/source/dto/CachedCategoryDto.kt` — **new** DTO + mappers (decouple persisted form from
  Firestore `CategoryDto`).
- provide `DataStoreDurableCache<List<Category>>` (key `category_catalog_$licenseId`).
- `data/repository/CategoryRepositoryImpl.kt` — durable-cache-first read (above the data source);
  persist non-empty results; add the `refresh` revalidate path.
- DI module — bind the cache into the repo (explicit lambda, not `singleOf`).
- `ui/app/AppViewModel.kt` — extend SWR/prefetch to categories (completes the offline boot).

---

## 6. Testing

- **Gateway throw-rule (the core distinction, tested once for all domains):**
  `neverEmpty && isFromCache && empty` ⇒ throws `BackendUnreachableException`;
  `!neverEmpty && isFromCache && empty` ⇒ returns `[]`; `!isFromCache && empty` ⇒ returns `[]`;
  `isFromCache && non-empty` ⇒ returns data; `UNAVAILABLE`/`DEADLINE_EXCEEDED` ⇒ throws
  `BackendUnreachableException` (original kept as `cause`); other Firestore errors propagate unchanged.
- **Use case:** source throws ⇒ `Data.Error`; returns `[]` ⇒ `Data.Content(emptyList)` (+ `Logger.w`);
  non-empty ⇒ `Data.Content`; cancellation still rethrows.
- **Repository (Phase 2):** durable hit short-circuits the data source; miss → fetch → persists
  non-empty; offline-cold miss → throw propagates, **nothing persisted**; corrupt cache ⇒ `read`
  returns null → falls through to fetch.
- **ViewModel:** `Data.Error → CategoriesUiState.Error(cause, header)`; `retry()` re-fires the fetch;
  licence header survives the error state.
- **Manual / device (required, Koin graph):** airplane-mode first launch → connection error + retry;
  reconnect + retry → grid; relaunch offline after one online load → grid from durable cache.

Per the kmp-unit-testing skill: `commonTest`, fakes over mocks, no real DataStore/Firestore in unit
tests (fake the `CategoryDataSource` and the durable cache; for the gateway rule, fake the snapshot's
`isFromCache`/`documents`).

---

## 7. Follow-ups (not this phase)

- **Exam/question offline.** `QuestionRepository`/exam start is Firestore-backed and goes through the
  same gateway — so the throw-rule fixes its latent offline bug for free once it reads through the
  gateway helper; decide separately whether to cache question payloads (larger).
- **Phase 3 — connectivity awareness.** `expect/actual ConnectivityMonitor` (Android
  `ConnectivityManager`, iOS `NWPathMonitor`) → `Flow<Boolean>`: tailored copy, auto-retry on
  reconnect, "showing saved data" banner over stale cache.
