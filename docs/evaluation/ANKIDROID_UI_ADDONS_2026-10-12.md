# AnkiDroid UI & Add-ons Feasibility Audit — 2026-10-12

> Status: SOURCE-INSPECTED / DESIGN-ONLY; NO APP PATCH, NO APK BUILD, NO DEVICE QA.
> Upstream: https://github.com/ankidroid/Anki-Android
> Source baseline: `153ee7521b598049218c56ad8401948018fc18e5`
> User fork: https://github.com/lite-milkfrog/Anki-Android
> Local worktree: `D:\AgentData\10_Workspaces\ankidroid-ui-lab`; branch `eval/ui-plugin-market`
> Scope: all-app appearance, Android frontend workflow routing, on-device add-on installation/store.

## 1. Source observations vs. unverified UX hypotheses

- The main deck home is `AnkiDroid/src/main/java/com/ichi2/anki/DeckPicker.kt` (~2,561 lines), combining navigation, tree/list, sync, import, menu and error handling; it uses `ActivityHomescreenBinding`, `IncludeDeckPickerBinding`, `DeckAdapter` and `NavigationDrawerActivity`. **Complexity: high** for a wholesale replacement, medium for incremental redesign.
- The bottom navigation already exists (`HomeScreenNavigation.kt`), but `DeckPicker.bottomNavigationEnabled` requires `Prefs.devBottomNavEnabled` and a non-fragmented layout. Do not blindly rebuild this; stabilize/adapt existing code. **Complexity: medium**.
- Global styling is mixed: legacy Android XML/AppCompat/Material View-based screens, `PreferenceFragmentCompat` in `preferences/Preferences.kt`, and a smaller Compose island with `com/ichi2/compose/theme/Theme.kt`. Compose currently derives its colors from XML theme attributes; this is a useful token integration seam, **not** evidence that the entire app is Compose. **Complexity: medium–high**.
- The study/reviewer stack mixes native controls and HTML/JS in WebView. Full reviewer UI changes require preserving answer gestures, media, WebView JS API, templates, accessibility and FSRS scheduling. **Complexity: high**; do not alter the Rust-backed scheduler.
- Confirmed upstream bug reports (as opposed to our own untested visual opinions): #21807 Back in template editor shows discard dialog before dismissing keyboard; #21795 keyboard covers template editor; #21794 keyboard covers bottom of screen; #21780 edge-to-edge deprecation cleanup; #21779 and #21778 dialog UX improvements.
- UX opportunities **not yet verified on a running device**: deck hierarchy legibility, dense counts, cleaner toolbars and FAB, coherent dark/light/system palette, responsive tablet split-view, unified search, predictable back behavior, modest animations/reduced motion.

## 2. Correct existing workflow route

- Use local `wf_03a33f50feebc50626b9` (Android × Frontend Product Builder), which incorporates the design/UX standards of `wf_d8dc26b97fa1c1164f59`. Resolve `android-mobile` → `apk-app-factory`, with `android-dev-workflow` and selected native design/QA specialists.
- Route **native View/XML + Compose** as Android native; use frontend skills for UX specification, design tokens, information architecture, polish criteria and independent review. **Do not** ship React/Tailwind/CSS as a substitute for native pages, or count browser Playwright QA as APK QA.
- The reviewer WebView is a separate eligible web-like surface: browser-level tests supplement, but do not replace, on-device tests.
- Gates: inspect upstream diff → freeze design contract → incremental native refactor → Gradle build/unit/lint → emulator install/interaction/screenshots/logcat → screen states/375-ish compact + tablet, font scaling, Light/Dark/System and reduced-motion → independent regression → opt-in phone verification.
- Windows inspection: Git clone successful; Git 2.55; Java 17; adb 37; Android SDK/emulator files available; `adb devices -l` showed no connected/running device. **No Gradle build, emulator screenshots or measured performance results are claimed**.
- Upstream `AI_POLICY.md` restricts AI-produced upstream contributions for newcomers; independent fork experimentation is not permission to submit unapproved AI-generated upstream PRs.

## 3. Add-on marketplace: feasibility, compatibility and security

- Official Anki **desktop Python** add-ons are not directly executable in Android. A store cannot safely claim general compatibility with ankiweb.net/addons.
- There is partial code under `jsaddons/`: `AddonData.kt` validates npm-like package metadata and the JS API version, including `addonType=reviewer|note-editor`, and parses registry-like lists; `TgzPackageExtract.kt` extracts `.tgz`. `AnkiDroidJsAPI.kt` exposes reviewer JavaScript integration, current API version `0.0.3`. A source usage scan of `AnkiDroid/src` found the manifest/extract utilities used only within their implementations/tests, **not a wired user-facing installer/store**. This is partial groundwork, not a released add-on platform.
- Upstream issue #7959 (“JavaScript Addons support for AnkiDroid”) remains open, targeting a future release. See https://github.com/ankidroid/Anki-Android/issues/7959.
- Recommended extension tiers:
  1. **Theme/layout packs**: declarative palettes, typography, spacing, icons and approved native UI slots; safest initial extension type. Prevent arbitrary XML injection.
  2. **Reviewer/card JavaScript extensions**: inject only into controlled local review content under a versioned, minimized capability API, with per-plugin enable/disable, compatibility checks and safe-mode recovery.
  3. **Companion Android apps / explicit AnkiDroid API integrations**: external app-to-app calls without arbitrary host code loading. Clearly label as “requires companion app”.
  4. **Desktop Python add-ons**: catalog only if an independently verified Android alternative exists; mark unsupported otherwise.
- Marketplace UI: Explore / Installed / Updates, filters by compatibility and permissions, source (curated/community/local), versions, checksums, changelog, per-plugin reviews or warnings, one-tap install/disable/uninstall, rollback and per-addon logs. Sources can include a curated manifest as well as audited GitHub or npm packages. “Official” requires actual AnkiDroid team endorsement; our own list must be labeled third-party.
- Security requirements: HTTPS, publisher/manifest validation, cryptographic digest, archive path traversal + decompression limit, no untrusted third-party URL in privileged WebView, explicit capabilities & least privilege, content security controls, kill switch/safe mode, rollback, compatibility/API version guard, malicious-reporting mechanism, no bypass of Play device/network policy. Dynamic downloaded DEX/JAR/native code is **not** a suitable Google Play strategy; restricted interpreted JS in WebView can be feasible subject to policy and security review.
- Estimates (rough engineering complexity, NOT schedules): declarative themes 2/5; UX redesign of deck list 3/5; unified native navigation/settings 4/5; JS add-on sandbox/runtime 5/5; one-tap secure marketplace 5/5; desktop Python add-on compatibility 5/5 and not recommended.

## 4. Suggested order and exit criteria

- P0 audit/QA: establish reproducible baseline APK and emulator screenshots; check existing user data backup, schema, sync and offline flows. No changing the app id of the installed production app during testing.
- P1 UI: unify colors/type/touch targets, customize deck home (hierarchy/metrics), navigation and settings, then fix editor keyboard/insets. Each substep passes device regression and handles tablet.
- P2 marketplace MVP: install/uninstall **declarative themes** and a tiny curated set of reviewer JS samples via a local manifest, with offline state and safe-mode. No remote execution of arbitrary code.
- P3 trusted community marketplace: signed/verified catalog, update/rollback, report/audit pipeline and secure permissioning.
- P4 sync/compatibility: keep `.apkg`, nested deck structures, notes, scheduling history, AnkiWeb sync unchanged; do not silently inject card modifications into synced templates.

## 5. Findings / decision

**GO for an incremental Android UI fork. GO with constraints for a curated add-on marketplace. NO-GO for claiming that desktop Python Anki add-ons can be installed unchanged on Android.**

This audit is a source-based feasibility assessment, **not** a completed frontend redesign or an installed extension store. Independent emulator tests, security review and package QA remain open.

References: local source paths above; https://github.com/ankidroid/Anki-Android/issues/7959 ; https://support.google.com/googleplay/android-developer/answer/16559646 ; https://developer.android.com/privacy-and-security/risks/insecure-webview-native-bridges ; https://github.com/ankidroid/Anki-Android/blob/main/AI_POLICY.md .
