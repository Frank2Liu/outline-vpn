# Development Guideline — Eclipse on Windows (Local)

Step-by-step guide to set up a **local Windows environment** for developing the
`nthlink-os-android` project with **Eclipse** — including required packages, the dependency
libraries the project needs, the Gradle (Buildship) import flow, and Windows-specific
troubleshooting.

---

## 0. What this guide gives you

By the end you will be able to:

- Import the project as a Gradle project in Eclipse.
- Build `app-debug.apk` from Eclipse or the terminal (same Gradle wrapper, identical output).
- Run unit tests.
- Install and launch the app on a device / emulator and read `adb logcat` logs.
- Edit Kotlin/Java/XML with basic tooling.

> **Important reality check:** Eclipse has no first-class **Kotlin *Android*** toolchain. The
> JetBrains Kotlin plugin gives you syntax/language help, but the *authoritative* compiler is the
> Gradle build (Kotlin 2.2.21 + KSP + SafeArgs run inside Gradle, not inside Eclipse). Full
> on-device breakpoint debugging is not available in Eclipse — use `adb logcat` instead, or switch
> to Android Studio for deep debugging.

---

## 1. Prerequisites — Windows

| Item | Requirement | Why |
|---|---|---|
| OS | Windows 10 / 11 64-bit | dev host |
| RAM / disk | 8 GB+ RAM, ~10 GB free disk | Gradle + SDK + Eclipse |
| Java | **JDK 17** (64-bit) | required by AGP 8.13.0 |
| Eclipse | **Eclipse IDE for Java Developers** (2024-09 or newer, 64-bit) | hosts Buildship + Kotlin plugin |
| Android SDK | **platform 36**, **build-tools 36.x**, platform-tools/adb | project uses `compileSdk 36`, `minSdk 26` |
| Gradle | bundled **wrapper** (`gradlew.bat`) — no manual install | provided by the repo |
| Device/emulator | Android 8.0+ device, or an AVD | running the app |

Dependency libraries are **not** installed manually — Gradle resolves all of them at build time from
Google Maven / Maven Central / JitPack:

- AndroidX: core-ktx 1.17.0, appcompat 1.7.1, constraintlayout 2.2.1, lifecycle-runtime-ktx 2.9.4,
  navigation 2.9.5, datastore-preferences 1.1.4, drawerlayout 1.2.0, work 2.11.0, room 2.8.3.
- Material Components 1.13.0 · Play core review-ktx 2.0.2 · app-update-ktx 2.1.0 ·
  kotlinx-serialization-json 1.9.0 · kotlinx-coroutines-play-services 1.10.2 ·
  Koin (koin-android) 4.1.1.
- Tests: JUnit 4, androidx.test ext-junit 1.3.0, espresso-core 3.7.0.
- Build plugins: AGP 8.13.0, Kotlin 2.2.21 (+ serialization plugin), KSP 2.2.21-2.0.4,
  Navigation SafeArgs 2.9.5.

Versions are pinned in `build.gradle` / `app/build.gradle` / `core/build.gradle`.

---

## 2. Step 1 — Install JDK 17

Download an **Eclipse Temurin (Adoptium) JDK 17 Windows x64** build from
https://adoptium.net/temurin/releases/?version=17 and install (`.msi`). Note the install path, e.g.:

```text
C:\Program Files\Eclipse Adoptium\jdk-17.0.x.x-hotspot
```

Verify in a new terminal:

```powershell
java -version
```

Output must show `openjdk version "17.0.x"`. Later (Step 5) we point Eclipse at this JDK — it must
be a **JDK**, not a JRE.

---

## 3. Step 2 — Install the Android SDK (command-line tools)

```powershell
$sdk = "$env:LOCALAPPDATA\Android\Sdk"
New-Item -ItemType Directory -Force -Path $sdk | Out-Null

# Download "command-line tools only" from:
#   https://developer.android.com/studio#command-line-tools-only
# (commandlinetools-win-*.zip) → extract with the folder renamed to "latest":
New-Item -ItemType Directory -Force -Path "$sdk\cmdline-tools" | Out-Null
Expand-Archive -Path "$env:USERPROFILE\Downloads\commandlinetools-win-*.zip" -DestinationPath "$sdk\cmdline-tools" -Force
Rename-Item "$sdk\cmdline-tools\cmdline-tools" "$sdk\cmdline-tools\latest"

# Install the packages and accept licenses
& "$sdk\cmdline-tools\latest\bin\sdkmanager.bat" "platform-tools" "platforms;android-36" "build-tools;36.0.0"
& "$sdk\cmdline-tools\latest\bin\sdkmanager.bat" --licenses     # answer 'y' until finished
```

If `sdkmanager.bat` fails with a Java error, set `JAVA_HOME` (Step 3) first and retry.

---

## 4. Step 3 — Environment variables (per-user)

```powershell
$sdk = "$env:LOCALAPPDATA\Android\Sdk"
$javaHome = "C:\Program Files\Eclipse Adoptium\jdk-17.0.x.x-hotspot"   # adjust

[Environment]::SetEnvironmentVariable("JAVA_HOME", $javaHome, "User")
[Environment]::SetEnvironmentVariable("ANDROID_HOME", $sdk, "User")
[Environment]::SetEnvironmentVariable("ANDROID_SDK_ROOT", $sdk, "User")

# adb on PATH
$oldPath = [Environment]::GetEnvironmentVariable("Path", "User")
$newPath = "$sdk\platform-tools;$sdk\cmdline-tools\latest\bin" + ($(if ($oldPath) { ";$oldPath" } else { "" }))
[Environment]::SetEnvironmentVariable("Path", $newPath, "User")
```

Verify in a **new** terminal: `java -version`, `adb --version`, `echo $env:JAVA_HOME`.

---

## 5. Step 4 — Clone + `local.properties`

```powershell
git clone <your-repo-url> "D:\OpenSource\nthlink-os-android"
cd "D:\OpenSource\nthlink-os-android"

# sdk.dir must be a valid SDK path; backslashes escaped (or use forward slashes)
echo "sdk.dir=C\:\\Users\\<YourUserName>\\AppData\\Local\\Android\\Sdk" | Out-File -Encoding ascii "D:\OpenSource\nthlink-os-android\local.properties"
```

Do **not** commit `local.properties` (machine-specific path).

Quick sanity build before touching Eclipse (proves the toolchain works):

```powershell
.\gradlew.bat :app:assembleDebug
.\gradlew.bat :app:testDebugUnitTest :core:testDebugUnitTest
```

APK output: `app\build\outputs\apk\debug\app-debug.apk`.

---

## 6. Step 5 — Install / configure Eclipse with JDK 17

1. Download **Eclipse IDE for Java Developers** (64-bit) from https://eclipseide.org and install
   (simplest: unzip / run the installer).
2. Launch once so it creates a workspace, then exit.
3. Point Eclipse at the JDK by editing `eclipse.ini` next to `eclipse.exe` — insert `-vm` lines at
   the **top** of the file, before `-vmargs`:

   ```ini
   -vm
   C:/Program Files/Eclipse Adoptium/jdk-17.0.x.x-hotspot/bin/javaw.exe
   -vmargs
   -Dosgi.requiredJavaVersion=17
   -Xms512m
   -Xmx2048m
   ```

4. Relaunch Eclipse. `Help → About Eclipse IDE` should show Java 17 runtime.
5. Inside Eclipse also register the JDK: `Window → Preferences → Java → Installed JREs → Add →
   Standard VM` → point to `C:\Program Files\Eclipse Adoptium\jdk-17.0.x.x-hotspot` → make it the
   default.

---

## 7. Step 6 — Install the Gradle and Kotlin plugins

`Help → Eclipse Marketplace…` and install:

| Plugin | Purpose | Note |
|---|---|---|
| **Buildship: Eclipse Plug-ins for Gradle** | Import & run Gradle projects from Eclipse | Already bundled in most recent "Java Developers" distributions; install only if missing |
| **Kotlin Plugin for Eclipse** (JetBrains) | Kotlin syntax/semantic help | Editor support is limited vs IntelliJ; the Gradle build remains authoritative |

After installing, restart Eclipse when prompted (`Help → Check for Updates` first is a good idea).

Optional legacy note: the old **Android Development Tools (ADT)** is deprecated and irrelevant here —
do **not** install it. All Android builds go through Gradle/Buildship.

---

## 8. Step 7 — Configure Gradle & SDK in Eclipse

1. `Window → Preferences → Gradle` set:
   - **Gradle distribution**: "Gradle wrapper (from project)" (recommended) — the repo pins the
     wrapper version.
   - **Gradle JVM home**: point to the JDK 17 path above.
   - Optionally tick **Show console view when running tasks**.
2. `Window → Preferences → General → Workspace` — set **Text file encoding** to `UTF-8` (the project
   defaults to UTF-8; avoid mojibake in `strings.xml`, especially non-Latin locales).
3. Confirm `ANDROID_HOME` / `local.properties` are present — Buildship forwards both to Gradle
   automatically (Steps 3–4). Gradle reads `sdk.dir` from `local.properties` in the project root.

---

## 9. Step 8 — Import the project

1. `File → Import…` → **Gradle → Existing Gradle Project** → **Next**.
2. **Project root directory**: `D:\OpenSource\nthlink-os-android` → **Next**.
3. Recommended import options (leave defaults):
   - Override workspace settings hashes: **Clear project configuration** (when first importing)
   - Advanced: Gradle distribution = **Gradle wrapper (from project)**
   - JVM: your JDK 17.
4. **Finish**. Buildship reads `settings.gradle` and discovers **`:app`** and **`:core`** modules.
   - The `outline/` directory is a standalone AAR folder, **not** a Gradle module — it will (and
     should) not appear; that is expected.
5. First import downloads all dependencies. Wait for the Gradle sync to finish (progress bottom
   right / Gradle Tasks view populating).

> If you later change `build.gradle`, right-click the project → **Gradle → Refresh Gradle Project**
> (or `Alt+F5`).

---

## 10. Step 9 — Build, test, install from Eclipse

### Build the debug APK

- Open the **Gradle Tasks** view: `Window → Show View → Other… → Gradle → Gradle Tasks`.
- Expand `nthlink6-android` → `:app` → `build` → double-click **assembleDebug**.
- Or right-click the project → **Gradle → Run Gradle Tasks**… → enter `:app:assembleDebug`.

### Run unit tests

- Gradle Tasks view → `:app` → `verification` → **testDebugUnitTest**; repeat for `:core`.
- Or run `.\gradlew.bat :app:testDebugUnitTest :core:testDebugUnitTest` in a terminal.

### Alternative: always use the terminal

Everything Eclipse runs in Gradle is identical to:

```powershell
.\gradlew.bat :app:assembleDebug
.\gradlew.bat :app:installDebug
```

---

## 11. Step 10 — Run on a device / emulator

```powershell
# USB device
adb devices                                                 # must show "device"
.\gradlew.bat :app:installDebug                            # build + install
adb shell am start -n com.nthlink.android.client/.ui.LaunchActivity

# Optional AVD
& "$env:ANDROID_HOME\cmdline-tools\latest\bin\avdmanager.bat" create avd -n nthlink_pixel -k "system-images;android-36;google_apis;x86_64" --device "pixel_5"
& "$env:ANDROID_HOME\emulator\emulator.exe" -avd nthlink_pixel

# Logs (tags: nthlink_app = app, RootVpn = core)
adb logcat -s nthlink_app RootVpn
adb logcat | findstr /i "nthlink RootVpn"
```

There is **no section** for a Kotlin Android launch-config in Eclipse: Eclipse cannot attach a
debugger to an Android app process. Debugging = `adb logcat` + the app's existing `Log` statements;
for breakpoint debugging keep Android Studio installed alongside.

---

## 12. Known Eclipse limitations (read before you panic)

| Symptom | Reality / workaround |
|---|---|
| Kotlin files show red underlines / "cannot be resolved" | Editor hint only — the JetBrains Kotlin plugin lags Kotlin 2.2.21. Run the **Gradle build**; if it compiles, it's fine. |
| `AppDatabase_Impl`, `*FragmentArgs`, `*Binding` classes "missing" | Generated by KSP/SafeArgs **during** the Gradle build. Run `assembleDebug` once, then right-click project → **Refresh** (`F5`). |
| No "Run as Android Application" | Correct — that is Android Studio tooling. Use `installDebug` + `adb start`/`logcat`. |
| `outline/` folder doesn't import | Expected; it is not a Gradle module (`settings.gradle` includes only `:app`, `:core`). |
| Koin / Room / serialization "cannot be found" in editor | Build once so the classpath is resolved; refresh. |

---

## 13. Troubleshooting — Windows specific

| Symptom | Cause / fix |
|---|---|
| Eclipse won't start (`Version ... is no longer supported`) | JDK mismatch — fix `eclipse.ini` `-vm` (Step 5) and ensure a 64-bit Eclipse. |
| Gradle build fails with `Unable to locate a Java Runtime` | `JAVA_HOME` unset/old (Step 3) or Buildship JVM = a JRE (Step 7). Restart terminal/Eclipse. |
| `SDK location not found` | Missing/invalid `local.properties` — recreate with escaped backslashes (Step 5). |
| `Failed to install the following Android SDK packages` / licenses | Run `sdkmanager.bat --licenses`; reinstall `platforms;android-36`, `build-tools;36.0.0`. |
| Import shows no Gradle project | Buildship missing → install from Marketplace (Step 6); verify `settings.gradle` includes `:app`, `:core`. |
| Build slow / `Daemon ... heap exhausted` | Increase JVM heap: `gradle.properties` → `org.gradle.jvmargs=-Xmx4096m -Dfile.encoding=UTF-8`. |
| Downloads stall (proxy/firewall) | `gradle.properties`: `systemProp.http.proxyHost` / `systemProp.http.proxyPort` + https variants. |
| Mojibake (方块字) in `strings.xml` | Set workspace encoding to UTF-8 (Step 7, §2) and re-open files. |
| Stale build after changing JDK | Delete `C:\Users\<you>\.gradle\daemon` and the project `.gradle`, then refresh/rebuild. |
| `adb` shows `unauthorized` | Accept the RSA prompt on the device; `adb kill-server; adb start-server`; re-plug. |
| AVD won't boot / hax error | Use `x86_64` system images; enable virtualization (WHPX) in Windows Features. |
| `requires compileSdk ...` build errors | Install `platforms;android-36` (Step 2). |

---

## 14. Daily workflow summary

```powershell
# Edit in Eclipse → build/install from the Gradle Tasks view OR terminal

.\gradlew.bat :app:assembleDebug                       # build APK
.\gradlew.bat :app:installDebug                        # build + install
adb shell am start -n com.nthlink.android.client/.ui.LaunchActivity
adb logcat -s nthlink_app RootVpn                      # watch status/error flows
.\gradlew.bat :app:testDebugUnitTest :core:testDebugUnitTest   # unit tests
```

Remember: this repo is a **framework** — the VPN engine and backend are the TODOs in
`core/RootVpnClient.kt` and `core/Core.kt`. Eclipse + Gradle let you build/run the shell and iterate
on those two files; keep Android Studio handy when you need real on-device debugging.
