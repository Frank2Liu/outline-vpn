# Development Guideline — VSCode on Windows 10

Step-by-step guide to set up a **local Windows 10 environment** for developing the
`nthlink-os-android` project with **VSCode** — including every required package, the dependency
libraries the project uses, and troubleshooting for the common Windows pitfalls.

---

## 0. What this guide gives you

By the end you will be able to:

- Build the app (`app-debug.apk`) from the command line or the VSCode UI.
- Run unit tests.
- Install and launch the app on a connected device / emulator.
- Read live app logs (VPN status / error flows).
- Edit and code with Kotlin + Java + resource/XML support.

> The officially recommended IDE is Android Studio. VSCode works because everything here drives the
> same **Gradle wrapper** — the build result is identical. Full on-device breakpoint debugging is
> only available in Android Studio.

---

## 1. Prerequisites — Windows 10

| Item | Requirement | Why |
|---|---|---|
| OS | Windows 10 64-bit | dev host |
| RAM / disk | 8 GB+ RAM, ~10 GB free disk | Gradle + SDK + emulator |
| Java | **JDK 17** (64-bit) | required by AGP 8.13.0 |
| Android SDK | **platform 36**, **build-tools 36.x**, platform-tools/adb | project uses `compileSdk 36` |
| Gradle | bundled **wrapper** (`gradlew.bat`) — no manual install | repo already provides it |
| VSCode | latest stable (1.9x+) | editor |
| Device/emulator | Android 8.0+ (minSdk 26) device, or an AVD | running the app |

Required packages and library dependencies the project pulls in automatically at build time
(downloaded from Google Maven / Maven Central / JitPack — no manual install):

- AndroidX: core-ktx 1.17.0, appcompat 1.7.1, constraintlayout 2.2.1, lifecycle-runtime-ktx 2.9.4,
  navigation 2.9.5, datastore-preferences 1.1.4, drawerlayout 1.2.0, work 2.11.0, room 2.8.3.
- Material Components 1.13.0 · Play core review-ktx 2.0.2 · app-update-ktx 2.1.0 ·
  kotlinx-serialization-json 1.9.0 · kotlinx-coroutines-play-services 1.10.2 ·
  Koin (koin-android) 4.1.1.
- Test: JUnit 4, androidx.test ext-junit 1.3.0, espresso-core 3.7.0.
- Build plugins (from the wrapper): AGP 8.13.0, Kotlin 2.2.21, KSP 2.2.21-2.0.4,
  Navigation SafeArgs 2.9.5.

All versions are pinned in `build.gradle` / `app/build.gradle` / `core/build.gradle` — you never
download these by hand.

---

## 2. Step 1 — Install JDK 17

Use an Adoptium (Eclipse Temurin) build. Download from
https://adoptium.net/temurin/releases/?version=17 (pick **Windows x64**, `.msi`) and install.

Verify in a **new** PowerShell window:

```powershell
java -version
```

Expected output similar to:

```text
openjdk version "17.0.x" 2026-...
```

Also note the JDK install path (needed later):

```powershell
$jdks = Get-ChildItem "C:\Program Files\Eclipse Adoptium" -Directory | Select-Object -First 1
Write-Output $jdks.FullName    # e.g. C:\Program Files\Eclipse Adoptium\jdk-17.0.x.x-hotspot
```

---

## 3. Step 2 — Install the Android SDK (command-line tools)

The official Android Studio dump of SDK packages is fine, but the lightest path is the
**command-line tools only** bundle.

```powershell
# 1) Where the SDK will live
$sdk = "$env:LOCALAPPDATA\Android\Sdk"
New-Item -ItemType Directory -Force -Path $sdk | Out-Null

# 2) Download command-line tools from
#    https://developer.android.com/studio#command-line-tools-only
#    (file: commandlinetools-win-*.zip) and extract, then arrange to the path below.
#    IMPORTANT: the folder MUST be named "latest" for sdkmanager to find it.
New-Item -ItemType Directory -Force -Path "$sdk\cmdline-tools" | Out-Null
Expand-Archive -Path "$env:USERPROFILE\Downloads\commandlinetools-win-*.zip" -DestinationPath "$sdk\cmdline-tools" -Force
Rename-Item "$sdk\cmdline-tools\cmdline-tools" "$sdk\cmdline-tools\latest"

# 3) Install the packages this project needs and accept licenses
& "$sdk\cmdline-tools\latest\bin\sdkmanager.bat" "platform-tools" "platforms;android-36" "build-tools;36.0.0"
& "$sdk\cmdline-tools\latest\bin\sdkmanager.bat" --licenses     # press 'y' until done
```

If `sdkmanager.bat` complains about Java, run the two lines above from a terminal where
`JAVA_HOME` is set (Step 3 below) before retrying.

---

## 4. Step 3 — Environment variables

Set these **per-user** (persist across reboots), then open a **new** terminal.

```powershell
$sdk = "$env:LOCALAPPDATA\Android\Sdk"
$javaHome = "C:\Program Files\Eclipse Adoptium\jdk-17.0.x.x-hotspot"   # adjust to your version

[Environment]::SetEnvironmentVariable("JAVA_HOME", $javaHome, "User")
[Environment]::SetEnvironmentVariable("ANDROID_HOME", $sdk, "User")
[Environment]::SetEnvironmentVariable("ANDROID_SDK_ROOT", $sdk, "User")

# Add tools to PATH
$oldPath = [Environment]::GetEnvironmentVariable("Path", "User")
$newPath = "$sdk\platform-tools;$sdk\cmdline-tools\latest\bin" + ($(if ($oldPath) { ";$oldPath" } else { "" }))
[Environment]::SetEnvironmentVariable("Path", $newPath, "User")
```

Verify (fresh terminal):

```powershell
java -version
adb --version
echo $env:JAVA_HOME
echo $env:ANDROID_HOME
```

---

## 5. Step 4 — Clone the project

```powershell
git clone <your-repo-url> "D:\OpenSource\nthlink-os-android"
cd "D:\OpenSource\nthlink-os-android"
```

---

## 6. Step 5 — Point Gradle at the SDK (`local.properties`)

Create **`local.properties`** at the project root (never commit it — it is machine-specific).
Backslashes must be escaped (or use forward slashes):

```powershell
echo "sdk.dir=C\:\\Users\\<YourUserName>\\AppData\\Local\\Android\\Sdk" | Out-File -Encoding ascii "D:\OpenSource\nthlink-os-android\local.properties"
```

Verify content:

```powershell
Get-Content "D:\OpenSource\nthlink-os-android\local.properties"
```

---

## 7. Step 6 — First Gradle build (no IDE needed yet)

```powershell
# Run from the project root
.\gradlew.bat --version
.\gradlew.bat :app:assembleDebug
```

First run downloads the Gradle distribution + all dependency libraries (several minutes).
On success the APK is at:

```text
app\build\outputs\apk\debug\app-debug.apk
```

Run the unit tests too:

```powershell
.\gradlew.bat :app:testDebugUnitTest :core:testDebugUnitTest
```

---

## 8. Step 7 — Install VSCode extensions

Open VSCode, go to the Extensions panel (`Ctrl+Shift+X`) and install:

| Extension | Publisher / ID | Purpose |
|---|---|---|
| Java Extension Pack | `vscjava.vscode-java-pack` | Java language server, debugger, project support |
| Kotlin Language | `mathiasfrohlich.vscode-kotlin` | Kotlin syntax + Kotlin Language Server |
| Gradle for Java | `msvscode.gradle` | Gradle Tasks view + file sync features |
| XML | `redhat.vscode-xml` | Android layout / manifest / strings editing |

> If any ID no longer resolves, install by exact name from the Marketplace — the publisher shown
> above is the one to look for.

### Extension settings

```powershell
# (optional) allow Gradle scripts / helper scripts to run
Set-ExecutionPolicy -Scope CurrentUser RemoteSigned
```

---

## 9. Step 8 — VSCode workspace configuration

Create `.vscode/settings.json` (and optionally `.vscode/extensions.json` to recommend the plugins):

```json
{
  "java.configuration.updateBuildConfiguration": "automatic",
  "java.completion.importOrder": ["java", "javax", "androidx", "com", ""],
  "java.debug.settings.onBuildFailureProceed": true,
  "kotlin.debugAdapter.enabled": true,
  "files.exclude": {
    "**/build": true,
    ".gradle": true
  },
  "editor.formatOnSave": false,
  "search.exclude": {
    "**/build": true
  }
}
```

Create `.vscode/extensions.json`:

```json
{
  "recommendations": [
    "vscjava.vscode-java-pack",
    "mathiasfrohlich.vscode-kotlin",
    "msvscode.gradle",
    "redhat.vscode-xml"
  ]
}
```

---

## 10. Step 9 — Open, sync, and build in VSCode

1. `File → Open Folder` → `D:\OpenSource\nthlink-os-android`.
2. When the Java extension asks to import the project, accept it (it uses Gradle's model).
3. Wait for the status bar to stop showing a Gradle/Java sync spinner.
4. Build via **Gradle Tasks** view:
   - Open the Gradle Tasks view (Explorer → **GROOVY GRADLE** section).
   - Expand `nthlink6-android` → `:app` → `build` → double-click **`assembleDebug`**.
   Or just use the integrated terminal:
   ```powershell
   .\gradlew.bat :app:assembleDebug
   ```
5. Errors in the Problems panel are live; the *authoritative* result is the Gradle build.

> Code-generated classes (`AppDatabase_Impl`, `ApkUpdateFragmentArgs`, ViewBinding classes) are
> produced by KSP/SafeArgs during the build. If they show as “missing”, run `assembleDebug` once,
> then `java.clean.workspace` (Ctrl+Shift+P) and sync again.

---

## 11. Step 10 — Run on a device / emulator

### Physical device (USB)

```powershell
adb devices                     # confirm the device shows as "device" (not "unauthorized")
.\gradlew.bat :app:installDebug # builds + installs
adb shell am start -n com.nthlink.android.client/.ui.LaunchActivity
```

### Emulator (AVD) — optional

```powershell
# If you have a system image installed, create and boot one:
& "$env:ANDROID_HOME\cmdline-tools\latest\bin\avdmanager.bat" create avd -n nthlink_pixel -k "system-images;android-36;google_apis;x86_64" --device "pixel_5"
& "$env:ANDROID_HOME\emulator\emulator.exe" -avd nthlink_pixel
```

### Read runtime logs

```powershell
# All app logs (tags: nthlink_app for the app, RootVpn for core)
adb logcat -s nthlink_app RootVpn

# Or a broad filter
adb logcat | findstr /i "nthlink RootVpn"
```

More adb tips:

```powershell
adb reverse tcp:8080 tcp:8080   # if you later run a local backend on this machine
adb logcat -c                   # clear log before a fresh start
```

---

## 12. Dependencies recap (what the build actually needs)

Automatically downloaded by Gradle from `google()`, `mavenCentral()`, and `jitpack.io`:

| Layer | Artifacts |
|---|---|
| Android plugins / toolchain | AGP 8.13.0, Kotlin 2.2.21 + serialization plugin, KSP 2.2.21-2.0.4, SafeArgs 2.9.5 |
| UI / framework | core-ktx 1.17.0, appcompat 1.7.1, constraintlayout 2.2.1, material 1.13.0, drawerlayout 1.2.0, navigation 2.9.5, lifecycle 2.9.4, viewbinding (built-in) |
| Data / async | datastore-preferences 1.1.4, room 2.8.3 (+compiler via KSP), work-runtime-ktx 2.11.0, coroutines |
| Serialization | kotlinx-serialization-json 1.9.0 |
| DI | Koin `io.insert-koin:koin-android:4.1.1` |
| Google Play services | review-ktx 2.0.2, app-update-ktx 2.1.0, kotlinx-coroutines-play-services 1.10.2 |
| Tests | junit 4.x, androidx.test.ext:junit 1.3.0, espresso-core 3.7.0 |
| Project modules | `:app` (depends on `:core`) · `:core` (library) · `outline/` (legacy AAR, **not** in settings.gradle → unaffected) |

SDK packages that must exist locally: `platform-tools`, `platforms;android-36`,
`build-tools;36.0.0` (Step 2) — nothing else is installed manually.

---

## 13. Troubleshooting — Windows 10 specific

| Symptom | Cause / fix |
|---|---|
| `Unable to locate a Java Runtime` when running `gradlew.bat` | `JAVA_HOME` not set (Step 3) or points to a JRE, not a full JDK 17. Restart the terminal after changing it. |
| `SDK location not found` / `local.properties` missing | Create `local.properties` (Step 5). Verify escaped backslashes: `sdk.dir=C\:\\Users\\<you>\\AppData\\Local\\Android\\Sdk`. |
| `Failed to install the following Android SDK packages` / license not accepted | Run `sdkmanager.bat --licenses` and accept; reinstall `platforms;android-36` and `build-tools;36.0.0`. |
| `sdkmanager.bat` won't start | `JAVA_HOME` must be set in the same terminal before invoking sdkmanager. |
| PowerShell blocks running scripts | `Set-ExecutionPolicy -Scope CurrentUser RemoteSigned`. |
| Gradle build is very slow / OOM (`Daemon exited... heap`) | Increase heap in `gradle.properties`: `org.gradle.jvmargs=-Xmx4096m -Dfile.encoding=UTF-8`. Close other JDK processes. |
| Downloads stall behind a proxy/firewall | Set proxy in `gradle.properties`: `systemProp.http.proxyHost=...`, `systemProp.http.proxyPort=...` (and https variants). |
| `Java 17` mismatch errors after switching JDK | Delete `C:\Users\<you>\.gradle\daemon` and `.gradle` in the project, rebuild. |
| VSCode: “Kotlin language server not found” | Set `KOTLIN_LANGUAGE_SERVER_JVM_PATH` to your JDK or install Kotlin via the extension prompt; restart VSCode. |
| VSCode: generated classes marked as missing | Run `:app:assembleDebug` once so KSP/SafeArgs run, then `Java: Clean Java Language Server Workspace`. |
| `adb` device shows `unauthorized` | Accept the RSA fingerprint dialog on the device; re-plug the cable / `adb kill-server; adb start-server`. |
| Emulator won't boot / no acceleration | Enable WHPX/HAXM in Windows features; use an `x86_64` system image on x64 hosts. |
| `minSdk`/`targetSdk` Gradle errors | Ensure you installed `platforms;android-36`; wrong platform causes `requires compileSdk` build errors. |

---

## 14. Daily workflow summary

```powershell
# Edit code in VSCode → build from terminal

.\gradlew.bat :app:assembleDebug                       # build APK
.\gradlew.bat :app:installDebug                        # build + install on device/emu
adb shell am start -n com.nthlink.android.client/.ui.LaunchActivity
adb logcat -s nthlink_app RootVpn                      # watch status/error flows
.\gradlew.bat :app:testDebugUnitTest :core:testDebugUnitTest   # unit tests
```

Remember: this project is a **framework** — the VPN engine and backend live in the TODOs of
`core/RootVpnClient.kt` and `core/Core.kt`. The environment above lets you build and run the shell
today and iterate on those two files as you implement them.
