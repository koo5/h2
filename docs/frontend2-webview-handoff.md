# Web UI inside frontend2 (CMP) — handoff and TODO

Status: plan only; **no integration code has been implemented or tested**.
Written: 2026-11-22. Source snapshot: 2026-10-26.

## Source of truth / repo warning

This document was prepared from read-only inspection of **koo5/hillview**, branch **dev42026-10-21_20-12-32_audit**, head **1020df5290ce67938a90b5e9f91ce70f9f847047** (2026-10-26). Links to particular files below are pinned to that SHA.

**H2 is stale.** At the time of inspection its master branch is only a README, and its terrain-depth-panoramas branch is at **c6bb79fc18633b78f5e5e61cdcd720b186886a32** (July 2026), without frontend2. GitHub confirmed that this terrain branch commit is an ancestor of the Hillview source head (589 commits ahead, zero behind). A normal git fetch + fast-forward is possible from a real Git checkout, but the available GitHub connector cannot transfer Git objects from a different, independent repository; the October Hillview commit is not present in H2's object database. **Do not implement against H2's old tree. Sync H2 first, then cherry-pick this document.**

A non-destructive example on a machine with git push credentials:

~~~sh
git clone git@github.com:koo5/h2.git
cd h2
git remote add hillview git@github.com:koo5/hillview.git
git fetch origin terrain-depth-panoramas
git switch -c sync/hillview-2026-10-26 origin/terrain-depth-panoramas
git fetch hillview dev42026-10-21_20-12-32_audit
git merge --ff-only FETCH_HEAD
git push -u origin sync/hillview-2026-10-26
~~~

This preserves master and old refs. The source head can advance later; recheck it before syncing. H2's master is an unrelated README-only history; don't force-push or reset it as part of this proof of concept.

## Product intent and constraints

- **Keep maintaining the existing SvelteKit website** and its routes. No independent Svelte fork/copy under frontend2.
- Eventually replace the Android Tauri app with the CMP app; tiny legacy Android user base means migration of its local preferences/tokens/queues is not a priority.
- Keep the **native map**, capture, sensors, native upload machinery, and the authoritative map/location/bearing state in KMP/CMP; the Android map engine can change later, independently of this WebView work. The **website map engine is not decided**.
- Reuse simpler web UIs: **/bestof, /users, /users/[id], /activity**, likely **/about**, perhaps **/photos** eventually, and **photo details + OpenSeadragon zoom**.
- **Do not bring along the Tauri runtime or Tauri plugin framework.** Introduce a narrow third platform implementation beside browser and Tauri only where the reused web code needs native capabilities.
- Do not recreate all old routes, all old platform commands, or old Android storage migration.

## Findings from the actual source

### Web app

- [frontend/svelte.config.js](https://github.com/koo5/hillview/blob/1020df5290ce67938a90b5e9f91ce70f9f847047/frontend/svelte.config.js): adapter-node for ADAPTER=node, otherwise adapter-static with index.html fallback. Thus a *single maintained Svelte source tree already supports website + static SPA output*. No route source duplication needed.
- [frontend/src/routes/+layout.svelte](https://github.com/koo5/hillview/blob/1020df5290ce67938a90b5e9f91ce70f9f847047/frontend/src/routes/%2Blayout.svelte): app-wide initialisation, auth checks, Tauri imports, deep links, notifications, navigation and **global ZoomView overlay**. A web-embedded mode needs deliberate gates here, not just a WebView that points to /bestof.
- [frontend/src/routes/+layout.ts](https://github.com/koo5/hillview/blob/1020df5290ce67938a90b5e9f91ce70f9f847047/frontend/src/routes/%2Blayout.ts) specifies ssr=false and prerender=true in the static configuration; the web build has .web route/load variants. /bestof and /activity use SSR-backed initial loads on the website and client fetching for the static/SPA case. Test **both** production web and embedded modes after modifying them.
- [frontend/src/routes/users/+page.svelte](https://github.com/koo5/hillview/blob/1020df5290ce67938a90b5e9f91ce70f9f847047/frontend/src/routes/users/%2Bpage.svelte): straightforward client-side /users/ API fetch; ideal first proof of concept alongside /bestof.
- [frontend/src/routes/bestof/+page.svelte](https://github.com/koo5/hillview/blob/1020df5290ce67938a90b5e9f91ce70f9f847047/frontend/src/routes/bestof/%2Bpage.svelte) and [activity/+page.svelte](https://github.com/koo5/hillview/blob/1020df5290ce67938a90b5e9f91ce70f9f847047/frontend/src/routes/activity/%2Bpage.svelte): both use createSsrBackedLoad and /api calls. Verify paging, refresh and auth change, not only first render.
- [frontend/src/lib/tokenManagerFactory.ts](https://github.com/koo5/hillview/blob/1020df5290ce67938a90b5e9f91ce70f9f847047/frontend/src/lib/tokenManagerFactory.ts) selects Tauri AndroidTokenManager vs browser WebTokenManager. [http.ts](https://github.com/koo5/hillview/blob/1020df5290ce67938a90b5e9f91ce70f9f847047/frontend/src/lib/http.ts) calls its getValidToken(), attaches Bearer headers and retries on 401. This is the natural place for **CmpTokenManager**, not a fake full Tauri IPC implementation.
- [frontend/src/lib/navigation.svelte.ts](https://github.com/koo5/hillview/blob/1020df5290ce67938a90b5e9f91ce70f9f847047/frontend/src/lib/navigation.svelte.ts) centralises many internal goto/back operations, but components also use normal anchors. [PhotoItem.svelte](https://github.com/koo5/hillview/blob/1020df5290ce67938a90b5e9f91ce70f9f847047/frontend/src/lib/components/PhotoItem.svelte) sends thumbnail clicks to a constructed map URL and title clicks to /photo/[uid]. These need separate WebView navigation rules.
- [frontend/src/routes/photos/+page.svelte](https://github.com/koo5/hillview/blob/1020df5290ce67938a90b5e9f91ce70f9f847047/frontend/src/routes/photos/%2Bpage.svelte) is **not** a simple server-side My Photos list: it also imports PhotoImport, PhotoUpload, UploadSettings, Kotlin photo-status sync, local/device information and Tauri gates. Defer until a server-only embedded UX is separated.
- [frontend/src/lib/components/Gallery.svelte](https://github.com/koo5/hillview/blob/1020df5290ce67938a90b5e9f91ce70f9f847047/frontend/src/lib/components/Gallery.svelte) is the *directional map-state viewer*, using photoInFront/photosInRange and updating bearing. It should **not** be embedded wholesale alongside the existing native map and native viewer.
- [frontend/src/lib/components/ZoomView.svelte](https://github.com/koo5/hillview/blob/1020df5290ce67938a90b5e9f91ce70f9f847047/frontend/src/lib/components/ZoomView.svelte), [OpenSeadragonViewer.svelte](https://github.com/koo5/hillview/blob/1020df5290ce67938a90b5e9f91ce70f9f847047/frontend/src/lib/components/OpenSeadragonViewer.svelte), and [zoomView.svelte.ts](https://github.com/koo5/hillview/blob/1020df5290ce67938a90b5e9f91ce70f9f847047/frontend/src/lib/zoomView.svelte.ts): zoom is a **global overlay/store**, not its own route. OpenSeadragonViewer imports mapState, annotations, photo rating, terrain overlay, share utilities and aliases from shared/zoomview. A dedicated embedding entry point for a photo UID is preferable to reconstructing the whole Svelte map state.

### CMP app

- [frontend2/README.md](https://github.com/koo5/hillview/blob/1020df5290ce67938a90b5e9f91ce70f9f847047/frontend2/README.md): thin androidApp/desktopApp, UI+logic in shared commonMain, platform implementations under androidMain/jvmMain. iOS is not currently wired.
- [frontend2/shared/src/commonMain/kotlin/cz/hillview/App.kt](https://github.com/koo5/hillview/blob/1020df5290ce67938a90b5e9f91ce70f9f847047/frontend2/shared/src/commonMain/kotlin/cz/hillview/App.kt) and [nav/Routes.kt](https://github.com/koo5/hillview/blob/1020df5290ce67938a90b5e9f91ce70f9f847047/frontend2/shared/src/commonMain/kotlin/cz/hillview/nav/Routes.kt): Navigation3, serializable NavKeys, central NavDisplay and Back stack. Web routes should be represented by a small serializable WebKey(path) or domain-specific route keys; add them to polymorphic saved-state registration.
- [frontend2/shared/src/commonMain/kotlin/cz/hillview/auth/SessionManager.kt](https://github.com/koo5/hillview/blob/1020df5290ce67938a90b5e9f91ce70f9f847047/frontend2/shared/src/commonMain/kotlin/cz/hillview/auth/SessionManager.kt) with [TokenStore.kt](https://github.com/koo5/hillview/blob/1020df5290ce67938a90b5e9f91ce70f9f847047/frontend2/shared/src/commonMain/kotlin/cz/hillview/auth/TokenStore.kt) owns login, refresh and auth state. Android uses shared-kt AuthenticationManager, also used by uploads. **Do not build a parallel refresh-token engine inside JS.**
- [frontend2/shared/src/commonMain/kotlin/cz/hillview/map/MapState.kt](https://github.com/koo5/hillview/blob/1020df5290ce67938a90b5e9f91ce70f9f847047/frontend2/shared/src/commonMain/kotlin/cz/hillview/map/MapState.kt) has MapStateHolder, the single state authority. Read docs/one-state.md and OneStateArchitectureTest before exposing map interactions to the bridge.
- [frontend2/shared/src/commonMain/kotlin/cz/hillview/viewer/ViewerPane.kt](https://github.com/koo5/hillview/blob/1020df5290ce67938a90b5e9f91ce70f9f847047/frontend2/shared/src/commonMain/kotlin/cz/hillview/viewer/ViewerPane.kt): **native directional viewer already exists**. Keep it and use Svelte primarily for high-resolution OpenSeadragon zoom / photo detail instead of duplicating viewer state.
- [frontend2/CLAUDE.md](https://github.com/koo5/hillview/blob/1020df5290ce67938a90b5e9f91ce70f9f847047/frontend2/CLAUDE.md): preserve one-state rule, shared-kt service ownership and existing build/test architecture contracts.

## Map engines — a deliberately open, independent decision (2026-11-22)

**Do not couple the map replacement to the WebView route reuse.** The new CMP app should own a native map; the Svelte website's map renderer is undecided and may remain Leaflet or change independently. Sharing MapLibre's *data/style concepts* can be useful, but using the same rendering library on the two platforms is **not** a requirement. The embedded web screens must communicate map destinations to the **native** CMP map instead of opening the old web map.

The inspected `frontend2/shared/src/androidMain/kotlin/cz/hillview/map/MapScreen.android.kt` already uses a **native osmdroid MapView** through Compose AndroidView. `OsmMapView.kt` adapts the current XYZ raster tile provider palette; `PhotoMarkerOverlay.kt` draws the custom crosses, bearing arrows and roses and owns tap hit-testing. This is currently not a Leaflet WebView. The existing `MapStateHolder` is library-agnostic; keep it that way. However, osmdroid itself was archived on 2024-11-20, so it is an interim rather than ideal long-term engine.

**Native candidate:** evaluate [MapLibre Compose](https://maplibre.org/maplibre-compose/) (as of November 2026 v0.20.0, pre-1.0 API) first, with direct [MapLibre Native Android](https://maplibre.org/maplibre-native/android/api/) as a fallback if Compose's marker/annotation APIs prove restrictive. MapLibre Compose targets Android/iOS/desktop and supports raster as well as vector sources, GPU-rendered styles and GeoJSON layers; check current release/compatibility with frontend2's Kotlin/Compose versions. The hard part will be faithfully replacing the custom `PhotoMarkerOverlay` drawing, bearing rotation, grouping and tap targets without sacrificing performance or the fixed **70-screen-pixel range circle** and `MapStateHolder` state contract. **Do not rewrite the map just to embed simple Svelte routes.**

**Web candidates:** retain Leaflet until there's a measured problem; evaluate [MapLibre GL JS](https://maplibre.org/maplibre-gl-js/docs/guides/leaflet-migration-guide/) if rotation/vector styling or many photo markers motivate the change. It can still consume raster XYZ tiles. [OpenLayers](https://openlayers.org/) is another option if mixed raster/complex geospatial overlays matter more. Consider WebGL compatibility on older hardware and preserve a fallback if needed. `frontend/src/lib/components/Map.svelte` currently mixes Leaflet rendering with location/bearing and other application logic; first extract a map renderer adapter rather than attempt a direct 2,899-line port.

### Separate map evaluation TODO (not a prerequisite for P0–P2)

- [ ] Record the actual map-engine requirements: existing raster providers, vector style/terrain goals, map rotation, bearing and location cues, 70px range, layered photo markers, rose grouping, source filters, tap/long-press and hit testing, offline/cache behaviour, and memory/battery profile.
- [ ] Prototype MapLibre Compose on **Android** behind the existing `expect/actual MapScreen` seam. Demonstrate raster XYZ basemap + a meaningful number of photo markers with heading arrows; benchmark gestures and panning, and test actual devices, not only emulators.
- [ ] Port custom marker rendering/hit testing only if validated; make geo model/state independent of the renderer. Existing native osmdroid map remains working until replacement is feature-complete.
- [ ] Evaluate MapLibre GL JS versus current Leaflet as a **separate website project** only after identifying what Leaflet is limiting. Existing web Map.svelte remains functional throughout.
- [ ] Keep website maps and native maps linked through **domain intents** (lat/lon/zoom/bearing/photo UID), not shared renderer APIs or a synchronised JS map store.

## Proposed architecture

1. SvelteKit remains **one web codebase**. Introduce an explicit *embedded CMP* environment flag/handshake (do not masquerade as Tauri). Suppress duplicate web menu/header only in embedded mode.
2. Compose owns overall navigation, lifecycle and login. Android WebView is plumbing in androidMain. Put navigation/bridge *contracts* in commonMain. For desktop, a browser fallback is acceptable at first.
3. Start by serving the **live website** to prototype /users and /bestof: no new asset pipeline, immediate web updates. This requires internet; an app-hosted origin may have different cookie/auth semantics from the real browser. Compare later with **bundled static SPA**, which the existing Svelte adapter supports, for offline startup, release reproducibility and security isolation. Do not choose permanently without measurement.
4. Implement a **typed, versioned, minimal message protocol** in both runtimes. Request-id + success/error responses, explicit navigation events. Use AndroidX WebKit WebViewCompat.addWebMessageListener with exact HTTPS allowed origins; do not enable broad JS-to-native interfaces or navigate arbitrary origins inside a bridge-enabled WebView.
5. Auth: implement CmpTokenManager behind the existing TS TokenManager interface. Native SessionManager/shared-kt remains the *sole* refresh-token authority. JS may request a short-lived current access token (with expiry handling); **never send refresh tokens**. Login/logout and session invalidation must update both sides and survive WebView re-creation; check that backend CORS allows the chosen origin. For a remotely hosted website, minimise token exposure and consider a more restricted API/proxy if needed. Server-side SvelteKit loaders cannot call this bridge; use CSR for authenticated embedded content or establish an intentional cookie session.
6. Navigation: internal whitelisted web routes stay in WebView; map/camera/native destinations emit structured messages to CMP, rather than loading the old Svelte map at /. Open external links outside the privileged WebView. Handle Android Back: zoom overlay first, then web history, then native Navigation3 Back.
7. Viewer: define a **photo-UID-based web entry point** that can open ZoomView/OSD without constructing a second mapState. Pass photo UID and optional normalized zoom bounds. Web clicks that imply map position/bearing should issue typed intents to KMP; MapStateHolder remains authoritative. Do not start WebView sensor subscriptions independently.
8. WebView lifecycle: retain one instance where useful for route transitions; guard duplicate loads/recompositions; test gesture ownership (OSD pinch/drag), foreground/background, restoration, link handling and page errors.
9. Never gate security solely on ?embedded=1 or a user-agent string. Establish trusted bridge presence and validate message types, origins and navigation targets on both sides.

## Ordered TODO and acceptance criteria

### P0 — First anonymous working route

- [ ] Sync H2 to pinned/current Hillview source tree; place this plan alongside the actual frontend2 module.
- [ ] Add a serializable web route/navigation destination in commonMain + App.kt; retain native MainKey map/camera and existing viewer.
- [ ] Implement Android WebView Compose hosting in androidMain, with correct disposal, retained state, safe HTTPS navigation policy, error page and Back handling. Add AndroidX WebKit dependency if absent.
- [ ] Open /users and /bestof from CMP using remote Hillview URL; hide only the duplicate site chrome in verified embedded mode. Preserve regular web behaviour.
- [ ] Verify public fetch, pagination, direct linking and returning to native CMP; don't accidentally route / or /?photo= into the old Svelte map.
- [ ] Create minimal tests for route classification/navigation and one Android UI smoke test (a web route opens, Back returns to CMP).

### P1 — Bridge + shared login

- [ ] Create a small JSON protocol document/type definitions (protocol version, requestId, action, payload, result/error).
- [ ] Implement allowed-origin message bridge and tested Kotlin dispatcher; reject arbitrary URLs/actions, unknown messages and malformed JSON.
- [ ] Add CMP platform detection and CmpTokenManager in frontend/src/lib/tokenManagerFactory.ts (with interface compatibility, no Tauri internals required).
- [ ] Expose SessionManager current access-token/login status to the web side, never refresh tokens. Ensure serialised refresh on Android, and bidirectional logout/session-expired propagation.
- [ ] Verify signed-in and anonymous /bestof, /activity, /users, /users/[id] plus the browser website unaffected; check 401 refresh, offline and 5xx behaviour.
- [ ] Make embed-mode init robust across a new WebView, process recreation, network failure and repeated login.

### P2 — Native map ↔ web photo / OpenSeadragon

- [ ] Inspect the current native ViewerPane entry points and old ZoomView/Photo.svelte data shape; choose a stable openPhoto(uid, optional normalized bounds) contract.
- [ ] Add a web-only entry route/component that obtains its own photo details / pyramid metadata and instantiates ZoomView or OpenSeadragonViewer without a second map/gallery owner. Preserve normal website photo routes and sharing.
- [ ] From native directional viewer, open high-res OSD in WebView. From bestof/activity/user results, ensure photo links lead to an intentional destination (web detail/zoom vs native map).
- [ ] Translate web “show on map” to an intent into MapStateHolder: lat, lon, zoom, bearing, optional UID. Do not sync the entire Svelte mapState store.
- [ ] Verify annotations, DZI/pyramids, terrain overlay, pinch-zoom, tap, image orientation, close/back, share links, 360 Pannellum fallback and viewport bounds.
- [ ] Pay particular attention to Android WebView scroll/gesture interactions and full-screen composable insets.

### P3 — Extras and release

- [ ] Add /about (external links/version/chrome); optionally account/contributions/notifications if justified.
- [ ] Revisit /photos only after splitting server photos from old Tauri local imports/uploads/statuses; keep native device photo queue UI separate.
- [ ] Choose remote website vs bundled SPA for release using measured startup/cache/offline behaviour, SSR/cookies, compatibility and remote-JS bridge risk. A static build must include shared/zoomview aliases and the existing SPA fallback.
- [ ] Check non-Tauri website regression in the Node adapter build and Playwright.
- [ ] Device-test release replacement of Tauri (same applicationId cz.hillview, Play App Signing/upload key and incremented versionCode). Given the minimal user base, no general legacy state migration is planned; warn users if unsent photos would be lost.

## Explicitly out of scope

- Embedding the Rust/Tauri runtime inside Compose; emulating all __TAURI_INTERNALS__ APIs.
- Replacing native camera, location sensors, uploader, map or directional viewer with Svelte.
- Duplicating Svelte route source within frontend2.
- Migrating old Tauri WebView IndexedDB/preferences/uploads (unless a concrete release blocker arises).
- Solving iOS/desktop WebView parity before Android proof of concept.

## Verification commands (in synced Hillview checkout)

~~~sh
cd frontend
bun run check
bun run test:unit
# Build website with the existing Node-adapter deployment process
# Build static SPA using its existing default adapter
cd ../frontend2
./gradlew :shared:jvmTest :shared:testAndroidHostTest
./gradlew :androidApp:assembleDebug
~~~

Automated builds and device tests have **not** been run for this document. The first implementer should validate the actual build variants and environment variables rather than assuming that both adapter modes behave identically.
