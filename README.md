# Write-Up: Reviving Hyper Heroes — From a Dead APK to a Fully Functional Local Server

> **Project:** Phyper — Full reverse engineering of the mobile game Hyper Heroes (Unity/IL2CPP)  
> **Platform:** Android ARM64  
> **Engine:** Unity 2018.3.5f1, IL2CPP backend  
> **Status:** Full tutorial working — login, battle, equipment, promotion, gacha  

---

## Table of Contents

1. [Context and Motivation](#1-context-and-motivation)
2. [Phase 1 — APK Decompilation](#2-phase-1--apk-decompilation)
3. [Phase 2 — Traffic Interception with Burp Suite](#3-phase-2--traffic-interception-with-burp-suite)
4. [Phase 3 — Network Protocol Discovery](#4-phase-3--network-protocol-discovery)
5. [Phase 4 — IL2CPP Dump and Static Analysis](#5-phase-4--il2cpp-dump-and-static-analysis)
6. [Phase 5 — Frida and the Emulator Problem](#6-phase-5--frida-and-the-emulator-problem)
7. [Phase 6 — Migration to a Physical Device (Motorola)](#7-phase-6--migration-to-a-physical-device-motorola)
8. [Phase 7 — Frida Instrumentation (hhproxy.js)](#8-phase-7--frida-instrumentation-hhproxyjs)
9. [Phase 8 — Resource Server](#9-phase-8--resource-server)
10. [Phase 9 — NRpc Protocol Implementation](#10-phase-9--nrpc-protocol-implementation)
11. [Phase 10 — APK Patching and Build Pipeline](#11-phase-10--apk-patching-and-build-pipeline)
12. [Phase 11 — Making the Tutorial Work](#12-phase-11--making-the-tutorial-work)
13. [Phase 12 — Cash Server (Billing)](#13-phase-12--cash-server-billing)
14. [Phase 13 — Critical Bugs and How They Were Fixed](#14-phase-13--critical-bugs-and-how-they-were-fixed)
15. [Final Result](#15-final-result)
16. [Lessons Learned](#16-lessons-learned)

---

## 1. Context and Motivation

**Hyper Heroes** was a mobile action/RPG pinball-style game developed by Allstar Games (published by Hyperjoy). The game had its servers shut down, making it completely unplayable — upon opening the app, it simply wouldn't get past the login screen because it depended entirely on remote servers for absolutely everything: login, player data, assets, battle, chat, shop.

The goal of this project was to **revive the game by creating a complete local server**, allowing it to be played offline/LAN without depending on any external infrastructure.

---

## 2. Phase 1 — APK Decompilation

### Tools used
- **apktool 3.0.1** — APK decompilation and recompilation
- **IL2CppDumper** — C# class dump from the IL2CPP binary

### Process

The first step was to obtain the original Hyper Heroes APK (version 1.1.80, build `1bbb10831.2208240940`) and decompile it with apktool:

```powershell
apktool d hyper-heroes.apk -o hyper-heroes-original
```

This extracted:
- `AndroidManifest.xml` — App configuration
- `assets/` — Unity assets (AssetBundles, Addressables, binary data)
  - `assets/aa/Android/` — Unity Addressables (catalog.json, settings.json, 35 .bundle)
  - `assets/bin/Data/` — Unity binary data, including `global-metadata.dat`
  - `assets/AtlasData/`, `assets/Font/`, `assets/UITexture/` — Textures and fonts
- `lib/arm64-v8a/` — Native binaries, including **`libil2cpp.so`** (the IL2CPP-compiled ARM64 executable)
- `smali/`, `smali_classes2/` — Decompiled Java/Kotlin code (SDK, wrappers)
- `res/` — Android resources (layouts, strings, drawables)

### Initial discovery: The game uses IL2CPP

Upon examining `lib/arm64-v8a/`, we found `libil2cpp.so` — which confirmed that all Unity C# code was compiled to native ARM64 code via IL2CPP. This meant we wouldn't have direct access to C# (as we would with Mono), but we could use IL2CppDumper to extract metadata.

### IL2CppDumper

We ran IL2CppDumper with `libil2cpp.so` + `global-metadata.dat`:

```
Il2CppDumper.exe libil2cpp.so global-metadata.dat output/
```

Result:
- **`dump.cs`** — ~600,000 lines of C# class definitions with method names, fields, memory offsets, and RVAs (relative virtual addresses in the binary). This file became our "map" for all reverse engineering.
- **`script.json`** — Decimal addresses for RVA conversion
- **`il2cpp.h`** — C struct definitions
- **`DummyDll/`** — Dummy DLLs for reference in tools like dnSpy

The `dump.cs` revealed the entire game architecture: classes like `NRpcNetwork`, `NRpcParams`, `NContainerLobby`, `NManagerGame`, `PlayerInfo`, `OwnedCardInfo`, `LevelEndResponse`, etc.

---

## 3. Phase 2 — Traffic Interception with Burp Suite

### Proxy setup

To understand the game's network behavior, we set up an HTTP proxy with Burp Suite:

```
adb shell settings put global http_proxy <HOST_MACHINE_IP>:8080
```

Additional setup required:
- Install the Burp CA certificate on the device (`.cer` format)
- Enable TLSv1.1, TLSv1.2, and TLSv1.3 in Burp (the game used legacy ciphers)
- Enable "Allow unsafe renegotiation" in Burp
- Configure the proxy on "All interfaces" to accept device connections

### First captured requests

Upon opening the game with the proxy configured, we captured the HTTP bootstrap sequence:

1. **`GET /notice.txt`** → `notice-hczz-oversea.oss-us-east-1.aliyuncs.com` (Aliyun OSS)
   - Game notice/announcement — returned empty (Content-Length: 0)

2. **`GET /Addressables/ServerData/P21/Android/catalog_2022.08.24.06.02.11.hash`** → `resource.hyperheroes.net` (CloudFront/S3)
   - Addressables catalog hash — the game checks whether it needs to update assets
   - Response: `79a50b6a9a636b6a450312e471f6c2d6` (32 bytes, MD5)

3. **`GET /Addressables/ServerData/P21/Android/catalog_2022.08.24.06.02.11.json`** → `resource.hyperheroes.net`
   - Full Unity Addressables catalog — lists all `.bundle` files the game needs to download

4. **`GET /serverlist/api/queryList`** → `serverlist.hyperheroes.net`
   - **Critical endpoint**: returns the server list with `loginProxy` (game server TCP address), `resourceUrl`, `noticeUrl`, etc.

5. **`GET /login/guest/guestlogin`** → Guest login
6. **`GET /globalservice/account/accountList`** → XD SDK account listing
7. **`GET /myip`** → `myip.hyperheroes.net:1053` — returns the client's IP address

### Crucial discovery: AssetBundles + Resource server

Analyzing the `catalog.json` inside the APK, we discovered that the game makes requests for **Unity Addressables** — asset bundles (audio, effects, UI, textures) hosted at:

```
http://resource.hyperheroes.net/Addressables/ServerData/P21/Android/<bundle>.bundle
```

The catalog listed **35 Addressable bundles**. Additionally, the game also downloaded **update AssetBundles** (126 `.assetbundle` files) from:

```
http://resource.hyperheroes.net/2021120801/600001/Android/<nome>.assetbundle
```

The update list came from a manifest `NBundleInfos.bytes` — a simple text file with the format:
```
-1@ElementName@FileName@MD5Hash@Extension@FileBytes
```

**These assets were still online** on Amazon's S3/CloudFront even with the game servers dead — which gave us a window to mirror all of them before they were removed.

### What was NOT visible through the HTTP proxy

Burp only captured HTTP traffic. After `queryList`, the game opened **direct TCP connections** (not HTTP) to ports like 9150, 9151, 9152. This TCP traffic was the game's proprietary **NRpc** protocol — protobuf + DES-CBC encryption. To analyze it, we needed a different approach.

---

## 4. Phase 3 — Network Protocol Discovery

### PCAP capture with Wireshark

Since NRpc traffic didn't go through the HTTP proxy, we used Wireshark/tcpdump to capture all network traffic from the device:

```
adb shell tcpdump -w /sdcard/capture.pcap
```

The captured PCAPs (`Pcap1.pcapng`, `Pcap2.pcapng`) revealed the complete TCP communication.

### NRpc protocol decoding

Cross-referencing the PCAPs with `dump.cs`, we mapped the protocol:

**Framing:**
```
[varint32(len+1)] [segmento 0 — Header protobuf, plaintext]
[varint32(len+1)] [segmento 1 — Body protobuf, DES-CBC criptografado]
[varint32(len+1)] [segmento 2 — CommonResponse, DES-CBC criptografado]  (nas respostas)
[0x00]            ← terminador de pacote
```

- Each segment's length is encoded as varint32 with value = `actual_size + 1`
- The `0x00` byte marks the end of the packet (since `0 + 1 = 1`, and zero-length segments don't exist)

**Encryption:**
- Algorithm: **DES-CBC**
- Key: `7e9ac962` (8 bytes UTF-8)
- IV: `01c20de0` (8 bytes UTF-8)

The key and IV were found in `dump.cs` (`NRpcParams` class) and confirmed via Frida hooks on `NRpcParams.Encrypt` / `NRpcParams.Decrypt`.

**Header protobuf (Segment 0 — plaintext):**
- Field 1 (varint) = Error code (0 = success)
- Field 2 (string) = RPC method name (e.g., "XDLogin", "HeartBeat", "CommonUserData")
- Field 3 (string) = Request token (UUID to correlate request/response)

**Body protobuf (Segment 1 — encrypted):**
- Content specific to each RPC method
- Protobuf fields mapped directly to classes from `dump.cs`

### Decoding tools created

To analyze the PCAPs, we created specialized Node.js scripts:

- **`decode_pcap.js`** — Basic NRpc segment decoder (generic protobuf + DES-CBC)
- **`decode_pcap2.js`** — Decoder with full varint32 framing, parsed entire PCAPs
- **`decode_pcap3.js`** — Decoder specific to the second PCAP capture
- **`decode_lotto.js`** — Decoder focused on gacha system responses

These scripts allowed us to see exactly what the original server sent for each RPC — the "Rosetta Stone" for reimplementing every endpoint.

---

## 5. Phase 4 — IL2CPP Dump and Static Analysis

### Class and method mapping

The `dump.cs` with ~600k lines was the foundation of all analysis. We extracted:

**Network classes:**
- `NRpcNetwork` / `NRpcNetworkEx` — TCP transport layer + serialization
- `NRpcParams` — DES-CBC encryption (static Encrypt/Decrypt)
- `DelegatePushRPC` — Push system (server → client)

**Data classes (protobuf):**
- `PlayerInfo` — Player data (userId, nick, level, rmb/diamonds, gold)
- `OwnedCardInfo` — Hero data (55+ protobuf fields: actorId, quality, stats, equipment)
- `CommonUserDataResponse` — Mega-response post-login (51 fields: config, inventory, heroes, levels, tasks...)
- `LevelEndResponse` — Response upon completing a level (result, drops, XP, updated cards)
- `LottoResponse` / `OpenLottoHmiResponse` — Gacha system

**Gameplay classes:**
- `NContainerLobby` — Lobby lifecycle (Awake → Start → InitFunctionMap)
- `UIGuide` — Tutorial system (Open/Close/CloseAndCompleteIntro)
- `NManagerGame` — Global manager (playerInfo, SubPlayerRmb, etc.)
- `NManagerChannel` — Billing layer (Pay, PayOver, IsGuest)

### RVA mapping (binary addresses)

Each method in `dump.cs` had a commented offset. We converted these offsets to RVAs (Relative Virtual Addresses) that could be used as Frida hook points:

| Method | RVA |
|---|---|
| `NRpcParams.Encrypt` | `0x26CBA54` |
| `NRpcParams.Decrypt` | `0x26CBC5C` |
| `NRpcNetworkEx.RequestRpc` | `0x210AE98` |
| `NRpcNetworkEx.OnPackedReceived` | `0x210C354` |
| `NContainerLobby.Awake` | `0x21830CC` |
| `UIGuide.Open` | `0x298D510` |
| `NDebug.LogError` | `0x237A4AC` |

### Ghidra analysis

Alongside the dump, we used Ghidra scripts (`ghidra.py`, `ghidra_with_struct.py`, `ghidra_wasm.py`) from IL2CppDumper to import type definitions and facilitate static analysis of `libil2cpp.so`. This helped understand the memory layout of IL2CPP objects:

- `Il2CppString`: klass(8) + monitor(8) + length(i32@0x10) + chars(utf16@0x14)
- `Il2CppArray`: klass(8) + monitor(8) + bounds(ptr@0x10) + max_length(i32@0x18) + data(@0x20)
- `List<T>`: klass(8) + monitor(8) + _items(ptr@0x10) + _size(i32@0x18)

---

## 6. Phase 5 — Frida and the Emulator Problem

### Initial attempt with emulator

The first instrumentation attempt was on an **x86_64 Android emulator** (BlueStacks / Android Studio AVD). The idea was to use Frida to hook `libil2cpp.so` and observe NRpc traffic in real time.

### The problem: SIGSEGV on the emulator

**x86_64 emulators with ARM translation (houdini/ndk_translation) DO NOT support IL2CPP hooks.** When attempting to attach Frida to any address in `libil2cpp.so`, the process crashed immediately with:

```
SIGSEGV (Bad access due to invalid address)
```

The reason: `libil2cpp.so` is natively compiled for **ARM64** (`arm64-v8a`). On x86_64 emulators, it runs inside a binary translation layer (houdini) that maps ARM → x86 instructions. Frida's Interceptor works by rewriting instructions at the memory address of the hooked method — but when the code is being translated in real time by houdini, memory addresses don't correspond to the expected layout, and the rewrite corrupts the execution flow.

**Attempts that failed:**
- Hook on `libil2cpp.so` → SIGSEGV
- Using `Module.findExportByName` → IL2CPP exports aren't standard ELF exports
- Hooking at the Java layer → doesn't give access to the NRpc protocol (which is native)
- Android Studio AVD with ARM64 image → extremely slow and unstable

### Conclusion

There was no workaround: **we needed a real ARM64 device**.

---

## 7. Phase 6 — Migration to a Physical Device (Motorola)

### Hardware

We migrated to a **Motorola One Vision** (ARM64, Android 9+) — a native ARM64 device where `libil2cpp.so` runs without translation.

### Setting up Frida on the real device

1. **frida-server** (ARM64, version 17.8.2) was transferred to the device:
```powershell
adb push fridaserver/frida-server-17.8.2-android-arm64/frida-server /data/local/tmp/
adb shell chmod 755 /data/local/tmp/frida-server
adb shell su -c "/data/local/tmp/frida-server &"
```

2. **frida-tools** no PC:
```powershell
pip install frida-tools==17.8.2
```

3. **Connectivity test:**
```powershell
frida -U -l  # lists processes on the device
```

### First successful hook

With the real device, the first hook on `NRpcParams.Encrypt` (RVA `0x26CBA54`) worked perfectly:

```javascript
Interceptor.attach(base.add(0x26CBA54), {
    onEnter: function(args) {
        var data = readByteArr(args[0]);
        console.log("ENCRYPT plaintext: " + toHex(data));
    }
});
```

For the first time, we could see decrypted NRpc traffic in real time — the protocol's "black box" was opened.

### LAN networking

With a physical device, communication now went through local WiFi:
- PC (server): `192.168.0.5`
- Motorola (client): `192.168.0.X` (DHCP)

This added the need to configure `PUBLIC_HOST` on the server and adjust the build scripts to use the LAN IP instead of `10.0.2.2` (host alias on the emulator).

---

## 8. Phase 7 — Frida Instrumentation (hhproxy.js)

### Script evolution

The instrumentation script went through several iterations:

1. **`hhproxy_aggressive.js`** — Initial aggressive version with many hooks, including libc and Java hooks. Caused instability.

2. **`hhproxy_full.js`** — Intermediate version with extensive hooks.

3. **`hhproxy.js`** — **Final version: 21 hooks + 4 diagnostic**, purely passive (read-only). No libc hooks, no Java replacements, no Unity engine hooks. Observation only.

### Implemented hook categories

| Category | Hooks | Description |
|---|---|---|
| Game logger | 1 | `NDebug.LogError` — throttled + deduplicated (avoids spam) |
| Request (send) | 2 | `RequestRpc` + `RequestRpc(List)` — captures method name + token |
| Encrypt/Decrypt | 2 | `NRpcParams.Encrypt/Decrypt` — plaintext payload pre/post-encryption |
| Response (recv) | 1 | `OnPackedReceived` — header parse + segment count |
| Callback | 1 | `ResponseCallback` — log of dispatch to registered callback |
| Push | 2 | `DelegatePushRPC.Invoke` + `InitPushCallBack` |
| Connection | 2 | `SendConnect` + `OnLog` |
| HeartBeat | 1 | Throttled: logs #1, #2, then 1 every 30 |
| Tutorial | 3 | `UIGuide.Open/Close/CloseAndCompleteIntro` — tracks each tutorial step |
| Lobby lifecycle | 3 | `NContainerLobby.Awake/Start/InitFunctionMap` |
| Billing | 2 | `IsGuest` (forced false) + `PayOver` (forced success) |
| Java HTTP | 1 | `URL.openConnection` — captures all HTTP URLs the game accesses |
| LevelEnd diagnostics | 4 | Specific hooks to debug the LevelEnd `b__0` callback |

### How Frida helped us

Frida was essential in all subsequent phases:

- **NRpc traffic capture** in plaintext (pre-encrypt / post-decrypt)
- **Tutorial tracking** — each tutorial step (UIGuide introId) was logged, allowing us to replicate the exact sequence
- **Crash debugging** — the 4 diagnostic hooks on LevelEnd showed us exactly which field/index caused `ArgumentOutOfRangeException`
- **Lifecycle understanding** — seeing the exact order: `Awake → Start → InitFunctionMap → [RPCs] → UIGuide.Open(introId=2) → ...`

### Execution

```powershell
# Launch the game with instrumentation
frida -U -f com.nkm.kp.hh -l MOD/hhproxy.js

# With log to file
frida -U -f com.nkm.kp.hh -l MOD/hhproxy.js > execution.log
```

---

## 9. Phase 8 — Resource Server

### Motivation

With the original servers dead, the game couldn't:
1. Download the Addressables catalog
2. Download asset bundles (textures, audio, effects)
3. Complete the HTTP bootstrap (queryList, login, myip)
4. Establish NRpc session (lobby, game, chat)

### Server architecture

We created a unified Node.js server (`resource-server/server.js`) that responds on **6+ ports** simultaneously:

| Port | Protocol | Function |
|---|---|---|
| 3000 | HTTP | Main — assets, catalogs, bootstrap |
| 80, 9050, 1888, 1053 | HTTP | Auxiliary bootstrap (replicating original hosts) |
| 9150 | TCP NRpc | Lobby — login, data, battle |
| 9151 | TCP NRpc | Game zone — real-time battle |
| 9152 | TCP NRpc | Chat |

### Asset mirroring

First of all, we took advantage of the CDNs still being online to mirror all assets:

**Addressables (35 bundles):**
```powershell
cd resource-server
node download-assets.js
```

The `download-assets.js` script reads `catalog.json` from the APK, extracts all bundle URLs, and downloads them to `mirror/Addressables/ServerData/P21/Android/`.

**Update AssetBundles (126 files):**
```powershell
node sync-updates.js
```

The `sync-updates.js` script:
1. Downloads `NBundleInfos.bytes` from the CDN
2. Parses the manifest (text format with `@`-separated fields)
3. Does parallel download (8 workers) of all `.assetbundle` files

### Implemented HTTP endpoints

| Endpoint | Function |
|---|---|
| `/notice.txt` | Empty notice |
| `/activities*.html` | Local notice HTML page |
| `/serverlist/api/queryList` | Server list — points `loginProxy` and `resourceUrl` to localhost |
| `/serverlist/api/checkServerStatus` | Always online |
| `/login/guest/guestlogin` | Guest login → `code=0&ext=<MD5>` |
| `/globalservice/account/accountList` | Account list XD SDK |
| `/myip` | Client IP |
| `/login/check/getExt` | SHA-256 hash |
| `/Addressables/.../catalog*.json` | Patched catalog (dynamic URLs to local host) |
| `/Addressables/.../catalog*.hash` | Static hash (forces catalog refresh) |
| `/Addressables/.../*.bundle` | Serves the 35 bundles from mirror |
| `/<ver>/<ch>/Android/*.assetbundle` | Serves the 126 update AssetBundles |
| `/<ver>/<ch>/Android/*.bytes` | Update manifest (`NBundleInfos.bytes`) |
| `ANY *` | Catch-all → empty 200 (avoids blocking 404) |

### Dynamic catalog patching

The original `catalog.json` contained URLs pointing to `resource.hyperheroes.net`. The server intercepts catalog requests and rewrites all URLs dynamically to `http://<PUBLIC_HOST>:3000/...`, without needing to modify the file in the APK (although `build.ps1` also does this as a fallback).

---

## 10. Phase 9 — NRpc Protocol Implementation

### The codec (nrpc-codec.js)

We created a framing module for the NRpc protocol:

```javascript
const { encodePacket, NRpcStreamDecoder } = require('./nrpc-codec');

// Montar e enviar um pacote com 3 segmentos
const packet = encodePacket([headerBuf, bodyBuf, commonResponseBuf]);
socket.write(packet);

// Decodificar stream incremental (chunks parciais OK)
const decoder = new NRpcStreamDecoder();
socket.on('data', (chunk) => {
    const packets = decoder.push(chunk);
    for (const segments of packets) { /* processar */ }
});
```

The codec handles:
- Varint32 framing (value = `segmentLength + 1`)
- `0x00` terminator
- Incremental stream (TCP can fragment packets)

### DES-CBC Encryption

Each request/response body is encrypted with DES-CBC:
- **Key:** `7e9ac962` (8 bytes)
- **IV:** `01c20de0` (8 bytes)

Note: Node.js v17+ blocks DES as it's considered insecure. We used the `--openssl-legacy-provider` flag to work around it.

### Manual protobuf

Since we didn't have the original `.proto` files, we reimplemented protobuf serialization manually with helper functions:

```javascript
function pbFieldVarint(fieldNum, value) { /* ... */ }
function pbFieldBytes(fieldNum, buf) { /* ... */ }
function pbFieldString(fieldNum, str) { /* ... */ }
```

Each RPC method had its fields mapped from:
1. The `dump.cs` (field names and types)
2. Decoded PCAPs (real values from the original server)
3. Frida hooks (decrypted traffic in real time)

### Implemented NRpc methods

The implementation grew organically as we progressed through the tutorial:

| Method | Type | Description |
|---|---|---|
| `XDLogin` | Login | PlayerInfo + Token + zone addresses + UserGuide |
| `HeartBeat` | Keepalive | Header-only ACK (no body) |
| `CommonUserData` | Mega-response | Config + heroes + inventory + levels + chat + lotto config |
| `LevelData` | Query | Level data (empty for new player) |
| `StartLevel` | Battle | CardInfo (ranking) + OwnedCardInfo + SyncHeart |
| `LevelEnd` | Battle | Compound RPC: result + drops + XP + inventory + MapData |
| `UpdateUserGuide` | Tutorial | Persists tutorial bitmask |
| `UpdatePlayerTeam` | Lobby | Saves team composition |
| `PlayerTeam` | Lobby | Returns saved team |
| `LoginChat` | Chat | `canLogin=true` |
| `OnePieceEquipItem` | Equipment | Equips 6 slots + consumes inventory |
| `QualityUp` | Promotion | Promotes quality (10→20→21→30→...) |
| `OpenLottoHmi` | Gacha | Pool of 8 heroes + LottoRank |
| `Lotto` | Gacha | ShowNewCard + OwnedCardInfo + hero persistence |
| `GetMonthOperActivityDot` | UI | Activity indicator |
| `PlayerActorListOwnedCardAfter` | Cards | Complete list of updated cards |
| +18 stubs | Various | Header OK + minimal body |

### Per-account state (in memory)

The server maintains state per guest account (UUID):

```javascript
const accountState = new Map();
// Each account has:
// - ownedCards: Map<cardId, { actorId, quality, star, equippedSlots, stats }>
// - inventory: Map<itemId, { stuffId, count }>
// - userGuide: tutorial bitmask
// - team: array of cardIds
// - nextCardId, nextStuffId: auto-increment counters
```

---

## 11. Phase 10 — APK Patching and Build Pipeline

### The problem: host redirection

The game had hardcoded hostnames in two places:
1. **`catalog.json` / `settings.json`** — Addressables URLs (plain text)
2. **`global-metadata.dat`** — Bootstrap hostnames (binary strings)

### build.ps1 — Patch + build + sign pipeline

The `build.ps1` automates the entire process:

**Steps 1-2: Patch settings.json and catalog.json**
```
http://resource.hyperheroes.net/Addressables/ServerData/P21/Android
→ http://192.168.0.5:3000/Addressables/ServerData/P21/Android
```

**Step 3: Patch global-metadata.dat**

The challenge: strings in `global-metadata.dat` are stored with fixed length. If we replace `serverlist.hyperheroes.net` (26 bytes) with `192.168.0.5` (11 bytes), we corrupt the binary.

**Solution: sslip.io** — A service that resolves any IP embedded in the hostname:

```
serverlist.hyperheroes.net  (26 bytes)  →  serve.192-168-0-5.sslip.io  (26 bytes)
server-kp.hyperheroes.net   (25 bytes)  →  serv.192-168-0-5.sslip.io   (25 bytes)
myip.hyperheroes.net        (20 bytes)  →  192-168-0-5.sslip.io        (20 bytes)
```

The `New-FixedLengthRedirectHost` function generates prefixes with the exact required length.

The `Replace-ByteSequence` function does byte-by-byte binary search and replaces while maintaining the same offset and size.

**Step 4: Rebuild with apktool**
```powershell
apktool b hyper-heroes-original -o hyper-heroes-patched.apk
```

**Step 5: Zipalign**
```powershell
zipalign -f 4 hyper-heroes-patched.apk hyper-heroes-aligned.apk
```

**Step 6: v2 Signing**
```powershell
apksigner sign --ks debug.keystore --ks-pass pass:android hyper-heroes-aligned.apk
```

The v2 signature is required for Android 7+ (API 24+). The original APK used v1+v2.

### build-cash.ps1 — Alternative build (billing only)

A variant that keeps `settings.json` and `catalog.json` original (pointing to the real CDN) and only patches the billing host:

```
http://39.107.237.64:8080  →  http://192.168.0.5:3001
```

This allows using the original CDN for assets while redirecting purchases to the local cash-server.

---

## 12. Phase 11 — Making the Tutorial Work

The Hyper Heroes tutorial is a rigid and strictly server-orchestrated sequence. Each step depends on the previous one. The implementation required faithfully reproducing the same sequence that the original server would send.

### Complete tutorial flow (captured via Frida)

```
 1. HTTP bootstrap: queryList → guestlogin → accountList → checkServerStatus → myip
 2. TCP Lobby (9150): SendConnect → XDLogin → CommonUserData
 3. TCP Chat (9152): SendConnect → LoginChat
 4. Lobby RPCs: OpenTreasurePage, GetFriends, GetBlackList, SyncRankingAward,
    LevelData×2, SyncEvent, SyncOperActivity, GetPlayerEvent (stubs OK)
 5. Continuous HeartBeats (Lobby 9150 + Chat 9152)
 6. Tutorial guides: introId 4899→4913 (intro sequence)
 7. Lobby init: Awake → Start → InitFunctionMap
 8. GetMonthOperActivityDot, GetCeremonyExchange, etc.
 9. UIGuide introId=2 → UpdateUserGuide → introId=3 → PlayerTeam → UpdatePlayerTeam
10. LevelStart (levelId=300101) → Tutorial battle guides (1001→1012)
11. LevelEnd (compound RPC, 3 segs) → drops + inventory + XP
12. Lobby reinit → guides 3011→3012 → UpdateUserGuide
13. Guide 3021: PlayerActorListOwnedCardAfter → Guide 3022
14. OnePieceEquipItem (hero 100017) → equip 6 slots, consume items
15. Guide 3023 → QualityUp (hero 100017) → quality 10→20
16. Guides 3024→3026 → UpdatePlayerTeam (team: 260398;260397)
17. LevelStart (levelId=300102) → Tutorial battle guides
18. LevelEnd (level 300102)
19. Lobby reinit → OpenLottoHmi (pool of 8 heroes)
20. Tutorial gacha → Lotto (RMBFree) → ShowNewCard (hero 100060) → hero added
```

### Extracted configuration data

For the tutorial to work correctly, we needed to extract data from the game's configuration tables (CSV assets):

- **`DataPlayerActorItemConfig.csv`** — Map `actorId_quality → [6 itemIds]` for each hero/tier
- **`DataItem.csv`** — Item table (id, name, type, stackSize)
- **`DataLevel.csv`** — Level configuration (id, drops, energy)
- **`DataLevelBossDrop.csv`** — Drop tables (levelId → [{itemId, count}])
- **`DataEquipment.csv`** — Equipment and stats

These CSVs were extracted from Unity AssetBundles using **UABEA** (Unity Asset Bundle Extractor Avalonia).

### Inventory pre-seed

A crucial discovery: the original server pre-loaded the inventory with the items needed to equip the starter heroes at quality 10:

- Hero 100003: `[800010×2, 800015×2, 800004×1, 800110×1]`
- Hero 100017: `[800009×2, 800008×2, 800000×1, 800006×1]`

Without this, the tutorial got stuck at the equipment step because the inventory was empty.

### SyncConfigResponse — The mega-configuration

The `CommonUserData` needed to include, in field 1 (`SyncConfigResponse`), a functional `SyncLottoTimeResponse` so the gacha wouldn't crash:

```
SyncLottoTimeResponse {
  F1 = SecsToGoldFree (0 = free pull available)
  F2 = SecsToRmbFreeSecs (0 = free pull available)
  F3 = repeated LottoAwards [Gold (type=1), RMB (type=2)]
  F4 = repeated AllLottoPoolActors [8 heroes with actorId]
}
```

Without this, `NDataLottoPackage.m_lottoArray` remained null and the gacha callback crashed with `NullReferenceException`.

---

## 13. Phase 12 — Cash Server (Billing)

### Motivation

The original game communicated with a separate billing server (`39.107.237.64:8080` and `47.90.247.46:9150`) to verify purchases, check diamond balance, and process transactions.

### Implementation (cash-server/)

The cash-server is a standalone Express server that operates in **FORCE_SUCCESS mode** (default) — never contacts upstream, always returns fake success with maximum values:

```javascript
const MAX = 2147483647; // INT_MAX

function makeFakeSuccess(pathname) {
    // Returns maximum balance for any balance query
    if (p.includes('balance') || p.includes('diamond')) {
        return { code: 0, data: { balance: MAX, rmb: MAX, diamond: MAX } };
    }
    // Returns success for any purchase
    if (p.includes('topup') || p.includes('purchase')) {
        return { code: 0, data: { orderId: 'LOCAL_' + Date.now(), status: 1, rmb: MAX } };
    }
    // ...
}
```

### Features

1. **Balance queries** → Always returns `INT_MAX` for all currency fields
2. **Order creation** → Generates fake local orderId
3. **Payment verification** → Always returns success
4. **VIP status** → VIP level 10 with no expiration
5. **Diamond patching** — Recursively replaces numeric fields in JSON responses (`rmb`, `diamond`, `crystal`, `gem`, `coin`, `balance`, etc.) with `INT_MAX`

### Integration with Frida

On the client side, Frida hooks complement:

- `NManagerChannel.Pay()` → After each call, injects `PayOver(EProductResult.Success=1)` automatically
- `NGooglePay.purchase()` → Blocked via Java hook (no Google Play UI appears)
- `NManagerChannel.get_IsGuest()` → Forced `false` to unlock purchases
- `PlayerInfo.get_Rmb()` → Always returns `INT_MAX`
- `NManagerGame.SubPlayerRmb()` → No-op (diamonds never decrement)

---

## 14. Phase 13 — Critical Bugs and How They Were Fixed

### Bug #1: SIGSEGV — Responses with 2 segments instead of 3

**Symptom:** Immediate crash upon receiving any NRpc response.

**Cause:** The server was sending `[header, body]` (2 segments), but the original protocol uses `[header, body, commonResponse]` (3 segments). The 3rd segment is an encrypted protobuf "common response" (`{1: {1:1, 2:0, 4:0}}`).

**Fix:** Added constant `COMMON_RESPONSE_RAW` and function `buildCommonResponseSeg()`. All RPCs (except XDLogin, HeartBeat, and Chat RPCs) now send 3 segments.

### Bug #2: LevelEnd — "Index was out of range"

**Symptom:** Crash in the LevelEnd `b__0` callback after completing level 300101.

**Cause:** Field 92 (`StuffInfo[]`) of the `LevelEndResponse` had fewer entries than field 7 (`LevelDropItem[]`). The callback iterates through drops and accesses `GetOwnedStuffInfo(idx)` for each one. Level 300101 drops `[{800010, ×2}, {800010, ×2}]` — the inventory merged into 1 entry, but the callback expected 2.

**Fix:** Emit 1 StuffInfo PER drop item (not per unique item), referencing the same stuffId when the itemId repeats.

**Critical rule:** `count(StuffInfo[]) == count(LevelDropItem[])` — ALWAYS.

### Bug #3: GetMonthOperActivityDot — incorrect body

**Symptom:** Crash when entering the lobby.

**Cause:** Body was `pbFieldVarint(127, 0)` (generic stub), but the client expects `pbFieldVarint(1, 0)` (field 1 = isHasActivity = false).

**Fix:** Separated from the stub group and returns the correct field.

### Bug #4: OnePieceEquipItem — empty inventory

**Symptom:** Tutorial got stuck at the equipment step.

**Cause:** Account created with `inventory: new Map()` empty. The quality-10 items didn't come from any drop.

**Fix:** Pre-seed the inventory with quality-10 items from both starter heroes at account creation.

### Bug #5: Double quality promotion

**Symptom:** Hero jumped from quality 10 straight to 21.

**Cause:** `OnePieceEquipItem` was promoting internally (10→20), and then `QualityUp` promoted again (20→21). They are separate RPCs.

**Fix:** Removed promotion from `OnePieceEquipItem`. Now it only equips items. Promotion is 100% handled by `QualityUp`.

### Bug #6: Gacha — NullReferenceException on Lotto

**Symptom:** Crash when opening gacha during the tutorial.

**Cause (triple):**
1. `OpenLottoHmi` returned an empty stub instead of a hero pool
2. `Lotto` returned invalid StuffInfo without `ShowNewCard`
3. `CommonUserData` didn't include `SyncLottoTimeResponse` — leaving `m_lottoArray` null

**Fix (triple):**
1. `OpenLottoHmi` → returns 8 heroes with actorIds
2. `Lotto` → rewritten with `ShowNewCard`, `OwnedCardInfo`, hero persistence
3. `CommonUserData` → added `SyncLottoTimeResponse` in `SyncConfigResponse`

---

## 15. Final Result

### What works

| Feature | Status |
|---|---|
| Full guest login | ✅ |
| 35 Addressables + 126 AssetBundles | ✅ Local mirror |
| Patched Addressables catalog | ✅ |
| 15+ HTTP bootstrap endpoints | ✅ |
| NRpc protocol (Lobby, Game, Chat) | ✅ |
| Complete tutorial (intro → battle → equip → quality up → gacha) | ✅ |
| Equipment system | ✅ |
| Quality promotion | ✅ |
| Functional gacha | ✅ |
| Infinite Diamonds/Gold | ✅ |
| Purchases without Google Play | ✅ |
| Cash server with fake billing | ✅ |
| 25 Frida hooks (passive observation) | ✅ |
| Automated build pipeline | ✅ |

### Final stack

```
┌──────────────────────────────────────────────┐
│           Motorola (ARM64, Android)          │
│  ┌────────────────────────────────────────┐  │
│  │  Hyper Heroes (APK patchado)          │  │
│  │  + Frida hooks (hhproxy.js, 25 hooks) │  │
│  └─────────────────┬──────────────────────┘  │
│                    │ WiFi LAN                 │
└────────────────────┼─────────────────────────┘
                     │
┌────────────────────┼─────────────────────────┐
│    PC (192.168.0.5)│                         │
│  ┌─────────────────┴──────────────────────┐  │
│  │  resource-server (Node.js)             │  │
│  │  ├── HTTP :3000 (assets + bootstrap)   │  │
│  │  ├── HTTP :80, :9050, :1888, :1053     │  │
│  │  ├── TCP  :9150 (Lobby NRpc)           │  │
│  │  ├── TCP  :9151 (Game NRpc)            │  │
│  │  └── TCP  :9152 (Chat NRpc)            │  │
│  └────────────────────────────────────────┘  │
│  ┌────────────────────────────────────────┐  │
│  │  cash-server (Node.js)                 │  │
│  │  └── HTTP :3001 (billing fake)         │  │
│  └────────────────────────────────────────┘  │
└──────────────────────────────────────────────┘
```

---

## 16. Lessons Learned

### 1. ARM emulators are not suitable for IL2CPP hooks
The houdini/ndk_translation layer corrupts Frida's Interceptor. Real ARM64 devices are mandatory for IL2CPP binary instrumentation.

### 2. The third NRpc segment is mandatory
Without the `commonResponse` (3rd segment), the client crashes with SIGSEGV. This wasn't evident in the initial PCAPs because the segments looked like "just another byte" — it took hours of debugging to identify.

### 3. Counters must match exactly
The `LevelEnd.b__0` callback iterates through `GetDropItem(idx)` and accesses `GetOwnedStuffInfo(idx)` for each one. If there are 2 drops but 1 StuffInfo (because we merged duplicates), the second access crashes. The rule is: **1 StuffInfo per drop item, not per unique item**.

### 4. PCAPs + IL2CPP dump + Frida = powerful trio
- PCAPs show the real binary format
- The dump shows field names and types
- Frida confirms in real time and enables crash debugging

### 5. The tutorial is a server-side state machine
Each `UpdateUserGuide` persists a bitmask. If the server doesn't save correctly, the tutorial repeats or skips steps, causing inconsistent states.

### 6. Documenting everything prevents future crashes
The 800+ line `README.md` and this write-up document every decision. When a new NRpc method needs to be implemented, the pattern is already established.

### 7. Mirror assets while the CDNs are still alive
Game servers die first, but CDNs (S3, CloudFront) usually stay online longer. The preventive mirror saved the project.

---

*Phyper Project — March 2026*
