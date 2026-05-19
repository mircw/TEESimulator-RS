# Self-Sufficient Spoofing — Session Handoff

Authoring session date: 2026-05-19
Branch: `feat/self-sufficient-spoofing`
Source plan: `/home/rootdev/.claude/plans/fancy-humming-firefly.md`
Status: 5 phase commits landed, full debug ZIP builds clean, all device tests deferred, 5 CRITICAL + 10 MAJOR critic findings pending decision and fix.

## Resume protocol for next session

Read in order:

1. This document end-to-end.
2. `/home/rootdev/.claude/plans/fancy-humming-firefly.md` (source plan).
3. `git log --oneline feat/self-sufficient-spoofing -12` to confirm branch state matches the table below.
4. The four critic findings sections below — every unresolved finding has a file:line citation and a proposed fix.

Then ask the user which findings to prioritize before writing code.

## Branch state

```text
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
e0105d6 wip(keystore): add F1 Phase A diagnostic logs in updateAad path  (pre-existing on fix/bhim-regression base)
```

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
- Modified: `app/src/main/java/org/matrix/TEESimulator/App.kt` (added import + `BootStateManager.apply()` call at line 50)
- Behavior: at every boot, resetprops `ro.boot.verifiedbootstate=green`, `ro.boot.flash.locked=1`, `ro.boot.veritymode=enforcing`. Skips work when current already matches target.

### Phase 3 — PatchLevelManager
- `acdf452 feat(spoof): PatchLevelManager with PIF resolution`
- New file: `app/src/main/java/org/matrix/TEESimulator/config/PatchLevelManager.kt`
- Modified: `App.kt` (added import + `PatchLevelManager.initialize()` at line 51)
- Behavior: resolves the active patch date from PlayIntegrityFix via 6-path override chain (`PatchLevelManager.kt:23-30`), falls back to `SystemProperties.get("ro.build.version.security_patch", Build.VERSION.SECURITY_PATCH)`. Validates YYYY-MM-DD format, rejects dates `< 20200101` or more than `MAX_PAST_OFFSET = 10000` (~1 year) in the past. On accept: atomic stage-and-rename `security_patch.txt` with `system=$date\nboot=$date\nvendor=$date\n`, then resetprops both system and vendor patch props.

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

`App.kt:33-65` flow:

1. `SystemLogger.info("Welcome to TEESimulator!")`
2. `Thread.setDefaultUncaughtExceptionHandler { ... }` (logs only)
3. `prepareEnvironment()`
4. `initializeInterceptors()` (blocks until keystore2 hook attaches)
5. `ConfigurationManager.initialize()`
6. `BootStateManager.apply()` (Phase 2)
7. `PatchLevelManager.initialize()` (Phase 3)
8. `AndroidDeviceUtils.setupBootKeyAndHash()`
9. BouncyCastle provider swap
10. `NativeCertGen.initialize("/data/adb/modules/tricky_store/libcertgen.so")`
11. `BulletinPoller.start()` (Phase 4)
12. `Looper.loop()` (blocks forever)

Critic finding M2 below proposes moving BootStateManager + PatchLevelManager to step 3 (before `initializeInterceptors`) to close a race where keystore2 caches the un-spoofed values during its init.

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

Then per-phase checks per each task's description in the OMC task system (run `TaskGet` on tasks 11/15/19/23/24 in next session to retrieve full step lists).

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

## File index — everything touched this session

### Created (4 files in repo + 3 system files)

```text
/home/president/Git-repo-success/TEESimulator/app/src/main/java/org/matrix/TEESimulator/config/BootStateManager.kt
/home/president/Git-repo-success/TEESimulator/app/src/main/java/org/matrix/TEESimulator/config/PatchLevelManager.kt
/home/president/Git-repo-success/TEESimulator/app/src/main/java/org/matrix/TEESimulator/config/BulletinPoller.kt
/home/president/Git-repo-success/TEESimulator/.omc/plans/self-sufficient-spoofing-session-handoff.md   (this file)
/etc/gradle-init.d/per-user-builds.gradle.kts                                                          (system)
/etc/profile.d/gradle-per-user-init.sh                                                                 (system)
/etc/profile.d/cargo-path.sh                                                                           (system)
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

### Pre-existing references cited in the work

```text
app/src/main/java/org/matrix/TEESimulator/config/ConfigurationManager.kt:253-256   (system=prop force-override)
app/src/main/java/org/matrix/TEESimulator/config/ConfigurationManager.kt:280       (ConfigObserver)
app/src/main/java/org/matrix/TEESimulator/util/AndroidDeviceUtils.kt:148-165       (private setProperty ByteArray variant — pattern source)
app/src/main/java/org/matrix/TEESimulator/util/AndroidDeviceUtils.kt:318-330       (parsePatchLevelValue — Phase 3 cert encoding handler)
app/src/main/java/org/matrix/TEESimulator/util/AndroidDeviceUtils.kt:346-350       (parsePatchLevelValue 6-char fallback — see M1)
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

- The 11 commits on `feat/self-sufficient-spoofing` are landed and audited. Do not amend.
- `/etc/gradle-init.d/` and `/etc/profile.d/*.sh` are durable system-level fixes. Don't relay symlinks in project trees again — the init script handles it.
- The project-tree `.gradle`, `build`, `app/build`, `native-certgen/target` paths are GONE. The init script puts them at `/mnt/companion/$USER/builds/TEESimulator/`. Do not recreate symlinks in the project tree.
- `~/.cargo` and `~/.rustup` symlinks for rootdev are laid correctly. Don't touch.

## Suggested first action next session

1. Read this file.
2. Run `git log --oneline feat/self-sufficient-spoofing -12` to confirm branch state matches.
3. Ask user which critic finding to address first. Recommended priority: C1 (passive default destruction) before any device test, because deferred Phase 1 test (#11) will pass on first deploy and silently fail on second deploy when the bulletin poller runs.
4. Use `TaskList` to see the 5 pending device-test tasks; treat them as the success criteria after fixes land.
