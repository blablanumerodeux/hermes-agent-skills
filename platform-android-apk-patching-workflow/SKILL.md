---
name: android-apk-patching-workflow
description: Patch Android APKs for SSL unpinning bypass using Frida gadget + network security config, with workarounds for aapt version mismatches and Android 15 binary manifest issues.
triggers:
  - android apk patching ssl pinning
  - apktool aapt version mismatch android 15
  - frida gadget non-rooted android ssl bypass
  - zenplanner memberapp ssl unpinning
---

# Android APK Patching Workflow for SSL Unpinning

## Context
Non-rooted Android phone + patched APK needed to intercept apiKey from a Cordova/Ionic app (ZenPlanner MemberApp) that uses SSL pinning + dynamically-loaded SPA.

## Standard APK Patch Cycle (works when aapt supports the target SDK)

1. **Decompile**: `apktool d original.apk -o decompiled/ -f`
2. **Inspect**: Check `decompiled/lib/` for native libs, `decompiled/res/xml/` for network config, `decompiled/AndroidManifest.xml`
3. **Modify**: Edit smali, XML, manifest, add native libs to `lib/<arch>/`
4. **Patch manifest**: Add `android:networkSecurityConfig="@xml/network_security_config"` and `android:usesCleartextTraffic="true"` to `<application>`
5. **Create network config**: `res/xml/network_security_config.xml`:
   ```xml
   <?xml version="1.0"?>
   <network-security-config>
     <base-config cleartextTrafficPermitted="true">
       <trust-anchors><certificates src="system"/><certificates src="user"/></trust-anchors>
     </base-config>
     <debug-overrides>
       <trust-anchors><certificates src="user"/></trust-anchors>
     </debug-overrides>
   </network-security-config>
   ```
6. **Set extractNativeLibs=true** in manifest if adding .so files
7. **Rebuild**: `apktool b decompiled/ -o rebuilt.apk`
8. **Sign**: `apksigner sign --ks debug.keystore --ks-pass pass:android rebuilt.apk`

## Key Discovery: aapt Version Mismatch

**Problem**: The bundled aapt in apktool (even apktool 2.7.0) cannot parse Android 15 (API 35) binary manifests. Error: `"First type is not attr!"`

**Symptom**: `apktool b` fails with `brut.androlib.AndrolibException: could not exec` during resource compilation.

**Fix**: Do NOT decompile/recompile. Instead, use the **zero-decompile direct patch** method:
1. Keep original APK as base
2. Only modify the binary AndroidManifest.xml using **baksmali** (smali patching) for code changes
3. For manifest attribute changes (cleartextTraffic, networkSecurityConfig), use the **v1 JAR signing trick** or the **manual binary XML patching** approach

## Critical Finding: `objection patchapk` Silent Failure

`objection patchapk` **appears to succeed** (no error) but the output APK is MISSING the Frida gadget:

- Build temp dir (`/tmp/tmpXXXXX.apktemp/`) contains `lib/arm64-v8a/libfrida-gadget.so` (24MB) ✓
- Final APK (`zenplanner.objection.apk`) is 9.8MB with **no `lib/` directory** at all ✗
- The aapt repack step crashes silently or produces an invalid APK

**Workaround**: Do NOT rely on objection's final output. Instead:
1. Run `objection patchapk` to populate the build temp dir with the gadget
2. Find the gadget in the temp dir: `ls /tmp/tmpXXXXX.apktemp/lib/<arch>/`
3. Copy gadget to the manually-unpacked APK: `cp /tmp/tmpXXXXX.apktemp/lib/<arch>/libfrida-gadget.so /tmp/manual_apk/lib/<arch>/`
4. Repack the APK manually (see below)

## Direct Binary Manifest Patching (without decompiling)

The binary AndroidManifest.xml is a compiled AXML format. Options:

### Option A: Python zipfile (for adding native libs / non-resource changes)
- Extract APK: `unzip -o original.apk -d manual_apk/`
- Add native libs: copy `.so` files to `manual_apk/lib/<arch>/`
- **Repack using Python** (no `zip` command needed on this system):
```python
import zipfile, os
with zipfile.ZipFile('original.apk', 'r') as zin:
    with zipfile.ZipFile('repacked.apk', 'w', zipfile.ZIP_DEFLATED) as zout:
        for item in zin.infolist():
            data = zin.read(item.filename)
            if item.filename.startswith('lib/'):
                continue  # skip existing lib/ entries
            zout.writestr(item, data)
        # Add new lib/ entries
        for root, dirs, files in os.walk('manual_apk/lib'):
            for f in files:
                path = os.path.join(root, f)
                arcname = path.replace('manual_apk/', '')
                zout.write(path, arcname)
```
- Run `zipalign -p 4 repacked.apk aligned.apk`
- Sign: `apksigner sign --ks debug.keystore rebuilt.apk`

### Option B: Zip command (if available)
- Extract APK with `unzip`
- Add/remove files with `zip -d` / `zip -a`
- Works for assets, smali, libs — NOT for binary XML changes

### Option B: Full rebuild via apktool (when aapt works)
- Standard decompile → modify → rebuild cycle
- Set `frameworkDIR` to an appropriate `framework-res.apk` for the target SDK

### Option C: Manual binary AXML editing (advanced)
- Requires understanding AXML file format (magic bytes, chunk headers, string pool)
- For adding `android:networkSecurityConfig` attribute — complex, not recommended
- Instead: use Option A + Frida gadget to achieve the same goal (bypass SSL at runtime)

## Key Discovery: Gadget Always in Build Temp Dir

**IMPORTANT**: Even when `objection patchapk` produces a broken/corrupt final APK, the gadget IS successfully injected into the **build temp dir** (`/tmp/tmpXXXXX.apktemp/`). The final output APK is what's broken, not the gadget injection.

This means the workflow is:
1. Run `objection patchapk -s original.apk -a arm64-v8a -N` 
2. Check `ls /tmp/tmpXXXXX.apktemp/lib/arm64-v8a/` — the gadget IS there (24MB) ✓
3. If final APK is broken (missing lib/), manually repack from the temp dir

## SSL Pinning Bypass via Frida Gadget (Non-Rooted)

The patched APK approach leverages the **Frida gadget** (not frida-server) embedded in the app:

1. **Add Frida gadget** to `lib/<arch>/libfrida-gadget.so`
2. **Modify** `AndroidManifest.xml`: `android:extractNativeLibs="true"` (for auto-extraction)
3. **Set** `android:usesCleartextTraffic="true"` and `android:networkSecurityConfig="@xml/network_security_config"` in `<application>`
4. **Sign** the APK

The gadget auto-connects to frida-server on the same network:
```bash
frida -H <phone-ip>:27042 -f com.zenplanner.memberapp -l hook.js
```

## APK Signing Cheatsheet

### Generate debug keystore
```bash
keytool -genkey -keystore debug.keystore -storepass android -alias androiddebugkey \
  -keypass android -keyalg RSA -keysize 2048 -validity 10000 \
  -dname "CN=Android Debug,O=Android,C=US"
```

### Sign with apksigner (v1 only, when v2 fails)
```bash
apksigner sign --min-sdk-version 24 --v1-signing-enabled true --v2-signing-enabled false \
  --ks debug.keystore --ks-pass pass:android --key-pass pass:android \
  --out signed.apk original.apk
```

### Verify signature
```bash
apksigner verify --min-sdk-version 24 signed.apk
```

### Zipalign (required before signing)
```bash
zipalign -p 4 input.apk aligned.apk
```

## Critical: Why `zenplanner_patched.apk` Was Corrupted

The APK from a previous session had:
- Mangled binary AndroidManifest.xml (from manual hex editing or failed apktool rebuild)
- `apksigner verify` failed with: `"Unable to determine APK's minimum supported platform version: malformed binary resource: AndroidManifest.xml"`
- The manifest was readable but `aapt` could not parse it → rebuild impossible

**Lesson**: Always keep the original APK as base. Never manually edit binary XML bytes. Test signing BEFORE sending to phone.

## Remote Frida Connection (frida-server on Android phone)

The Frida gadget embedded in the patched APK listens on port **27042** when the app is running. From a remote server:

### Basic remote spawn + attach workflow

```bash
# Connect and spawn app
frida -H 192.168.1.x:27042 -f com.zenplanner.memberapp

# In the frida REPL, load a hook script:
[local]:: load /tmp/zen_hook.js
```

### Key flags for remote frida

| Flag | Purpose |
|------|---------|
| `-H <ip:port>` | Connect to remote frida-server |
| `-f <package>` | Spawn the named app (blocks until app starts) |
| `-l <script.js>` | Load a hook script |
| `-U` | USB device (not used for WiFi remote) |

### Keeping frida-server alive on non-rooted phone

**Problem**: frida-server on the phone dies when:
- The patched app is force-closed
- The phone sleeps/phone restarts

**Solution**: Keep the app running in foreground on the phone while Frida work is active. The gadget auto-starts when the app launches.

To restart frida-server on the phone (via Termux or ADB):
```bash
# In Termux on the phone:
pkg install frida
frida-server -l 0.0.0.0:27042

# Or if frida-server is already installed:
su
frida-server -l 0.0.0.0:27042
```

### Capturing output from frida spawn (background process issue)

**Problem**: When running `frida -H IP:port -f package -l script.js` in the background with `&`, the output gets truncated because frida spawns the app and the REPL blocks briefly.

**Working pattern**: Run in foreground with sleep, redirect to log file:
```bash
frida -H 192.168.1.x:27042 -f com.zenplanner.memberapp -l /tmp/zen_hook.js > /tmp/frida_out.log 2>&1 &
FPID=$!
sleep 15
kill $FPID 2>/dev/null
cat /tmp/frida_out.log
```

**Better pattern** — use Python frida API to handle the async lifecycle properly:
```python
import frida, time
device = frida.get_device("192.168.1.x:27042", timeout=10000)
pid = device.spawn(["com.zenplanner.memberapp"])
session = device.attach(pid)
script = session.create_script(open("zen_hook.js").read())
script.on('message', lambda msg, data: print(msg))
script.load()
device.resume(pid)
time.sleep(15)
```

### Network reachability check

If frida connection fails with timeout:
```bash
# Check if port is reachable
timeout 3 bash -c 'echo > /dev/tcp/192.168.1.x/27042' && echo "OPEN" || echo "CLOSED"

# Check server's own IP (ensure same subnet)
ip a | grep 'inet '
```

Server was on `192.168.0.123/24`, phone on `192.168.1.x` — same LAN but different subnets. **Both must be on the same subnet or VPN for frida to connect.**

### Network subnet issue workaround

If server and phone are on different subnets:
1. **Phone as hotspot** — phone creates WiFi AP, server connects to it
2. **Tailscale VPN** — both on Tailscale, use the `100.x` Tailscale IP instead of LAN IP
3. **USB debugging + ADB reverse** — `adb reverse tcp:27042 tcp:27042` then connect to `127.0.0.1:27042`

## Frida Script Templates

### Hook WebView SSL errors (Cordova)
```javascript
Java.perform(function() {
    var WebViewClient = Java.use("android.webkit.WebViewClient");
    WebViewClient.onReceivedSslError.implementation = function(view, handler, error) {
        console.log("[+] SSL Error - bypassing!");
        handler.proceed();
    };
});
```

### Hook OkHttp (if used)
```javascript
Java.perform(function() {
    try {
        var OkHttpClient = Java.use("okhttp3.OkHttpClient");
        OkHttpClient.newCall.implementation = function(request) {
            console.log("[OkHTTP] " + request.method().string() + " " + request.url().toString());
            return this.newCall(request);
        };
    } catch(e) { console.log("[!] OkHttp: " + e.message); }
});
```

### Log all JavaScript network calls (Cordova bridge)
```javascript
Java.perform(function() {
    var CordovaHttpClient = Java.use("org.apache.cordova.CordovaHttpClient");
    CordovaHttpClient.send.implementation = function(req) {
        console.log("[HTTP] " + req.method + " " + req.url);
        return this.send(req);
    };
});
```

## Pinning Bypass Prerequisites

Before attempting SSL unpinning:
1. **Check if SSL pinning exists**: Look for Conscrypt, OkHttp certificate validators in `smali/`
2. **Network security config**: Does `<certificates src="user" />` already exist? If yes, user CA certs might already be trusted — try mitmproxy first without Frida
3. **Cleartext traffic**: Is `usesCleartextTraffic="false"`? This means certificate validation is enforced
4. **Cordova apps**: Often implement pinning in JS, not native code — network_security_config + mitmproxy may suffice

## Known Working: ZenPlanner MemberApp (com.zenplanner.memberapp)

- **App type**: Ionic/Cordova (dynamic SPA loaded at runtime)
- **Auth**: Bearer token via `POST /auth/v1/key-login` with `{apiKey, componentVersionId}`
- **apiKey endpoint**: `GET /auth/v1/organizations/{orgId}/key` — returns UUID apiKey
- **apiKey NOT hardcoded** in APK — fetched at runtime from server
- **Component version ID**: `CEE04405-FF91-4CC6-9FF6-3FE45B384569`
- **Org GUID (Pro1)**: `12cdc08f-eeed-4601-85a8-cf36e8486008`
- **FlareSolverr workaround**: Works for schedule scraping but NOT for apiKey (needs auth)
