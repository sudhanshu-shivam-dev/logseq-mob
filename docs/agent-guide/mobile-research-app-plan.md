# Mobile Research App Plan for Logseq (Android Focus)

## Context

Logseq is a privacy-first, open-source knowledge management platform with a mature mobile app built on ClojureScript + Capacitor. The iOS app is in **alpha** and Android is listed as **"coming soon"**. This plan focuses on **bringing the Android app to production quality** with research workflows (quick capture, search, offline use, sync reliability).

---

## App Flow (Android)

### Startup Sequence

```
AndroidManifest.xml (singleTask launch)
  ↓
MainActivity.onCreate()
  ├── Register 8 Capacitor plugins (FolderPicker, UILocal, NativeTopBar,
  │   NativeBottomSheet, NativeEditorToolbar, NativeSelectionActionBar,
  │   LiquidTabs, Utils)
  ├── super.onCreate() → Capacitor BridgeActivity loads WebView (index.html)
  ├── Configure WebView (no over-scroll, wide viewport)
  ├── Apply theme via LogseqTheme.update() (detect dark mode, set colors)
  ├── Hand WebView to ComposeHost for Compose-based navigation
  ├── Register BroadcastReceiver for ACTION_ROUTE_CHANGED
  └── Schedule delayed sendIntentReceived event (5s)
```

### ComposeHost WebView Architecture

```
ComposeView
├── NavHost (Jetpack Compose Navigation)
│   └── AndroidView
│       └── FrameLayout (root)
│           ├── webContainer (holds single shared WebView)
│           └── overlayContainer (WebViewSnapshotManager for transitions)
```

The **single shared WebView** is reparented between Compose navigation destinations. `WebViewSnapshotManager` captures a frozen bitmap before transitions (220ms slide + 180ms fade) to prevent visual glitches.

### ClojureScript Initialization (inside WebView)

```
index.html loads static/mobile/js/main.js
  ↓
mobile.core/init()
  ├── Install native bridge: window.LogseqNative.onNativePop()
  ├── Set up reitit router with 4 routes:
  │   ├── "/" → Home (journals)
  │   ├── "/page/:name" → Page view
  │   ├── "/import" → Import
  │   └── "/export" → Export
  ├── mobile.init/init!()
  │   ├── App lifecycle listeners (foreground/background)
  │   ├── Keyboard listeners (keyboardWillShow/Hide)
  │   ├── Network monitoring (Capacitor Network plugin)
  │   ├── Intent/share handler (appUrlOpen + getLaunchUrl)
  │   └── Deep link handler (logseq:// scheme)
  └── Render app component via ReactDOM
```

### Navigation Flow (Multi-Stack Per-Tab)

Each of the 5 tabs maintains its own navigation stack:

```
Bottom Tab Bar (LiquidTabsPlugin.kt - Compose Material3 NavigationBar)
├── Home    → stack: ["/"]
├── Graphs  → stack: ["/__stack__/graphs"]
├── Capture → stack: ["/__stack__/capture"]  
├── Go To   → stack: ["/__stack__/go to"]
└── Search  → stack: ["/__stack__/search"] (native EditText overlay on Android)
```

### Example: User Taps a Search Result

```
User taps result in native search overlay
  ↓
LiquidTabsPlugin fires "openSearchResultBlock" event
  ↓
bottom_tabs.cljs listener → route-handler/redirect-to-page!(uuid)
  ↓
reitit router navigates to /page/:name
  ↓
navigation.cljs records intent as "push", calls notify-route-change!()
  ↓
UILocal.routeDidChange({navigationType: "push", stack: "search", path: "/page/uuid"})
  ↓  (Capacitor bridge)
MainActivity BroadcastReceiver → NavigationCoordinator.onRouteChange()
  ↓
ComposeHost.applyNavigation("push", path)
  ↓
WebViewSnapshotManager.showSnapshot() → freeze current view
NavController.navigate() → slide-from-right animation (220ms)
WebViewSnapshotManager.clearSnapshot() (260ms later)
  ↓
Page renders in WebView with smooth left-to-right slide
```

### Back Navigation

```
Android back button / gesture
  ↓
onBackPressed() → window.LogseqNative.onNativePop()
  ↓
CLJS: pop-modal!() → checks lightbox, dialog, selection, edit mode
  ↓ (if no modal)
CLJS: pop-stack!() → remove last route from active stack history
  ↓
replace-state with previous route → notify-route-change!()
  ↓
ComposeHost animates slide-from-left (pop transition)
```

### Data & Sync Flow

```
DataScript (in-memory Datalog DB)  ←→  SQLite (persistent via db-worker thread)
      ↕
  RTC Sync (WebSocket)  ←→  Remote Server
      ↕
  mobile-flows/network-status monitoring
  rtc-background-tasks auto-start/stop/restart
```

- On app pause: `editor-handler/save-current-block!`
- On resume: Check db-worker lock via `navigator.locks`; reload if worker died
- Network changes: `mobile-flows/*mobile-network-status` tracks online/offline

### Android-Specific Native Plugins

| Plugin | File | Purpose |
|---|---|---|
| `UILocal` | `UILocal.kt` | Route notifications, native alerts/date pickers, theme sync |
| `LiquidTabs` | `LiquidTabsPlugin.kt` | Material3 bottom tab bar + native search overlay (EditText + ScrollView) |
| `ComposeHost` | `ComposeHost.kt` | WebView hosting in Compose NavHost with slide animations |
| `NavigationCoordinator` | `NavigationCoordinator.kt` | Per-stack path history tracking |
| `WebViewSnapshotManager` | `WebViewSnapshotManager.kt` | PixelCopy bitmap snapshots for transitions (max 4096x4096, 16MB) |
| `NativeTopBar` | `NativeTopBarPlugin.kt` | Top action bar with Material icons |
| `NativeEditorToolbar` | `NativeEditorToolbarPlugin.kt` | Context toolbar for editor actions |
| `NativeSelectionActionBar` | `NativeSelectionActionBarPlugin.kt` | Text selection actions |
| `Utils` | `Utils.kt` | Device utilities, theme colors |
| `FolderPicker` | `FolderPicker.java` | File/folder selection dialogs |

---

## Phase 1: Bug Fixes & Stability (Weeks 1-4)

### 1.1 Fix double `appUrlOpen` intent firing
- **File**: `src/main/mobile/init.cljs:19-38`
- **Problem**: `appUrlOpen` fires twice for the same intent; current workaround uses `.getSeconds()` comparison with a 1-second dedup window (wraps at 60, fragile)
- **Fix**: Use URL + timestamp hash for deduplication instead of time-second comparison

### 1.2 Fix Android keyboard auto-open for quick-add
- **File**: `src/main/mobile/components/app.cljs:90-99`
- **Problem**: `capture` component only calls `quick-add-open-last-block!` on iOS (line 94: `when (mobile-util/native-ios?)`)
- **Fix**: Add Android path using `Keyboard.show()` from Capacitor or native `InputMethodManager` via the Utils plugin

### 1.3 Fix bracket button icon on Android
- **File**: `src/main/mobile/components/editor_toolbar.cljs:110-111`
- **Problem**: `page-ref-action` uses `"parentheses"` as system icon (SF Symbol); `MaterialIconResolver.kt` must map this to a Material icon
- **Fix**: Verify the mapping exists in `MaterialIconResolver.kt`; if not, add a Material equivalent (e.g., `Icons.Outlined.Code`)

### 1.4 Implement export functionality
- **File**: `src/main/mobile/components/header.cljs:54-59`
- **Problem**: Export menu item is commented out with `TODO: support export`
- **Fix**: Uncomment and wire up the existing `frontend.components.export/export` component (route already exists in `routes.cljs:20-22`)

### 1.5 Android snapshot memory management
- **File**: `android/app/src/main/java/com/logseq/app/WebViewSnapshotManager.kt`
- **Problem**: Stores screenshot bitmaps without explicit eviction; max 4096x4096 / 16MB per bitmap
- **Fix**: Add `onTrimMemory` handler in `MainActivity.java` to call `WebViewSnapshotManager.clearSnapshot()` under memory pressure; implement LRU eviction if multiple snapshots accumulate

### 1.6 Developer experience improvements
- Add React error boundaries around tab content in `components/app.cljs` so crashes in one tab don't blank the app
- Document `LOGSEQ_APP_SERVER_URL` usage in `docs/develop-logseq-on-mobile.md`

---

## Phase 2: Performance & Android Parity (Weeks 5-8)

### 2.1 Startup performance
- **Lazy module loading**: `shadow-cljs.edn` already defers `:code-editor`; identify more candidates (import/export, audio recorder, graph management)
- **Defer non-critical listeners**: Move `sendIntentReceived` and network listener setup to after first render in `init.cljs`
- **Add timing instrumentation**: `performance.mark/measure` from JS load to first paint, expose in Settings > Log

### 2.2 Rendering performance
- Audit Rum components for missing `rum/static` annotations (prevents unnecessary re-renders)
- Add CSS `contain: layout style paint` to scroll containers (`#app-main-home`, `#main-content-container`)
- Consider virtual scrolling for `journal/all-journals` on large graphs

### 2.3 Android feature parity

| Feature | Gap | Fix |
|---|---|---|
| Audio transcription | `UILocal.kt` rejects with "not supported on Android" | Implement via Android `SpeechRecognizer` API or ML Kit |
| Rich alerts | Uses basic `Toast` vs iOS rich `Drops` library | Replace with Material 3 `Snackbar` or custom drop-in |
| Dynamic Type | Not tracked on Android | Read `resources.configuration.fontScale`, sync to `--ls-mobile-font-scale` CSS variable |
| Back gesture | `onBackPressed` is deprecated | Migrate to `OnBackPressedDispatcher` (import exists but registration is incomplete) |
| App shortcuts | Not implemented | Add Android App Shortcuts for Quick Add and Audio Record |

### 2.4 Sync resilience
- Ensure RTC auto-restart in `rtc-background-tasks.cljs` uses exponential backoff (avoid connection storms on mobile)
- Batch pending operations on reconnection rather than replaying one-by-one

---

## Phase 3: Features & Offline (Weeks 9-14)

### 3.1 Research-specific features (High Priority)

**Enhanced quick capture:**
- Configurable capture templates per content type (text, URL, image, audio)
- Automatic tagging based on source app in `frontend/mobile/intent.cljs`
- Android: Improve `ACTION_SEND` intent handling in `AndroidManifest.xml` for broader MIME type support

**Better search & discovery:**
- Full-text search with snippet preview
- Search filters (date range, tags, page type)
- Graph-aware search (find connected concepts)

**Offline reading & annotation:**
- Ensure all page content is available offline after initial sync
- Implement read-later/bookmark queue

### 3.2 Missing desktop features
- **All Pages view**: No route exists (mobile has only 4 routes in `routes.cljs` vs desktop's many)
- **Settings page**: Mobile has minimal settings popup; needs comprehensive settings
- **Page properties editing**: Verify inline editing support
- **Flashcards/spaced repetition**: Enable for mobile review sessions

### 3.3 Offline-first improvements
- **Offline operation queue**: Queue edits in SQLite when offline; replay on reconnection (currently only shows yellow indicator with no queue)
- **Conflict resolution UI**: Surface sync conflicts to users in a "Sync Conflicts" section
- **Background sync on Android**: Use `WorkManager` for reliable periodic background sync with network constraints
- **Sync progress panel**: Replace colored dot with detailed panel (pending ops, last sync time, errors)

---

## Phase 4: Polish & Testing (Weeks 15-18)

### 4.1 Testing strategy (currently zero mobile tests)

**Unit tests** (integrate into `bb dev:lint-and-test`):
- `mobile.navigation`: Test `push-state`, `pop-stack!`, `switch-stack!` with mock state
- `mobile.deeplink`: Test URL parsing for all deep link types
- `frontend.mobile.intent`: Test `handle-received-text` with various intent payloads

**E2E tests** (extend `clj-e2e/`):
- Quick capture: launch via deep link, type, save, verify block created
- Search: type query, verify results, navigate to result
- Audio recording: start, stop, verify asset created

**Android native tests** (JUnit/Espresso):
- `NavigationCoordinator`: Test per-stack push/pop/replace/reset
- `ComposeHost`: Test navigation event flow and WebView reparenting
- `LiquidTabsPlugin`: Test tab selection, search event handling
- `WebViewSnapshotManager`: Test bitmap capture/clear lifecycle

### 4.2 Android accessibility
- Add `aria-label` attributes to all interactive elements in Rum components
- Implement font scale tracking: Read `resources.configuration.fontScale`, sync to `--ls-mobile-font-scale` CSS variable (mirrors iOS Dynamic Type)
- TalkBack support: Ensure proper content descriptions on native Compose components (tabs, top bar, toolbars)
- Respect `prefers-reduced-motion` in CSS transitions
- Ensure all touch targets meet 48x48dp minimum (Material guidelines; editor toolbar already has `minWidth: 44.dp` — increase to 48)
- Audit color contrast for sync indicator colors (`#16A34A` green / `#CA8A04` yellow)

### 4.3 Android production readiness
- ProGuard/R8 keep rules for Capacitor plugins and Compose reflection
- Verify Sentry crash reporting on Android (DSN configured in `shadow-cljs.edn`)
- Play Store compliance: Target SDK level, data safety declarations, accessibility requirements
- Verify all `AndroidManifest.xml` permissions are justified and declared in data safety form
- Test on a range of Android versions (SDK 26+) and OEM WebView implementations
- App signing and release build pipeline verification

---

## Critical Files (Android Focus)

| File | Role |
|---|---|
| **Android Native** | |
| `android/app/src/main/java/com/logseq/app/MainActivity.java` | App entry, plugin registration, theme, back press |
| `android/app/src/main/java/com/logseq/app/ComposeHost.kt` | WebView hosting in Compose NavHost, transitions |
| `android/app/src/main/java/com/logseq/app/NavigationCoordinator.kt` | Per-stack path history tracking |
| `android/app/src/main/java/com/logseq/app/LiquidTabsPlugin.kt` | Material3 bottom tabs + native search overlay |
| `android/app/src/main/java/com/logseq/app/UILocal.kt` | Route notifications, alerts, transcription (gap) |
| `android/app/src/main/java/com/logseq/app/WebViewSnapshotManager.kt` | PixelCopy transitions, memory management |
| `android/app/src/main/java/com/logseq/app/Utils.kt` | Device utils, theme colors |
| `android/app/src/main/AndroidManifest.xml` | Permissions, intent filters, deep links |
| **ClojureScript (shared)** | |
| `src/main/mobile/core.cljs` | Entry point, router, render loop |
| `src/main/mobile/init.cljs` | App lifecycle, keyboard, intent listeners |
| `src/main/mobile/navigation.cljs` | Stack-based per-tab navigation, native bridge sync |
| `src/main/mobile/routes.cljs` | Route definitions (only 4 routes currently) |
| `src/main/mobile/components/app.cljs` | Root UI with two-layer DOM (journals + page overlay) |
| `src/main/mobile/components/header.cljs` | Header with export TODO, sync indicator |
| `src/main/mobile/bottom_tabs.cljs` | Tab management, search integration |
| `src/main/frontend/mobile/intent.cljs` | Share intent handling (v1/v2) |
| `src/main/frontend/mobile/util.cljs` | Native platform detection, plugin wrappers |
| **Config** | |
| `capacitor.config.ts` | Capacitor plugin/platform config (Android HTTP scheme) |
| `shadow-cljs.edn` | Mobile build target config |

## Risks & Dependencies

1. **Android WebView fragmentation**: Rendering varies across versions/OEMs; the `WebViewSnapshotManager` has fallback paths but rendering differences may need vendor-specific workarounds
2. **DataScript memory limits**: Large graphs may exceed mobile memory (especially on lower-end Android devices); needs benchmarking
3. **Android background execution limits**: `WorkManager` is more reliable than alarm-based approaches but still subject to Doze mode and battery optimization
4. **SpeechRecognizer availability**: Not all Android devices have on-device speech recognition; need graceful fallback when unavailable

## Verification

- **Build**: `yarn mobile-watch` (dev), `yarn release-mobile` (prod) — verify clean compilation
- **Android build**: `npx cap sync android && npx cap run android` or `bb release:android-app`
- **Lint & test**: `bb dev:lint-and-test` — ensure no regressions
- **Manual testing**: Android Emulator via Android Studio, `chrome://inspect/#devices` for WebView debugging
- **Specific test**: For each bug fix, verify the FIXME/TODO scenario works end-to-end on Android device/emulator
