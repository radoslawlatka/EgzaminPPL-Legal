# Offline resilience — framework architecture

*Feature slug: `offline-resilience` · Date: 2026-06-17 · Architect output*
*Generalizes the Categories design (`./design.md`) into reusable primitives.*

---

## 0. Scope

The offline-resilience solution is a set of shared framework primitives that every Firestore-backed
read adopts. Four domains use the same read pipeline:

| Domain | Data source | In-mem cache | Durable cache | Result type | UI state enum |
|---|---|---|---|---|---|
| Licences | `FirestoreLicenseDataSource` | `SuspendCache` | `LicenseCatalogCache` | `List<License>` | `AppUiState` |
| Categories | `FirestoreCategoryDataSource` | `SuspendCache` | — | `List<Category>` | `CategoriesUiState` |
| Questions | `FirestoreQuestionDataSource` | `SuspendCache` | — | `List<Question>` | (exam states) |
| Explanations | `FirestoreExplanationDataSource` | `SuspendCache` | — | `Explanation?` | `ExplanationUiState` |

**Principle:** the gateway is the single choke point every remote read passes through. Reachability is
decided there, once, and surfaced as a thrown exception, so the connectivity detail stays in the data
layer and domain code never learns that Firestore (or a cache) exists. Everything above uses
`try/catch` and a few thin helpers.

### 0.1 Two shippable phases

The framework lands — and each adopting feature is migrated — in two independent phases. **Phase 1
fixes the reported bug and ships alone; Phase 2 is purely additive.**

| Phase | Capability | Primitives | What a feature gets |
|---|---|---|---|
| **1 — network vs non-network error distinction** | tell "backend unreachable" apart from content, server-empty, and other errors | §1.1 `BackendUnreachableException` · §1.2 gateway throw-rule · §1.3 `flowData` · §1.6 `DataStateView`/`ErrorContent` | offline → connection error + retry; real empty → empty state; other failure → generic error |
| **2 — durable cache & cache-first** | keep populated screens offline across process death | §1.4 `DurableCache`/`DataStoreDurableCache` · §1.5 `CachedRemoteResource` | read-through (in-mem → disk → network), SWR boot |

Phase 1 needs no cache: Firestore's own offline read already sets `isFromCache`, so the gateway can
make the call on its own. Phase 2 layers caching *above* the gateway — a durable hit returns before
the gateway is even reached — so nothing in Phase 1 changes.

---

## 1. The primitives (all in shared `commonMain`)

Six small pieces across the three layers, tagged **[P1]** / **[P2]** by phase (see §0.1). Nothing here
is feature-specific.

### 1.1 [P1] `BackendUnreachableException` *(domain/common)*

The one marker the framework recognizes — the backend was not reached. Every other failure stays its
natural exception inside `Data.Error` (which already carries any `Throwable`).

```kotlin
/**
 * The backend could not be reached: a cache-only empty (offline with nothing stored), or a
 * transport-level failure (Firestore UNAVAILABLE / DEADLINE_EXCEEDED). NOT a claim that the device is
 * in airplane mode — the server may simply be unreachable — so callers phrase it as a connection
 * problem, not a fact about the user. Carries the original cause when there was one (null for the
 * synthesized cache-empty case).
 */
class BackendUnreachableException(cause: Throwable? = null) : Exception(cause)
```

The UI special-cases exactly one type (`is BackendUnreachableException`) and treats the rest
generically — no mapping, no wrapping, no type erasure.

### 1.2 [P1] Gateway throw-rule — decide reachability at the source *(data/source)*

`FirestoreGateway` is the only place that touches Firestore types, so it owns the decision. The key
fact: for a one-shot `get(Source.DEFAULT)`, `isFromCache = true` happens **only when the server
couldn't be reached** — online, `get()` round-trips and returns `isFromCache = false` even for a
genuinely-empty collection. The gateway acts on that locally and **throws**, instead of forwarding
provenance upward. Reads keep their plain return types (`List<T>`, `T?`):

```kotlin
suspend fun <T> list(
    map: (DocumentSnapshot) -> T,
    neverEmpty: Boolean,                                          // required — no default: every call site decides
    query: FirebaseFirestore.() -> Query,
): List<T> = firestoreCatching {
    awaitDelayIfEnabled()
    val snap = db.query().get()                                   // Source.DEFAULT — cache-fallback offline
    debug.recordFirestoreReads(snap.documents.size)
    // Only when the caller asserts the collection is never empty does an empty cache read mean offline.
    if (neverEmpty && snap.metadata.isFromCache && snap.documents.isEmpty()) throw BackendUnreachableException()
    snap.documents.map(map)
}

suspend fun <T> docOrNull(
    map: (DocumentSnapshot) -> T?,
    ref: FirebaseFirestore.() -> DocumentReference,
): T? = firestoreCatching {
    awaitDelayIfEnabled()
    val snap = db.ref().get()
    debug.recordFirestoreReads(1)
    if (snap.metadata.isFromCache && !snap.exists) throw BackendUnreachableException()
    snap.data<…>()?.let(map)
}

// the boundary that keeps Firestore types out of everything above
private inline fun <T> firestoreCatching(block: () -> T): T =
    try { block() }
    catch (e: FirebaseFirestoreException) {
        // Only transport failures are translated; every other error propagates unchanged.
        if (e.code == UNAVAILABLE || e.code == DEADLINE_EXCEEDED) throw BackendUnreachableException(e)
        throw e
    }
```

What it deliberately does **not** throw on — so the empty-vs-error distinction is preserved:
- `!isFromCache && empty` → server authoritatively returned nothing → **return `[]` / `null`** → calm
  empty / "no content" state. (This is the *online* genuinely-empty collection — no false alarm.)
- `isFromCache && non-empty` → offline but warm Firestore cache → **return the data** (free offline read).
- `list(neverEmpty = false)` → empty is always returned as `[]`, never thrown.

**Limitation — offline + a legitimately-empty collection.** `isFromCache` cannot distinguish "offline,
nothing cached" from "offline, the collection really is empty (and we cached that empty)" — both read
as `isFromCache && empty`. The `neverEmpty` flag resolves this by *assertion*: a caller that knows the
collection is never legitimately empty (`Categories`, `Licences`) opts in and gets the offline error;
a can-be-empty collection leaves it off and shows the empty state offline (transport-error translation
still throws on `UNAVAILABLE`/`DEADLINE`). Phase 3's connectivity monitor can later disambiguate this
case precisely. `docOrNull` throws on `isFromCache && !exists` unconditionally because for a single
document a *legitimate* empty is a present-but-empty doc (`exists = true` → returned); an **absent**
doc fetched offline genuinely warrants retry (this is exactly the explanation latent bug), and the
key-map gate already prevents fetching docs we expect to be absent.

*(`metadata.isFromCache`, `Source`, and `FirebaseFirestoreException.code` are all confirmed present in
gitlive `firebase-firestore` 2.1.0 — `firestore.kt`.)*

### 1.3 [P1] `flowData` — kill the use-case boilerplate *(domain/common)*

With reachability decided at the gateway, the observing use case is just Loading + the (now ordinary)
fetch. This wraps the copy-pasted `emit(Loading); try { Content } catch { Error }`:

```kotlin
fun <T> flowData(fetch: suspend () -> T): Flow<Data<T>> = flow {
    emit(Data.Loading)
    emit(Data.Content(fetch()))                                  // empty is just Content(empty)
}.catch { e -> e.rethrowIfCancellation(); emit(Data.Error(e)) }  // BackendUnreachable or anything else

// composes with provider streams for catalogs that depend on licence/language:
fun <K, T> Flow<K>.flatMapToData(fetch: suspend (K) -> T): Flow<Data<T>> =
    transformLatest { key -> emit(Data.Loading); emit(Data.Content(fetch(key))) }
        .catch { e -> e.rethrowIfCancellation(); emit(Data.Error(e)) }
```

No `isEmpty`, no classifier — empty-is-content falls out for free, and the gateway already routed
offline to a throw.

### 1.4 [P2] Durable cache — generalize `LicenseCatalogCache` *(data/source)*

```kotlin
interface DurableCache<K, V> {
    suspend fun read(key: K): V?                      // null = nothing stored for this key
    suspend fun write(key: K, value: V)
}

class DataStoreDurableCache<V>(                       // one impl, JSON via kotlinx.serialization
    private val dataStore: DataStore<Preferences>,
    private val serializer: KSerializer<V>,
    private val keyPrefix: String,
) : DurableCache<String, V>
```

`LicenseCatalogCache` becomes one configuration of this (`keyPrefix = "license_catalog"`). Categories'
Phase-2 cache is another (`"category_catalog_$licenseId"`). No content predicate here — the cache
stores whatever it's given (see §1.5 for why that's safe now).

### 1.5 [P2] `CachedRemoteResource` — read-through orchestration *(data/repository)*

The repository pattern (`SuspendCache` → durable → network) extracted so a repo becomes a declaration,
not a re-implementation:

```kotlin
class CachedRemoteResource<K, V>(
    private val durable: DurableCache<K, V>?,         // null = remote-only (no offline persistence)
    private val fetch: suspend (K) -> V,              // throws BackendUnreachableException when offline-cold
) {
    private val inMemory = SuspendCache<K, V>()
    suspend fun get(key: K): V = inMemory.getOrPut(key) {
        durable?.read(key)                            // durable hit (survives process death) → done
            ?: fetch(key).also { durable?.write(key, it) }   // miss → network, then persist
    }
    suspend fun refresh(key: K): V = fetch(key).also { durable?.write(key, it) }   // SWR revalidate
}
```

The durable read short-circuits **above** the gateway, so the network fetch (and its offline throw) is
only reached on a durable miss. An offline-empty throws and never reaches `write`, so the cache only
stores server-confirmed results — no persist predicate is needed.

### 1.6 [P1] `DataStateView` + connection-aware `ErrorContent` — uniform rendering *(ui/components)*

One composable resolves the 4-way render for any `Data<T>`, so features stop writing bespoke state
enums and bespoke loading/error/empty UI. Emptiness *as a rendering choice* stays in the presentation
layer via an `isEmpty` predicate (the data layer only decided reachable-or-not):

```kotlin
@Composable
fun <T> DataStateView(
    state: Data<T>,
    onRetry: () -> Unit,
    isEmpty: (T) -> Boolean = { false },
    loading: @Composable () -> Unit = { LoadingContent() },
    empty: @Composable () -> Unit = {},
    error: @Composable (Throwable) -> Unit = { ErrorContent(it, onRetry) },
    content: @Composable (T) -> Unit,
) = when (state) {
    Data.Loading   -> loading()
    is Data.Error  -> error(state.cause)
    is Data.Content -> if (isEmpty(state.value)) empty() else content(state.value)
}
```

`ErrorContent` takes the raw `Throwable` and special-cases the one type the framework recognizes:

```kotlin
fun ErrorContent(cause: Throwable?, onRetry: () -> Unit) =
    if (cause is BackendUnreachableException) ErrorContentBody(CloudOff, connectionCopy, onRetry)
    else ErrorContentBody(ErrorOutline, genericCopy /* common_text_generic_error */, onRetry)
```

Connection-aware copy, for free, everywhere — and the one `is` check is the *only* place any layer
above the gateway names the unreachable case.

---

## 2. Adoption walkthrough A — Categories

The `./design.md` plan re-expressed on the primitives — same behaviour, almost no bespoke code. The
two phases adopt independently:

**Phase 1 — distinction (ships alone, fixes the bug):**
- **Data source:** `firestore.list({ it.data<CategoryDto>().toDomain(it.id) }, neverEmpty = true) { collection(...) }` →
  `List<Category>`. The `neverEmpty = true` asserts a categories collection is never legitimately
  empty, so the gateway throws `BackendUnreachableException` on an offline-cold read (§1.2).
- **Repository:** in-memory `SuspendCache` only; unchanged `List<Category>` signature.
- **Use case:** unchanged — the existing `try { Content } catch { Error }` already classifies; or thin
  it to `flatMapToData { repo.getCategories(it.licenseId).filter { … } }`.
- **UI:** `DataStateView(state, onRetry = vm::retry, isEmpty = { it.categories.isEmpty() }, empty = { /* categories_text_empty */ }) { grid(it) }`.

**Phase 2 — durable cache (additive):**
- **Repository:** wrap the bare `SuspendCache` in `CachedRemoteResource(durable = categoryCatalogCache, fetch = { dataSource.getCategories(it) })`.
  Use case and UI are untouched.

`CategoriesUiState` shrinks to the screen-specific extras (selected licence, continue session, tile
progress); the Loading/Content/Error scaffolding moves to `DataStateView` after Phase 1.

## 3. Adoption walkthrough B — Explanations *(the second domain that proves it's generic)*

Explanations are a **single nullable document**, not a list — exercising the `docOrNull` path:

- **Data source:** `firestore.docOrNull({ it.toDomain(key) }) { collection(EXPLANATIONS)...document(lang) }`
  → `Explanation?`. The current `?: Explanation(emptyList())` fallback goes away — an absent doc is a
  clean `null`. (Gateway throws on `isFromCache && !exists` — offline with nothing cached.)
- **Use case:** `GetExplanationUseCase` returns `flowData { repo.getExplanation(key) }` after the
  feature-flag/key-map gate. **The win:** today an offline explanation fetch returns empty blocks →
  renders nothing, indistinguishable from "this question has no explanation." Now offline →
  `Data.Error(BackendUnreachableException)` → the existing retry affordance fires; an absent doc →
  `null` / a present-but-empty doc → renders nothing. Same bug, same fix, zero new design.
- **UI:** `ExplanationUiState` (Loading/Content/Error/Empty) collapses into `DataStateView` with
  `isEmpty = { it == null || it.blocks.isEmpty() }`, `empty = {}` (the "no card" case), and
  `error = { ExplanationError(onRetry) }` (its inline variant — slots are overridable, so a domain
  keeps its bespoke look where it wants one).

## 4. New feature, out of the box

A brand-new remote-backed screen adopts the whole stack in ~5 lines and inherits offline handling,
empty handling, caching, and retry copy without thinking about any of it:

```kotlin
// data
val resource = CachedRemoteResource(durable = null) { firestore.list(::map, neverEmpty = false) { collection("airports") } }
// domain
fun observeAirports() = flowData { resource.get(Unit) }
// ui
DataStateView(state, onRetry = vm::retry, isEmpty = List<Airport>::isEmpty,
    empty = { EmptyContent(stringResource(...)) }) { AirportList(it) }
```

---

## 5. Migration & phasing

Backward compatible — primitives land first and domains migrate one at a time; `Data` keeps its three
cases (see §7), so nothing breaks mid-migration. Each row below is an independent, shippable change.

**Phase 1 — network vs non-network error distinction**
1. **P1 primitives** — `BackendUnreachableException`, the gateway `list`/`docOrNull` throw-rule +
   `firestoreCatching`, `flowData`, connection-aware `ErrorContent`, `DataStateView` (§0.1 P1 set).
   Tested in isolation; no behaviour change until a domain adopts them.
2. **Categories P1** — route its read through the gateway helper. **Delivers the reported-bug fix**
   (`./design.md` Phase 1) with no caching.
3. **Explanations P1** — migrate (fixes its latent offline bug as a side effect).
4. **Questions P1** — route the exam/question read through the gateway helper.
5. Retire each domain's Loading/Error enum scaffolding as its screen moves to `DataStateView`.

**Phase 2 — durable cache & cache-first** *(only after the domain's P1 has landed)*
6. **P2 primitives** — `DurableCache`/`DataStoreDurableCache`, `CachedRemoteResource` (§0.1 P2 set).
7. **Categories P2** — back the repo with `CachedRemoteResource` + its catalog cache (`./design.md`
   Phase 2); extend `AppViewModel` SWR. Use case and UI unchanged.
8. **Questions P2** — decide separately whether the payload is worth a durable cache.
9. **Licences** — refold the existing `LicenseCatalogCache` into `DataStoreDurableCache` /
   `CachedRemoteResource` last; it already works offline, so this is pure consolidation (and the
   `if (isNotEmpty())` persist guard drops out — see §1.5).

## 6. Testing

- **Gateway throw-rule (the core distinction, tested once for all domains):**
  `neverEmpty && isFromCache && empty` ⇒ throws `BackendUnreachableException`;
  `!neverEmpty && isFromCache && empty` ⇒ returns `[]`; `!isFromCache && empty` ⇒ returns `[]` / `null`;
  `isFromCache && non-empty` ⇒ returns data; `docOrNull` `isFromCache && !exists` ⇒ throws;
  `UNAVAILABLE`/`DEADLINE_EXCEEDED` ⇒ throws `BackendUnreachableException` (original kept as `cause`);
  every other exception propagates unchanged. Fake the snapshot's `isFromCache` / `documents` / `exists`.
- **`flowData`** — fetch returns ⇒ `Loading` then `Content`; fetch throws ⇒ `Loading` then `Error`
  (cause preserved); cancellation rethrows.
- **`CachedRemoteResource`** — durable hit short-circuits the fetch; miss → fetch → persists; an
  offline-cold fetch *throws* (so nothing is persisted); in-memory dedupe.
- **Per domain** — only the screen's `isEmpty` wiring needs a test; the pipeline is already proven.
- **Device (Koin):** bind resources/caches with explicit lambdas (not `singleOf` — defaulted ctor
  params) and launch on the emulator, per project convention.

## 7. Decisions

- Reachability is decided at the gateway and surfaced as a thrown `BackendUnreachableException`;
  connectivity provenance never crosses into the domain.
- `Data<T>` stays three-state. Emptiness is a rendering choice, decided in the UI via `isEmpty` at
  `DataStateView`; there is no `Data.Empty`.
- `list`'s `neverEmpty` is a required parameter with no default — every call site states whether its
  collection can be legitimately empty (categories/licences `true`, can-be-empty collections `false`).
- `BackendUnreachableException` is the single failure type the framework recognizes; every other
  throwable propagates unwrapped inside `Data.Error`.
- `BackendUnreachableException` is named for the observation (backend unreachable), not the device;
  the connection-oriented copy lives in the UI.
- The cache has no persist predicate; only server-confirmed results reach `write`.
- Phase 3 (`expect/actual ConnectivityMonitor` for auto-retry-on-reconnect and a stale-data banner)
  layers on top of `BackendUnreachableException` without reworking these primitives.
