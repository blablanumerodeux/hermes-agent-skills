---
name: frida-android-apk-hook
description: Connect to Frida Gadget embedded in a patched Android APK and hook network calls to extract apiKey or secrets. Requires the APK has libfrida-gadget.so, networkSecurityConfig, and usesCleartextTraffic=true.
triggers:
  - "frida android APK apiKey hook"
  - "frida gadget patched APK"
  - "objection APK SSL bypass"
category: mobile
---

# Frida Android APK Hook — Extract apiKey from patched APK

## Prerequisites

**On the Android phone:**
- Patched APK with Frida Gadget embedded (libfrida-gadget.so in lib/)
- `networkSecurityConfig` in manifest pointing to a config that allows cleartext
- `usesCleartextTraffic=true` in `<application>`
- App must be running (Frida Gadget only listens on port 27042 while app is alive)
- The phone must be reachable on the network (same subnet or forwarded)

**On this server:**
- `frida` CLI installed (`frida --version` works)
- Python `frida` package (`pip install frida`)

## Network Setup

The phone was on `192.168.1.x` and this server on `192.168.0.x` — different subnets, can't reach directly.

Solutions:
1. **Phone as WiFi hotspot** — server connects to phone's hotspot → same subnet
2. **Tailscale VPN** — both on Tailscale network, reach via `100.x` addresses
3. **ADB forward** — `adb forward tcp:27042 tcp:27042` then `frida -H 127.0.0.1:27042`

## Hook Script (save as `/tmp/zen_hook.js`)

```javascript
Java.perform(function() {
    console.log('[ZENHOOK] Script loaded');

    // Intercept java.net.URL creation
    var URL = Java.use('java.net.URL');
    URL.$init.overload('java.lang.String').implementation = function(url) {
        var result = this.$init(url);
        if (url && (url.toLowerCase().includes('zenplanner') || url.toLowerCase().includes('api'))) {
            console.log('[ZENHOOK] URL: ' + url);
        }
        return result;
    };

    // Capture HTTP request headers containing secrets
    var HttpURLConnection = Java.use('java.net.HttpURLConnection');
    HttpURLConnection.setRequestProperty.implementation = function(key, value) {
        if (key && (key.toLowerCase().includes('key') || key.toLowerCase().includes('api') ||
            key.toLowerCase().includes('auth') || key.toLowerCase().includes('token') ||
            key.toLowerCase().includes('sign'))) {
            console.log('[ZENHOOK] ★★ RequestProperty: ' + key + ' = ' + value);
        }
        return this.setRequestProperty(key, value);
    };

    // Log network calls
    HttpURLConnection.getInputStream.implementation = function() {
        console.log('[ZENHOOK] >>> getInputStream: ' + this.getURL());
        return this.getInputStream();
    };

    // Hook OkHttp interceptor if available
    try {
        var Interceptor = Java.use('okhttp3.Interceptor$Interceptor');
        Interceptor.intercept.implementation = function(chain) {
            var req = chain.request();
            console.log('[ZENHOOK] OkHttp: ' + req.url());
            return this.intercept(chain);
        };
        console.log('[ZENHOOK] OkHttp hooked');
    } catch(e) {
        console.log('[ZENHOOK] OkHttp not available');
    }

    console.log('[ZENHOOK] Hooks ready');
});
```

## Workflow

### Step 1: Check if Frida Gadget port is open

```bash
timeout 3 bash -c 'echo > /dev/tcp/192.168.1.192/27042' 2>/dev/null && echo "UP" || echo "DOWN"
```

If DOWN → ask user to open the app on their phone.

### Step 2: Spawn app with Frida CLI (NOT Python API)

**Important:** The Python `frida.get_device("host:port")` hangs for remote servers even though CLI works. Use CLI.

```bash
frida -H 192.168.1.192:27042 -f com.zenplanner.memberapp -l /tmp/zen_hook.js 2>&1 | tee /tmp/frida_out.log &
FPID=$!
sleep 15
kill $FPID 2>/dev/null
cat /tmp/frida_out.log
```

### Step 3: If Python is needed for complex logic

Use background subprocess instead of direct `frida.get_device()`:

```python
import subprocess, time

# Launch via CLI, capture output to file
proc = subprocess.Popen([
    'frida', '-H', '192.168.1.192:27042',
    '-f', 'com.zenplanner.memberapp',
    '-l', '/tmp/zen_hook.js'
], stdout=open('/tmp/fout.txt','w'), stderr=subprocess.STDOUT)

time.sleep(20)
proc.terminate()
# Read /tmp/fout.txt for captured output
```

## Patching APK with Frida Gadget

Use `objection` or manual approach:

```bash
# Option 1: objection (easiest)
objection patchapk -s app.apk -a arm64-v8a -N -C
# -a arm64-v8a: target arch (required when adb not connected to auto-detect)
# -N: add network_security_config.xml (SSL pinning bypass on Android 7+)
# -C: skip signing (sign separately with objection signapk or apksigner)
# Output: app.objection.apk

# Sign the patched APK (uses built-in objection keystore: CN=objection, OU=SensePost)
objection signapk -s app.objection.apk
# Output: app.objection.apk (signed, ready to install)

# NOTE: If installation fails with "no details", rebuild from original APK:
#   cp original.apk working.apk
#   objection patchapk -s working.apk -a arm64-v8a -N
#   # The objection keystore sometimes fails on pre-existing patched APKs.
#   # Always patch the original, not an already-patched version.

# Option 2: Manual
# 1. Decompile: apktool d app.apk -o decompiled/
# 2. Add to lib/: libfrida-gadget.so (from frida-server)
# 3. Add network_security_config.xml allowing cleartext
# 4. Patch AndroidManifest.xml:
#    - android:networkSecurityConfig="@xml/network_security_config"
#    - android:usesCleartextTraffic="true"
# 5. Add to Application tag: <meta-data android:name="com.frida.gadget" android:value=""/>
# 6. Rebuild: apktool b decompiled/ -o patched.apk
# 7. Sign: apksigner sign --ks keystore.jks patched.apk
```

## Troubleshooting

| Problem | Solution |
|---------|----------|
| Port always DOWN | App not running — user must start it manually |
| Different subnets | Use phone hotspot or Tailscale VPN |
| Python `frida.get_device("host:port")` hangs | **Use CLI** (`frida -H ...`) instead — Python API hangs on TCP connections to remote frida-server |
| App dies quickly | Frida spawn is async — attach with script before app closes |
| No output captured | Check `/tmp/frida_out.log` — stdout may be buffered |
| Installation fails "no details" | Patched APK may have bad signature — rebuild from original APK with `objection patchapk -N` |
| APKTool aapt warnings | Ignorable — build still succeeds if final APK is produced |
