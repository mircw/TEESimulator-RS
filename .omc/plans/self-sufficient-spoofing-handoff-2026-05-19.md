# Self-Sufficient Spoofing — Session Handoff

Authoring session date: 2026-05-19
Branch: `feat/self-sufficient-spoofing`
Source plan: `/home/rootdev/.claude/plans/fancy-humming-firefly.md`
Status: 5 phase commits + 12 critic-fix commits + 5 second-adversarial-pass commits landed, full debug pipeline clean, all device tests deferred. 0 CRITICAL + 2 MAJOR (M6 device-only, M8 plan-only) remaining from the original critic list; two known limitations (F3 24 h re-arm, F13 PIF install-after-boot) documented. See "Second adversarial review — 2026-05-19" below.

## Resume protocol for next session

Read in order:

1. This document end-to-end.
2. `/home/rootdev/.claude/plans/fancy-humming-firefly.md` (source plan). Lives under rootdev's private home — readable directly under user `rootdev`, requires `sudo cat` under any other user.
3. `git log --oneline feat/self-sufficient-spoofing -14` to confirm branch state matches the table below.
4. The four critic findings sections below — every unresolved finding has a file:line citation and a proposed fix.
5. The "Untracked working-tree state" section before running any build or status check, so the noise doesn't trigger false alarms.

Then ask the user which findings to prioritize before writing code.

### Context that does NOT persist across sessions

- Raw critic agent transcripts (written to `/tmp/claude-*/.../tasks/*.output`) are wiped between sessions. The CRITICAL/MAJOR synthesis in this document is the only persistent record of those findings.
- OMC task system state (the 24 tracker tasks) lives in `.omc/state/sessions/<sessionId>/`. The 5 pending device-test tasks (#11, #15, #19, #23, #24) may not survive into the next session under a new session ID. Full step lists for all 5 are embedded below in the "Deferred device-test step lists" section so the work is reproducible even if `TaskList` returns empty.
- Background bash task outputs (`/tmp/claude-*/.../tasks/b*.output`) are session-scoped. Build artifacts in `out/` are durable on disk.

### User-context assumption

This session ran as user `rootdev`. The rust toolchain, `~/.cargo` HDD symlink, `~/.rustup` HDD symlink, `~/.gradle/init.d/00-per-user-builds.gradle.kts` symlink, and the build path under `/mnt/companion/rootdev/builds/TEESimulator/` are specific to that user. Other users (thinker, president, planner, claudetest) inherit the same `/etc/gradle-init.d/` + `/etc/profile.d/` infrastructure on next login — but their own `~/.cargo` and per-user `/mnt/companion/$USER/builds/TEESimulator/` will be empty until they run a build. If next session runs under a different user, expect to: install rust (`curl https://sh.rustup.rs -sSf | sh -s -- -y --default-toolchain stable --profile minimal --no-modify-path`), `cargo install cargo-ndk`, log out and back in so profile.d hooks fire.

## Branch state

```text
3bf58de fix(spoof): validate currentPatch against date regex
84161da docs(spoof): explain MAX_FUTURE_DAYS rationale
1c64688 fix(spoof): serialize concurrent applyToProps calls
0e752f8 fix(spoof): propagate read errors out of mergedContents
a4af3d5 fix(spoof): require = in global key-assignment check
44ea156 docs(plans): record critic-fix continuation in handoff
bf82486 docs(plans): add gradle init script modernization plan
49b7bfc fix(util): skip day synthesis for YYYY-MM input
9b982d4 fix(interception): emit KEY_SIZE for EC keys
c589e12 feat(spoof): hot-reload PIF via FileObserver
c111bd0 fix(spoof): skip empty PIF source files
5a519b9 fix(spoof): guard atomicWrite errors in updateTo
d14db7d fix(spoof): preserve [pkg] sections in atomicWrite
47222ab fix(spoof): respect system=prop passive default
b50d313 fix(spoof): order spoofers before keystore hook
311523e fix(spoof): isolate BulletinPoller.start failure
1ee2346 fix(spoof): wrap pollOnce in umbrella try/catch
9d693bc fix(spoof): allow UDP egress for DNS resolution
292f24a fix(spoof): bound future patch dates in updateTo
e287e96 docs(plans): audit handoff and fill gaps
9941dec docs(plans): wrap up session with full handoff
2f64730 chore(scripts): make package.sh find user-local cargo
14ec9c3 feat(spoof): periodic bulletin refresh via BulletinPoller
acdf452 feat(spoof): PatchLevelManager with PIF resolution
7abdba9 feat(spoof): resetprop bootloader lock at boot
90872a6 feat(install): drop default security_patch.txt at install
6fbb31d build(gradle): auto-rewrite update.json on packaging
d8326de build(gradle): expose cargo bin path to rust task
a8e40ef build(gradle): set kotlin jvmTarget to JVM_21
d34630f fix(interception): omit KEY_SIZE for EC keys with ecCurve
fe3c3b1 fix(interception): drop delete marker on key regen
e0105d6 wip(keystore): add F1 Phase A diagnostic logs in updateAad path
d4e2413 Merge pull request #21 from Andrea-lyz/fix/duck-detector-generate-fingerprint
```

The top 5 commits (`a4af3d5` through `3bf58de`) land the post-critic-review hardening from the second adversarial pass. The 12 commits below (`292f24a` through `49b7bfc`) are the first critic-fix continuation. `9b982d4` reverts the earlier `d34630f` (kept in history per "never amend published commits"). `e287e96` + `9941dec` are the prior handoff docs. Everything below `e287e96` matches the original session table. `e0105d6` was pre-existing on the base branch.

All authored by `Enginex0 <enginex0@users.noreply.github.com>`. No `Co-Authored-By` trailers anywhere. All commit messages conform to Conventional Commits (subject ≤50 chars where possible, body wraps at 72). Per project policy, branch lives local only — no `git push`, no PR.

## What was built

### Phase 1 — default install config
- `90872a6 feat(install): drop default security_patch.txt at install`
- File: `module/customize.sh` lines 95-105 (the inserted block)
- Behavior: out-of-box install seeds `/data/adb/tricky_store/security_patch.txt` with `system=prop`. `ConfigurationManager.kt:253-256` auto-forces `boot=prop` and `vendor=prop` when `system=prop`, giving full coverage from one line.

### Phase 2 — bootloader-lock spoofing
- `7abdba9 feat(spoof): resetprop bootloader lock at boot`
- New file: `app/src/main/java/org/matrix/TEESimulator/config/BootStateManager.kt`
- Modified: `app/src/main/java/org/matrix/TEESimulator/util/AndroidDeviceUtils.kt` (added `internal fun setProperty(name: String, value: String)` overload at line 167-183 after the existing private ByteArray variant)
- Modified: `app/src/main/java/org/matrix/TEESimulator/App.kt` (added import + `BootStateManager.apply()` call at line 48 after the M2 reorder)
- Behavior: at every boot, resetprops `ro.boot.verifiedbootstate=green`, `ro.boot.flash.locked=1`, `ro.boot.veritymode=enforcing`. Skips work when current already matches target.

### Phase 3 — PatchLevelManager
- `acdf452 feat(spoof): PatchLevelManager with PIF resolution`
- New file: `app/src/main/java/org/matrix/TEESimulator/config/PatchLevelManager.kt`
- Modified: `App.kt` (added import + `PatchLevelManager.initialize()` at line 49 after the M2 reorder)
- Behavior: resolves the active patch date from PlayIntegrityFix via 6-path override chain (`PatchLevelManager.kt:36-44`), falls back to `SystemProperties.get("ro.build.version.security_patch", Build.VERSION.SECURITY_PATCH)`. Validates YYYY-MM-DD format, rejects dates `< 20200101` or more than `MAX_PAST_OFFSET = 10000` (~1 year) in the past. On accept: atomic stage-and-rename `security_patch.txt` with `system=$date\nboot=$date\nvendor=$date\n`, then resetprops both system and vendor patch props.

### Phase 4 — BulletinPoller
- `14ec9c3 feat(spoof): periodic bulletin refresh via BulletinPoller`
- New file: `app/src/main/java/org/matrix/TEESimulator/config/BulletinPoller.kt`
- Modified: `module/sepolicy.rule` (appended 6 lines for ksu + magisk tcp_socket egress)
- Modified: `module/uninstall.sh` (added cleanup for `security_patch.txt`, `security_patch.txt.next`, `last_bulletin_fetch.json`)
- Modified: `App.kt` (added import + `BulletinPoller.start()` between `NativeCertGen.initialize(...)` and `Looper.loop()`)
- Behavior: dedicated `HandlerThread("BulletinPoller")` fetches `https://source.android.com/docs/security/bulletin/pixel` with 5s/30s/2m/10m/30m bootstrap backoff, then 24h steady cadence. Parses first `<td>YYYY-MM-DD</td>` match. If newer than current, calls `PatchLevelManager.updateTo(date)`. Persists last 10 attempts to `last_bulletin_fetch.json` (ring buffer, atomic rename).

### Build chore
- `2f64730 chore(scripts): make package.sh find user-local cargo`
- Modified: `scripts/package.sh` line 13-17 (prepend `$HOME/.cargo/bin` to PATH unconditionally so Gradle daemon inherits it)
- Reason: `commandLine("cargo")` in `buildRustCertgen` resolves against daemon-inherited PATH, not Exec.environment(). Non-login shells skip profile.d.

## App.kt init order (current)

`App.kt:37-76` flow (post-M2 reorder from `b50d313`, post-C5 isolation from `311523e`):

1. `SystemLogger.info("Welcome to TEESimulator!")`
2. `Thread.setDefaultUncaughtExceptionHandler { ... }` (logs only)
3. `prepareEnvironment()`
4. `BootStateManager.apply()` (Phase 2, pre-interceptor per M2)
5. `PatchLevelManager.initialize()` (Phase 3, pre-interceptor per M2)
6. `ConfigurationManager.initialize()`
7. `initializeInterceptors()` (blocks until keystore2 hook attaches)
8. `AndroidDeviceUtils.setupBootKeyAndHash()`
9. BouncyCastle provider swap
10. `NativeCertGen.initialize("/data/adb/modules/tricky_store/libcertgen.so")`
11. `BulletinPoller.start()` wrapped in its own `try { } catch (Throwable)` (Phase 4, C5)
12. `Looper.loop()` (blocks forever)

Steps 4 and 5 precede step 7 so keystore2's first binder calls observe the spoofed `ro.boot.verifiedbootstate` and `ro.build.version.security_patch` values rather than the un-spoofed defaults. Step 11 is isolated so a poller failure does not propagate to the outer `catch (e: Exception) { ...; throw e }` and kill the daemon — losing Phases 1/2/3 with it would defeat the point of C5.

## Infrastructure installed this session (system-level, NOT in git)

### Gradle multi-user build-dir collision fix

- `/etc/gradle-init.d/per-user-builds.gradle.kts` (root-owned, world-readable). Source-of-truth init script. Redirects `buildDir` + `projectCacheDir` to `/mnt/companion/$USER/builds/<projectSlug>/` for any project whose `rootDir` starts with `/home/president/Git-repo-success/`.
- `/etc/profile.d/gradle-per-user-init.sh` (root-owned). Lazy login-time symlink installer: ensures every user's `~/.gradle/init.d/00-per-user-builds.gradle.kts` points at the canonical script.
- `~/.gradle/init.d/00-per-user-builds.gradle.kts` (rootdev's symlink, laid manually this session because profile.d only fires on login).

### Cargo PATH fix
- `/etc/profile.d/cargo-path.sh` (root-owned). Prepends `$HOME/.cargo/bin` to PATH for any login shell that has the dir.

### Rust toolchain for rootdev
- `~/.cargo` → `/mnt/companion/rootdev/caches/cargo` (HDD symlink, canonical per CLAUDE.md)
- `~/.rustup` → `/mnt/companion/rootdev/caches/rustup` (HDD symlink)
- Installed: rust 1.95.0 stable, profile=minimal, default-toolchain=stable, via `rustup-init`
- Android targets: aarch64, armv7, i686, x86_64 (auto-pulled by `native-certgen/rust-toolchain.toml`)
- `cargo-ndk 4.1.2` installed via `cargo install`

### Project tree cleanup

The pre-existing `.gradle`, `build`, `app/build`, `native-certgen/target` symlinks (owned by `thinker`, pointing into `/mnt/companion/thinker/`) and the real `stub/build` directory (owned by `president`) were removed so the new Gradle init script handles all build-dir routing transparently. Other users (`thinker`, `president`) will need to re-lay their own gradle artifacts on their next build invocation — the init script handles this automatically for them too once they're logged in via profile.d.

The `app/.cxx` symlink (NDK cmake cache) was missed in the initial sweep and surfaced during the audit pass. It's now relaid to `/mnt/companion/rootdev/builds/TEESimulator/app-cxx` consistent with the others. The Gradle init script does NOT cover `app/.cxx` because that directory is owned by the CMake/NDK toolchain integration, not Gradle's `layout.buildDirectory`. If a future user hits a permission-denied on `app/.cxx`, the fix is `rm app/.cxx && mkdir -p /mnt/companion/$USER/builds/TEESimulator/app-cxx && ln -s /mnt/companion/$USER/builds/TEESimulator/app-cxx app/.cxx`.

## Untracked working-tree state

After this session, `git status --short` shows a long list of untracked files. None of them block the build, but next session should know what each is before deleting anything.

- `module/update.json` — shows as `M` (modified) almost always. The `refreshUpdateJson` gradle task (Step 0 Commit E) auto-rewrites this file on every package run to keep `versionCode` and `zipUrl` in lockstep with `gitCommitCount`. Current on-disk content reflects the last build's count, not the committed value. Safe to ignore unless you're explicitly bumping the release version.
- `local.properties` — untracked, owned by `thinker shared`. Contains `sdk.dir=/home/thinker/Android/Sdk` with an explanatory header saying it was redirected from president's path because president's NDK 27.3.13750724 sysroot is partially corrupted. Next session should keep this file as-is unless rootdev (or whoever) installs their own Android SDK. Thinker's SDK is readable by rootdev via group-shared mount.
- `app/.cxx` — symlink to rootdev's companion namespace (relaid during this session's audit). Don't delete.
- `harness/` — untracked dir containing `__pycache__/` and `tests/`. Purpose unknown — investigate before deleting. Likely test infrastructure left by a prior session.
- `.audit-refs/`, `.fixture-*/`, `logs_llm/`, `vectors.db*`, `.mcp-vector-search/` — agent-tooling artifacts (probably from prior sessions of vector search / fixture-based testing). Don't touch without understanding what created them.
- `.claude/`, `.vscode/`, `archives/`, `context.md`, `CLAUDE.md`, `docs/` — local config + scratch. The repo-root `CLAUDE.md` is the project instructions Claude reads on session start.
- `.omc/plans/_archive/` — historical handoffs and audits (including `tee-fingerprint-handoff-2026-05-02.md` which established the `<feature>-handoff-YYYY-MM-DD.md` naming convention this file now follows).
- `.omc/plans/tee-fingerprint-*` — 8 untracked plan files for the TEE fingerprint initiative (separate from this self-sufficient-spoofing work). `tee-fingerprint-phase-registry.md` is the registry. The relationship between that initiative and this one is not yet defined — open question whether they should converge.
- `.omc/.project-memory.json.tmp.*` — 7 stale tmp files. Safe to delete with `rm .omc/.project-memory.json.tmp.*` but they don't break anything.
- `.omc/sessions/`, `.omc/state/`, `.omc/research/` — OMC orchestration state. Treat as ephemeral; do not commit.

## Test status

### Validated this session
- `./gradlew :app:compileDebugKotlin` — clean after every phase commit (3-second incrementals, 59-second clean).
- `./gradlew zipDebug` (Kotlin pipeline only, `-x buildRustCertgen`) — `TEESimulator-RS-v6.0.0-175-Debug.zip` (12.4 MB).
- `./scripts/package.sh --debug` (full pipeline with fresh rust toolchain) — `TEESimulator-RS-v6.0.0-176-Debug.zip` (12.4 MB).
- `libcertgen.so` rebuilt by cargo-ndk under rootdev (replacing the pre-existing thinker-owned file with the same 1393792 bytes).

### Deferred — no ADB device attached this session
| Task | Plan section | What it verifies |
|---|---|---|
| #11 Phase 1 device test | `fancy-humming-firefly.md:124-131` | customize.sh writes 5-line default; Chunqiu code 26 absent; cert tags 706/718/719 match `getprop` |
| #15 Phase 2 device test | `fancy-humming-firefly.md:170-176` | resetprop took effect (`ro.boot.*` returns green/1/enforcing); KeyAttestation GREEN |
| #19 Phase 3 device test | `fancy-humming-firefly.md:265-273` | PIF date drives cert tags + getprop; YYYYMM vs YYYYMMDD encoding correct |
| #23 Phase 4 device test | `fancy-humming-firefly.md:370-377` | poller fetches successfully; ring buffer schema correct; sepolicy rules effective |
| #24 Final E2E | `fancy-humming-firefly.md:420-440` | full self-sufficient flow; no regressions in pre-existing module behavior |

To run all deferred tests: connect an ADB device, then:

```bash
./scripts/package.sh --release --deploy --clear-keys --reboot --verify
```

Then run the per-phase checks below. The step lists are embedded here in case the OMC task system has dropped tasks 11/15/19/23/24 by the time next session runs.

## Deferred device-test step lists

### Phase 1 (task #11) — default `security_patch.txt`

1. `./scripts/package.sh --release` (~11 min from clean).
2. `adb shell rm -f /data/adb/tricky_store/security_patch.txt` — wipe any existing config.
3. `./scripts/package.sh --deploy --clear-keys --reboot --verify`.
4. `adb shell cat /data/adb/tricky_store/security_patch.txt` — must show the 5-line default (4 comment lines + `system=prop`).
5. Launch Chunqiu detector on device; capture screenshot/log. Code 26 must NOT appear.
6. Launch KeyAttestation app, generate a test key, pull leaf cert, run `openssl x509 -text -in leaf.pem` and confirm tags 706/718/719 encode the same date that `adb shell getprop ro.build.version.security_patch` returns. Tag 706 encodes YYYYMM (6 digits); tags 718/719 encode YYYYMMDD (8 digits). The split is handled by `AndroidDeviceUtils.parsePatchLevelValue` at lines 336-357.

### Phase 2 (task #15) — bootloader-lock spoofing

1. Pre-deploy baseline: `adb shell "getprop ro.boot.verifiedbootstate; getprop ro.boot.flash.locked; getprop ro.boot.veritymode"` — capture current values (likely `orange / 0 / logging` on an unlocked test device).
2. `./scripts/package.sh --release --deploy --clear-keys --reboot --verify`.
3. Post-reboot, re-run step 1: expect `green / 1 / enforcing`.
4. KeyAttestation app — "Verified Boot State" row reports GREEN.
5. Chunqiu detector — any D44-family detections clear.
6. `adb logcat -d -s TEESimulator | grep -iE 'BootState|verifiedboot'` — confirm `BootStateManager.apply()` ran and logged each resetprop.

### Phase 3 (task #19) — PIF-driven patch level

1. `adb shell mkdir -p /data/adb/modules/playintegrityfix && echo '{"SECURITY_PATCH":"2025-11-01"}' | adb shell tee /data/adb/modules/playintegrityfix/custom.pif.json`.
2. `./scripts/package.sh --release --deploy --clear-keys --reboot --verify`.
3. Post-reboot: `adb shell getprop ro.build.version.security_patch` → `2025-11-01`.
4. `adb shell getprop ro.vendor.build.security_patch` → `2025-11-01`.
5. `adb shell cat /data/adb/tricky_store/security_patch.txt` → three `=2025-11-01` lines.
6. KeyAttestation app → generate key → pull cert → `openssl x509 -text` → confirm tag 706 encodes `202511` (YYYYMM, 6 digits), tags 718/719 encode `20251101` (YYYYMMDD, 8 digits).
7. Chunqiu detector — code 26 clear.
8. Negative test: delete the `custom.pif.json`. Reboot. Confirm patch resolves to `SystemProperties` fallback and no crash. `adb logcat -d -s TEESimulator | grep -i PatchLevel` should show the fallback log line.

### Phase 4 (task #23) — `BulletinPoller`

1. Pre-deploy: `adb shell rm -f /data/adb/tricky_store/last_bulletin_fetch.json`.
2. `./scripts/package.sh --release --deploy --clear-keys --reboot --verify`.
3. Wait 15 seconds post-reboot for first bootstrap attempt.
4. `adb shell cat /data/adb/tricky_store/last_bulletin_fetch.json` — first attempt entry shows `status: "success"` and a real `parsed_date`.
5. If `parsed_date` is newer than the device's current patch: `adb shell cat /data/adb/tricky_store/security_patch.txt` shows the new explicit dates; `adb shell getprop ro.build.version.security_patch` matches.
6. Negative network: `adb shell svc wifi disable && adb reboot`. Wait 60s. Check history shows `network_error` entries with exponential-backoff timestamp gaps.
7. SELinux verification: `adb shell sesearch -A -s ksu -c tcp_socket` (if sesearch present) OR `adb shell "dmesg | grep -iE 'avc.*denied' | grep -i ksu"` — must NOT show denials for outbound TCP. Note critic finding C3 — if denials appear, append the UDP rules first.

### Final E2E (task #24)

```bash
./scripts/package.sh --release --deploy --clear-keys --reboot --verify
```

On device:

```bash
adb root
adb shell getprop ro.boot.verifiedbootstate              # expect: green
adb shell getprop ro.boot.flash.locked                   # expect: 1
adb shell getprop ro.boot.veritymode                     # expect: enforcing
adb shell getprop ro.build.version.security_patch        # expect: matches cert tag 706
adb shell getprop ro.vendor.build.security_patch         # expect: matches cert tag 718
adb shell cat /data/adb/tricky_store/security_patch.txt
adb shell cat /data/adb/tricky_store/last_bulletin_fetch.json
```

Then:

1. Chunqiu detector → code 26 must NOT appear.
2. KeyAttestation app → "Verified Boot: GREEN" + matching patch dates in the cert (tag 706 = YYYYMM 6-digit, tags 718/719 = YYYYMMDD 8-digit).
3. Wait 24h (or trigger debug refresh) → confirm a new bulletin entry in the ring buffer history.
4. Regression check: `keybox.xml` still consumed (generate a key, confirm signing chain), `target.txt` still respected (try with a non-target app, confirm no interception), `hbk` seed still generated (file exists at `/data/adb/tricky_store/hbk`), `adb shell pidof supervisor` returns non-empty PID.

## Critic findings — adversarial review by 4 agents

Four critic agents were dispatched in parallel:
- `general-purpose` (Sonnet) — broad rapid sweep
- `general-purpose` (Opus) — deep architectural
- `rootdev-agents:kotlin-engineer` (Opus) — Kotlin specialist
- `rootdev-agents:mobile-security-coder` (Opus) — mobile security specialist

All findings below are convergent across 2+ critics or load-bearing in a single critic's analysis. Citations are file:line in the current branch state.

### CRITICAL — must fix before device deployment

#### C1. `BulletinPoller` destroys Phase 1's `system=prop` passive default

- Convergence: Sonnet adv + mobile-security + Opus deep
- Site: `BulletinPoller.kt:111-123` (`currentPatch()`), `BulletinPoller.kt:98-99` (newer check)
- Bug: `currentPatch()` filters out the literal `"prop"` value and returns null. The newer-check `current == null || date > current` makes every bulletin date "newer" when in passive mode. First successful poll calls `PatchLevelManager.updateTo(date)`, which overwrites `system=prop` with explicit dates. User who opted into passive gets active spoof.
- Fix: when `currentPatch()` would return null because of `system=prop`, fall back to `SystemProperties.get("ro.build.version.security_patch", "")` for the comparison. Only call `updateTo` if the bulletin date is genuinely newer than the live device prop AND the user wasn't in passive-only mode (consider an explicit opt-in flag for active mode).

#### C2. `PatchLevelManager.updateTo` has no future-date upper bound

- Convergence: all 4 critics
- Site: `PatchLevelManager.kt:55`
- Bug: code reads `if (today >= dateInt + MAX_PAST_OFFSET) return` which rejects dates more than ~1 year IN THE PAST. The plan prose at `fancy-humming-firefly.md:228` says "within 1 year future" — code does the opposite. A MITM injecting `<td>2099-12-31</td>` passes all validation and gets written to the cert.
- Fix: add `if (dateInt > today + 200) { log + return }` (~2 month future grace window covers pre-announced bulletins).

#### C3. sepolicy missing UDP rules for DNS resolution

- Convergence: mobile-security + Opus deep
- Site: `module/sepolicy.rule:4-9`
- Bug: only TCP rules present. `HttpsURLConnection` resolves the hostname via `getaddrinfo` → UDP port 53 first. Without UDP socket rules, DNS fails before TCP even attempts. Poller silently dies on enforcing SELinux kernels.
- Fix: append after existing rules:
  ```text
  allow ksu self:udp_socket { create connect read write getopt setopt }
  allow ksu port:udp_socket name_connect
  allow magisk self:udp_socket { create connect read write getopt setopt }
  allow magisk port:udp_socket name_connect
  ```

#### C4. `pollOnce` has no umbrella try/catch — single throw kills poller forever

- Convergence: Sonnet adv + Opus deep
- Site: `BulletinPoller.kt:38-42`
- Bug: body of `pollOnce` is `result = fetchAndParse(); appendHistory(result); scheduleNext(result.status == "success")`. `fetchAndParse` catches its own exceptions. `appendHistory` catches its own. But `scheduleNext` can throw `IllegalStateException` if the Looper is torn down. If anything in this chain throws, no reschedule happens and the poller is dead until reboot.
- Fix: wrap the entire body in `try { ... } catch (t: Throwable) { SystemLogger.error("BulletinPoller pollOnce failed", t); scheduleNext(false) }`.

#### C5. `BulletinPoller.start()` not wrapped in App.kt — failure becomes fatal

- Convergence: Sonnet adv
- Site: `App.kt:63` (call), `App.kt:61-64` (outer try/catch rethrows)
- Bug: plan at `fancy-humming-firefly.md:408` says "poller failure non-fatal — Phases 1/2/3 work without Phase 4." But `App.kt`'s outer `catch (e: Exception) { ...; throw e }` propagates everything, killing the daemon including keystore interception.
- Fix: wrap `BulletinPoller.start()` in its own try/catch that logs and continues.

### MAJOR — significant correctness or design issues

#### M1. `parsePatchLevelValue` synthesizes day=01 silently for 6-char input

- Convergence: Opus deep (single critic, but load-bearing)
- Site: `AndroidDeviceUtils.kt:346-350` (pre-existing code, NOT introduced this session)
- Bug: when `isLong=true` and input is YYYY-MM (6 chars), returns `year*10000 + month*100 + 1`. AOSP Tag.aidl says tags 718/719 are YYYYMMDD where the day field is significant. If `ro.vendor.build.security_patch` ever returns just YYYY-MM on the target device (older Samsung does this), the synthesized day=01 may disagree with the actual bulletin day.
- Action: verify on the user's test device whether `ro.vendor.build.security_patch` returns full YYYY-MM-DD. If it always returns full date, this is theoretical. If not, propagate null and fall back instead of synthesizing.

#### M2. `BootStateManager.apply()` runs AFTER `initializeInterceptors()`

- Convergence: mobile-security + Opus deep
- Site: `App.kt:46` (interceptor init) vs `App.kt:50` (BootStateManager call)
- Bug: keystore2 has already been hooked and may have cached `ro.boot.verifiedbootstate=orange` before BootStateManager spoofs it. Detectors that read via keystore2's binder calls during the window see the real value.
- Fix: move `BootStateManager.apply()` to be the FIRST init step after `prepareEnvironment()` at `App.kt:44`. Same for `PatchLevelManager.initialize()` if any process caches `ro.build.version.security_patch` at its own init.

#### M3. `PatchLevelManager.atomicWrite` has no try/catch

- Convergence: Opus deep
- Site: `PatchLevelManager.kt:86-96`
- Bug: `writeText` can throw IOException/SecurityException; `Files.move` can throw IOException/AtomicMoveNotSupportedException/FileSystemException. None caught. Exception propagates up through `updateTo`. When called from `BulletinPoller.fetchAndParse`, the broad `catch (e: Exception)` at `BulletinPoller.kt:104` swallows it and records `"network_error"` — wrong status.
- Fix: wrap `atomicWrite` in try/catch, log the actual error, return false. `updateTo` should still attempt resetprop independently or skip cleanly.

#### M4. `PIF_SOURCES.lastOrNull` semantics — plan prose contradicts plan table

- Convergence: Sonnet adv + mobile-security + Opus deep
- Site: `PatchLevelManager.kt:68`
- Issue: code uses `lastOrNull { it.exists() }` (later-wins). Plan table at `fancy-humming-firefly.md:197-206` implies later-wins (#2 overrides #1, #4 overrides #3). Plan prose at line 197 says "take first that exists" — contradicts the table.
- Decision needed: confirm with user which semantics they want. Probable intent matches code (custom.pif beats stock pif). Either keep code + fix plan prose, OR change to `firstOrNull`.
- Bonus: empty-file edge case — `lastOrNull` picks a zero-byte file if it exists, then `JSONObject("")` throws, caught, falls back to SystemProperties silently. Add `.filter { it.exists() && it.length() > 0 }`.

#### M5. Per-app patch overrides destroyed by atomicWrite

- Convergence: mobile-security
- Site: `PatchLevelManager.kt:86-95`
- Bug: overwrites entire `security_patch.txt` with only `system=$date\nboot=$date\nvendor=$date\n`. ConfigurationManager supports `[com.example.package]` sections (`ConfigurationManager.kt:259-261`); these are blown away on every poll.
- Fix: read existing file first, preserve `[pkg]` sections, only replace the global lines. Or document the regression as intentional (active spoof = uniform date across all packages).

#### M6. SELinux `node:tcp_socket node_bind` syntax disputed

- Convergence: Sonnet generic claims wrong; Opus deep verified against AOSP `fastbootd.te` and partially withdrew
- Site: `module/sepolicy.rule:5,8`
- Status: probably valid syntax but only confirmable on-device with `sesearch -A`. Low priority compared to C3.

#### M7. `/etc/gradle-init.d` poisons SSD if `/mnt/companion` unmounted

- Convergence: Sonnet generic + Opus deep
- Site: `/etc/gradle-init.d/per-user-builds.gradle.kts` (the `projectRoot.mkdirs()` call)
- Bug: if HDD mount fails (nofail in fstab), `mkdirs()` silently creates dirs on root fs, then once HDD remounts, real data is shadowed by the mount.
- Fix: add a check that `/mnt/companion` is actually a mountpoint (not just an empty dir) before mkdirs. Skip + log if not.

#### M8. Gradle 9/10 deprecation — `settingsEvaluated` + `beforeProject` may break

- Convergence: Sonnet generic + Opus deep
- Site: `/etc/gradle-init.d/per-user-builds.gradle.kts`
- Bug: `settingsEvaluated` deprecated since Gradle 7.6, may be removed in 10. `beforeProject { layout.buildDirectory.set(...) }` collides with Gradle's Isolated Projects feature.
- Action: pin Gradle version in `gradle/wrapper/gradle-wrapper.properties` (already at 9.2.0). Plan migration to settings plugin form before Gradle 10.

#### M9. Step 0 Commit B (EC KEY_SIZE omission) may be over-broad

- Convergence: Opus deep
- Site: `app/src/main/java/org/matrix/TEESimulator/interception/keystore/shim/KeyMintSecurityLevelInterceptor.kt:1071-1073`
- Issue: commit body claimed AOSP semantics require omitting redundant KEY_SIZE on EC keys. Opus argues real KeyMint TAs emit BOTH KEY_SIZE and EC_CURVE in characteristics list. The simulator's behavior may now differ from real hardware — a different forensic signal.
- Action: dump a real Pixel attestation cert with `openssl x509 -text`, check the auth list for tag 303 (KEY_SIZE) on an EC key with EC_CURVE present. If real hardware emits both, revert Commit B or add an "emit on EC" branch.

#### M10. PIF hot-reload not wired

- Convergence: Opus deep
- Site: `ConfigurationManager.kt:280` (ConfigObserver watches `/data/adb/tricky_store/` only)
- Bug: PIF lives at `/data/adb/modules/playintegrityfix/`. PatchLevelManager.initialize runs once at boot; edits to PIF JSON require a reboot to take effect.
- Fix: add a second FileObserver in PatchLevelManager that watches the PIF dir and triggers `initialize()` on change. OR document "reboot to refresh PIF."

### MINOR + SUGGESTION (worth knowing, not blocking)

- HTTPS no cert pinning + captive portal HTML containing `<td>YYYY-MM-DD</td>` accepted — security hardening (mobile-security + Opus). Pin Google's GTS Root SPKI. Add `User-Agent` that mimics a real browser instead of `TEESimulator/version`.
- `BulletinPoller.start()` double-call leaks the first HandlerThread (Sonnet adv). Add an idempotent guard.
- `@Volatile` on `bootstrapStep` + `steadyArmed` is decorative — fields only mutated from single HandlerThread (Sonnet adv + Opus). Either drop the annotation or document defensively.
- `appendHistory` JSON `optString` returns literal `"null"` on JSON null (Sonnet adv + Opus). Use `if (obj.isNull("field")) null else obj.optString(...)`.
- `latest_known_date` uses `lastOrNull` (insertion order) not `maxByOrNull` over actual dates (Opus). Regresses if dates arrive out of chronological order.
- `scripts/package.sh:17` — `$HOME` empty under `sudo --reset-env`. The PATH fix silently no-ops in restricted sudo (Sonnet generic).
- `scripts/package.sh --verify` doesn't grep for `BootStateManager|PatchLevelManager|BulletinPoller` in logcat, so it cannot confirm the new managers actually ran (Opus).
- Commit subject lengths: `7abdba9`, `acdf452`, `14ec9c3` are 44-57 chars (some over the 50 limit). Not amend-worthy retroactively; note for future commits.
- `uninstall.sh` missing cleanup for `last_bulletin_fetch.json.next` staging file (mobile-security + Opus).
- BootStateManager `linkedMapOf` is theater — `mapOf` would do (Sonnet adv).
- `appendHistory` sets `applied=isNewer` before `updateTo` returns; if `updateTo` rejects the date, history records `applied=true` (Sonnet generic).
- Cargo PATH symlinks pre-laid for rootdev only. Other users (thinker, president) already have their own setups. Future new users (planner, claudetest) need to either log in (profile.d auto-lays cargo PATH) or manually source `/etc/profile.d/cargo-path.sh` mid-session.

### Detection-surface gaps (out of plan scope but flagged)

These weren't in the 4-phase plan but the mobile-security critic flagged them for completeness:

- ATTESTATION_ID_BRAND/DEVICE/PRODUCT/MANUFACTURER/MODEL (tags 710-716) are caller-supplied, never reconciled against device props. Brand-spoofing PIF doesn't flip `Build.MANUFACTURER` for the attesting app.
- ATTESTATION_APPLICATION_ID (tag 709) integrity — no consistency check against IPackageManager-resolved signature.
- `/proc/cmdline` is untouched — detectors reading raw kernel cmdline see real `ro.boot.*` values regardless of resetprop.
- `boot_hash.bin` mtime observable — `AndroidDeviceUtils.kt:191-202` writes after install, real vbmeta digest is fixed at flash time.
- ABI coverage: cargo-ndk currently only builds aarch64. armv7/x86/x86_64 devices fall back to AOSP cert path.

## Decisions made this session

| # | Decision | Rationale |
|---|---|---|
| D1 | Use OMC TaskCreate (not tasks/todo.md) for the 24-task tracker | TaskCreate provides dependency wiring + persistence the markdown doesn't |
| D2 | Audit gates as explicit tasks between phases | User asked for self-audit to catch mistakes |
| D3 | Anchor-by-content (not line number) in Phase impl tasks | Each phase shifts subsequent App.kt line numbers |
| D4 | Strict serial chain via `blockedBy` | Per user's "lethal precision" request |
| D5 | Defer device tests rather than block | No ADB device this session; coding work can proceed independently |
| D6 | Relay project-tree build symlinks to rootdev namespace | User chose option B in the multi-user collision prompt; trade-off accepted |
| D7 | Install Gradle init script at `/etc/` for all users | User chose "All users via /etc/profile.d or shared init.d" — durable fix |
| D8 | Install rust via rustup into `~/.cargo` symlinked to HDD | Matches CLAUDE.md per-user namespace pattern |
| D9 | Add PATH line to `scripts/package.sh` as a separate `chore:` commit | Build infrastructure fix; keeps phase commits clean |
| D10 | NOT to fix critic findings same session | User requested handoff doc instead; next session decides priority |

## Open questions for next session

1. **M4 PIF semantics:** user must confirm whether to keep `lastOrNull` (later-wins matches plan table) or switch to `firstOrNull` (matches plan prose). Recommended: keep `lastOrNull`, fix plan prose.
2. **M5 per-app overrides:** acceptable regression or load-bearing for the user's use case? User must decide.
3. **M9 EC KEY_SIZE:** does user have a Pixel cert dump handy to validate the assumption that real KeyMint emits BOTH KEY_SIZE and EC_CURVE? If not, leave Commit B as-is until evidence available.
4. **Detection-surface gaps:** out-of-scope for the original 4-phase plan, but the user may want a follow-up plan addressing the brand/model/AAID/cmdline gaps.
5. **Active vs passive mode:** the C1 fix needs a design call. Options: (a) BulletinPoller respects `system=prop` and stays passive, (b) explicit opt-in flag for active mode, (c) drop BulletinPoller for users who want pure-passive.

## Critic fixes — 2026-05-19 continuation

The second pass on 2026-05-19 landed 12 commits resolving every CRITICAL and 8 of 10 MAJOR critic findings. M6 has a defensive recipe below; M8 is captured as a separate migration plan.

### Design decisions confirmed

| # | Question | Choice | Rationale |
|---|---|---|---|
| D11 | C1 poller behavior in passive mode | Respect `system=prop` (passive-safe) | Smallest blast radius; preserves Phase 1 default intent. The file becomes user-owned config; props track PIF or device default. |
| D12 | M4 PIF resolution semantics | Keep `lastOrNull` (custom wins) | Hand-edited custom file overrides stock; matches the existing code and the most common user mental model. |
| D13 | M9 EC KEY_SIZE | Revert d34630f | AOSP 15 KeyMint reference TA emits both `KEY_SIZE` and `EC_CURVE` for EC keys (research summary below). Omitting `KEY_SIZE` made the simulator's characteristics list shorter than real hardware — a detection fingerprint. |

### M9 AOSP research summary

Evidence path traced in `/mnt/companion/sources/aosp-android-15-6.6/`:

- `system/keymint/ta/src/keys.rs:267-316` — `Operation::generate_key` → `generate_key_material` → `tag::extract_key_gen_characteristics`.
- `system/keymint/common/src/tag.rs:270-377` — `extract_key_characteristics` filters input `KeyParam`s into the output list via `KEYMINT_ENFORCED_CHARACTERISTICS` membership check at lines 355-359.
- `system/keymint/common/src/tag/info.rs:61-89` — `KEYMINT_ENFORCED_CHARACTERISTICS` set contains both `Tag::EcCurve` (line 64) and `Tag::KeySize` (line 72).
- `system/keymint/common/src/tag.rs:623-649` — `check_ec_params` comment at line 632: "Key size is not needed, but if present should match the curve." The TA validates but never strips `KeySize`.
- `hardware/interfaces/security/keymint/aidl/vts/functional/KeyMintBenchmark.cpp:234,259` — EC key generation supplies both `TAG_KEY_SIZE` and `TAG_EC_CURVE` simultaneously; the TA returns both unchanged.

Conclusion: real KeyMint emits both. d34630f reverted in commit `9b982d4`.

### Commits landed this pass

| Commit | Type | What |
|---|---|---|
| 292f24a | fix(spoof) | Bound future patch dates to ~60 days (C2) |
| 9d693bc | fix(spoof) | Allow UDP egress for DNS resolution (C3) |
| 1ee2346 | fix(spoof) | Wrap pollOnce in umbrella try/catch (C4) |
| 311523e | fix(spoof) | Isolate BulletinPoller.start failure (C5) |
| b50d313 | fix(spoof) | Order spoofers before keystore hook (M2) |
| 47222ab | fix(spoof) | Respect system=prop passive default (C1) |
| d14db7d | fix(spoof) | Preserve [pkg] sections in atomicWrite (M5) |
| 5a519b9 | fix(spoof) | Guard atomicWrite errors in updateTo (M3) |
| c111bd0 | fix(spoof) | Skip empty PIF source files (M4) |
| c589e12 | feat(spoof) | Hot-reload PIF via FileObserver (M10) |
| 9b982d4 | fix(interception) | Re-emit KEY_SIZE for EC keys (M9) |
| 49b7bfc | fix(util) | Skip day synthesis for YYYY-MM input (M1) |

### M6 defensive recipe (sepolicy `node:*_socket node_bind`)

The current `module/sepolicy.rule` (after `9d693bc`) uses `node:*_socket node_bind` for both ksu and magisk on TCP and UDP. The critic dispute was over whether this syntax compiles on every device's policy compiler. The Opus critic verified the form against AOSP `fastbootd.te` and partially withdrew the objection.

Fallback recipe if device test #23 surfaces AVC denials referencing `node:tcp_socket node_bind` or `node:udp_socket node_bind` that fail to compile:

```text
allow keystore {adb_data_file shell_data_file} file *
allow crash_dump keystore process *

allow ksu self:tcp_socket { create connect read write getopt setopt }
allow ksu port:tcp_socket name_connect
allow magisk self:tcp_socket { create connect read write getopt setopt }
allow magisk port:tcp_socket name_connect

allow ksu self:udp_socket { create connect read write getopt setopt }
allow ksu port:udp_socket name_connect
allow magisk self:udp_socket { create connect read write getopt setopt }
allow magisk port:udp_socket name_connect
```

This drops the four `node:*_socket node_bind` lines. `node_bind` is only required when an app `bind()`s to a local address before `connect()` — HttpsURLConnection client paths do not bind, so removing it should not break outbound traffic. Apply only if denials appear; otherwise leave the current rules in place.

### M8 — Gradle init script modernization

`settingsEvaluated` + `beforeProject` are deprecated since Gradle 7.6 and slated for removal in Gradle 10. The wrapper is currently pinned at 9.2.0 (`gradle/wrapper/gradle-wrapper.properties`), so no immediate break. Migration is captured in a separate plan at `.omc/plans/gradle-init-script-modernization-2026-05-19.md`.

### Still deferred (not in this session's scope)

- **M6** — sepolicy syntax verification on-device (see recipe above).
- **M8** — Gradle init script migration (see plan above).
- **All 5 device tests** (#11, #15, #19, #23, #24) — ADB device still absent.

### Validation

- `./gradlew :app:compileDebugKotlin` clean after every commit (sub-second incrementals).
- Final smoke build via `./scripts/package.sh --debug`: `TEESimulator-RS-v6.0.0-190-Debug.zip` (12,379,724 bytes), 14 s incremental, clean.
- No device tests possible — ADB still absent.

## Second adversarial review — 2026-05-19

A second adversarial pass was dispatched after the 12 critic-fix commits landed: `general-purpose` at sonnet (broad cross-cut) and `rootdev-agents:kotlin-engineer` at opus (deep Kotlin/Android architecture). Five additional defects surfaced and were fixed; two were documented as known limitations.

### Findings and resolutions

| # | Source | Finding | Resolution | Commit |
|---|---|---|---|---|
| F1 | Opus | `isGlobalKeyAssignment` stripped any bare line whose first token matched a global key, eating malformed `system`/`all` shorthand on next atomicWrite | Require `'=' in trimmed` before treating as assignment | `a4af3d5` |
| F2 | Opus | `mergedContents` `runCatching` silently rewrote with global block only on read failure, destroying `[pkg]` overrides | Drop the `runCatching` so IO errors propagate to atomicWrite -> updateTo's M3 try/catch, which logs and leaves file + props untouched | `0e752f8` |
| F4 | Opus | `applyToProps` racy across main, BulletinPoller handler thread, and PifObserver inotify thread; concurrent resetprop for same prop with different dates could leave system/vendor mismatched | `@Synchronized` on the singleton's applyToProps | `1c64688` |
| F10 | Opus | `MAX_FUTURE_DAYS = 60L` lacked rationale | KDoc explaining Pixel bulletin cadence + slip | `84161da` |
| F14 | Sonnet | `currentPatch` returned raw `system=` value unvalidated; a malformed `system=tomorrow` made `date > current` always false and permanently suppressed updates | Validate against date pattern; log warning and treat as passive on mismatch | `3bf58de` |

### Known limitations (documented, not fixed)

- **F3** (Opus): when `security_patch.txt` is absent or in passive mode at first successful poll, BulletinPoller arms the 24 h steady cadence. A user who switches to active mode minutes later waits up to 24 h for the next bulletin compare. Workaround: reboot to restart the bootstrap cycle (next compare in 5 s). A proper fix (`FileObserver` on `PATCH_FILE` to re-arm on user edit) was deemed scope-creep for this pass.
- **F13** (Sonnet): the PIF hot-reload `FileObserver` only registers when `/data/adb/modules/playintegrityfix` exists at boot. Installing PIF after the daemon boots will not trigger hot-reload until next boot. A parent-directory observer on `/data/adb/modules` for `CREATE` would close this gap, but is over-engineered for the typical install order (PIF first, TEESimulator second).

### Findings explicitly NOT acted on (with reasoning)

- **Opus F5** — claimed M1's null-propagation was defeated by `toPatchLevelInt`'s `?: 20240401`. False: `getRealDevicePatchLevelInt`'s `?.let { return }` catches null BEFORE the extension fallback, falling through to `Build.VERSION.SECURITY_PATCH` (always YYYY-MM-DD). M1 works for its intended vendor-prop case.
- **Opus F6** — claimed `fetchAndParse` catching only `Exception` strands the poller on `Throwable`. C4's outer `pollOnce` already catches `Throwable` and calls `scheduleNext(false)`. Only history record is missed; thread survives and reschedules.
- **Opus F7-F9** — pre-existing comment/style issues in `ConfigurationManager.kt` and `App.kt`. Out of session scope.
- **Opus F8** — `SystemProperties.get(..., Build.VERSION.SECURITY_PATCH)` is functionally equivalent to `Build.VERSION.SECURITY_PATCH`. Cosmetic; touching it is scope creep.
- **Opus F11** — "UDP block" naming claim. "Block" reads as noun (group of rules), not verb. Semantics correct.
- **Opus F12** — `latestKnown` uses `lastOrNull` instead of `maxOrNull`. Pre-existing minor (also flagged in the prior critic pass); out of CRITICAL+MAJOR scope.

## File index — everything touched this session

### Created (4 files in repo + 3 system files)

```text
/home/president/Git-repo-success/TEESimulator/app/src/main/java/org/matrix/TEESimulator/config/BootStateManager.kt
/home/president/Git-repo-success/TEESimulator/app/src/main/java/org/matrix/TEESimulator/config/PatchLevelManager.kt
/home/president/Git-repo-success/TEESimulator/app/src/main/java/org/matrix/TEESimulator/config/BulletinPoller.kt
/home/president/Git-repo-success/TEESimulator/.omc/plans/self-sufficient-spoofing-handoff-2026-05-19.md   (this file, renamed from -session-handoff during audit pass)
/etc/gradle-init.d/per-user-builds.gradle.kts                                                            (system)
/etc/profile.d/gradle-per-user-init.sh                                                                   (system)
/etc/profile.d/cargo-path.sh                                                                             (system)
```

### Modified (6 files)

```text
/home/president/Git-repo-success/TEESimulator/module/customize.sh                                       (Phase 1)
/home/president/Git-repo-success/TEESimulator/app/src/main/java/org/matrix/TEESimulator/util/AndroidDeviceUtils.kt   (Phase 2 setProperty overload)
/home/president/Git-repo-success/TEESimulator/app/src/main/java/org/matrix/TEESimulator/App.kt                       (Phases 2/3/4 imports + init calls)
/home/president/Git-repo-success/TEESimulator/module/sepolicy.rule                                      (Phase 4)
/home/president/Git-repo-success/TEESimulator/module/uninstall.sh                                       (Phase 4)
/home/president/Git-repo-success/TEESimulator/scripts/package.sh                                        (cargo PATH chore)
```

### Pre-existing references cited in the work (verified against current branch state)

```text
app/src/main/java/org/matrix/TEESimulator/config/ConfigurationManager.kt:253-256   (system=prop force-override)
app/src/main/java/org/matrix/TEESimulator/config/ConfigurationManager.kt:280       (ConfigObserver)
app/src/main/java/org/matrix/TEESimulator/util/AndroidDeviceUtils.kt:148-165       (private setProperty ByteArray variant — pattern source)
app/src/main/java/org/matrix/TEESimulator/util/AndroidDeviceUtils.kt:167-183       (NEW internal setProperty String variant — Phase 2)
app/src/main/java/org/matrix/TEESimulator/util/AndroidDeviceUtils.kt:336-357       (parsePatchLevelValue — Phase 3 cert encoding handler)
app/src/main/java/org/matrix/TEESimulator/util/AndroidDeviceUtils.kt:346-350       (parsePatchLevelValue 6-char fallback — see critic finding M1)
app/src/main/java/org/matrix/TEESimulator/interception/keystore/shim/KeyMintSecurityLevelInterceptor.kt:1071-1073   (Step 0 Commit B EC KEY_SIZE guard)
```

### AOSP reference paths used

```text
/mnt/companion/sources/aosp-android-15-6.6/hardware/interfaces/security/keymint/aidl/android/hardware/security/keymint/Tag.aidl
  Lines 588-606  OS_PATCHLEVEL = TagType.UINT | 706, YYYYMM
  Lines 784-804  VENDOR_PATCHLEVEL = TagType.UINT | 718, YYYYMMDD
  Lines 806-824  BOOT_PATCHLEVEL = TagType.UINT | 719, YYYYMMDD
/mnt/companion/sources/aosp-android-15-6.6/system/security/keystore2/src/key_parameter.rs:964,1003,1006
  Confirms UINT field type for all three patchlevel tags
```

## Outputs

```text
out/TEESimulator-RS-v6.0.0-175-Debug.zip   (rootdev, 12,379,380 bytes — built without rust step)
out/TEESimulator-RS-v6.0.0-176-Debug.zip   (rootdev, 12,379,498 bytes — full pipeline with fresh cargo)
```

The version number is `gitCommitCount` (175 = after my 4 phase commits; 176 = after the chore commit). `module/update.json` was auto-rewritten by the Step 0 Commit E `refreshUpdateJson` gradle task on every package run.

## What NOT to touch in next session

- The 13 commits on `feat/self-sufficient-spoofing` are landed. Do not amend any of them (project hygiene: NEVER amend published commits).
- `/etc/gradle-init.d/` and `/etc/profile.d/*.sh` are durable system-level fixes. Don't relay symlinks in project trees again — the init script handles it. The exception is `app/.cxx` which is NDK-managed, not Gradle-managed (see the Project tree cleanup subsection above for the relay command if needed).
- The project-tree `.gradle`, `build`, `app/build`, `native-certgen/target` paths are GONE. The init script puts them under `/mnt/companion/$USER/builds/TEESimulator/`. Do not recreate symlinks in the project tree for those four.
- `~/.cargo` and `~/.rustup` symlinks for rootdev are laid correctly to `/mnt/companion/rootdev/caches/{cargo,rustup}`. Don't touch.
- `local.properties` is thinker's redirect to thinker's SDK because president's NDK sysroot is partially corrupted. Don't change it unless rootdev (or whoever) installs their own Android SDK + NDK.

## Suggested first action next session

1. Read this file end-to-end, with focus on "Critic fixes — 2026-05-19 continuation" for the most recent state.
2. Run `git log --oneline feat/self-sufficient-spoofing -26` to confirm the branch state matches the table above.
3. Run `git status --short` and cross-reference the "Untracked working-tree state" section so the noise doesn't trigger false-positive cleanup.
4. With all CRITICAL + 8 of 10 MAJOR critic findings resolved, the next logical step is the deferred device-test set (#11, #15, #19, #23, #24). Connect an ADB device and run `./scripts/package.sh --release --deploy --clear-keys --reboot --verify`, then walk each per-phase checklist in "Deferred device-test step lists" above.
5. If `TaskList` returns empty for the device tasks (session ID changed), the step lists are embedded in this file under "Deferred device-test step lists."
6. M6 (sepolicy `node:tcp_socket` syntax) must be confirmed during task #23 via `sesearch -A` or AVC denial logs. If denials surface, the UDP block landed in `9d693bc` plus the existing TCP block should be reviewed against the device's policy compiler version.
