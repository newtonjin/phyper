# Reviving a Dead Game: Hyper Heroes Private Server from Scratch

> Hyper Heroes is dead. Yep! GMs Gone. Game fully abandoned.
They even said it at their Reddit:::

<img width="302" height="177" alt="image" src="https://github.com/user-attachments/assets/3cb4bef4-ac0e-493b-a92b-9e658e4c3800" />

> So I tore the APK apart, reversed the protocol, hooked the binary with Frida, 
> and built a full replacement server from nothing but packet captures and a 600k-line C# dump.
>
> **Target:** Hyper Heroes v1.1.80 (Unity 2018.3.5f1 / IL2CPP / ARM64)
> **Result:** Full tutorial playable — login, battles, equipment, gacha, everything.

---

## The Roadmap

1. [Why](#1-why)
2. [Tearing the APK Apart](#2-tearing-the-apk-apart)
3. [Sniffing the Wire](#3-sniffing-the-wire)
4. [Cracking the Protocol](#4-cracking-the-protocol)
5. [The IL2CPP Dump — Mapping the Game's Brain](#5-the-il2cpp-dump--mapping-the-games-brain)
6. [The Emulator Problem](#6-the-emulator-problem-spoiler-it-doesnt-work)
7. [Going Physical — Motorola One Vision](#7-going-physical--motorola-one-vision)
8. [Frida Instrumentation — hhproxy.js](#8-frida-instrumentation--hhproxyjs)
9. [Building the Resource Server](#9-building-the-resource-server)
10. [Implementing the NRpc Protocol](#10-implementing-the-nrpc-protocol)
11. [Patching the APK](#11-patching-the-apk)
12. [Tutorial Hell](#12-tutorial-hell)
13. [The Cash Server](#13-the-cash-server)
14. [The Bug Graveyard](#14-the-bug-graveyard)
15. [What Works Now](#15-what-works-now)
16. [What I Learned](#16-what-i-learned)

---

## 1. Why

Hyper Heroes was a pinball-RPG thing by Allstar Games, later Republished by 90km then purchased by another Chinese company and by the end of this mess abandoned. The game hasn't received an updated since October 2022.
So the plan was simple: reverse engineer the entire client-server communication and build a replacement server that makes the game think nothing changed. Locally without messing around with their servers, their data and avoiding getting a "warn" from the company.

DISCLAIMER:
BEFORE DOING ALL THIS, I'VE REQUESTED PERMISSION, SENT EMAILS, TALKED TO PEOPLE AND MADE EVERYTHING OK SO WE COULDN'T TOUCH ANY SENSITIVE STUFF, BREAK ANYTHING GOING ON OR U KNOW THE DRILL....

---

## 2. Tearing the APK Apart

First thing I did was apktooled this:

```powershell
apktool d hyper-heroes.apk -o hyper-heroes-original
```

Out comes the guts:

```
hyper-heroes-original/
├── AndroidManifest.xml
├── assets/
│   ├── aa/Android/          # Unity Addressables — catalog.json, settings.json, 35 .bundle files
│   ├── bin/Data/             # global-metadata.dat lives here (IL2CPP string table)
│   ├── AtlasData/            # sprite atlases
│   └── Font/, UITexture/     # fonts, UI textures
├── lib/arm64-v8a/
│   └── libil2cpp.so          # <-- THE PRIZE. All game logic compiled to ARM64.
├── smali/, smali_classes2/   # Java/Kotlin SDK wrappers (boring)
└── res/                      # Android resources
```

The moment I saw `libil2cpp.so` in `lib/arm64-v8a/`, I knew what we were dealing with. Unity IL2CPP — all the C# game code compiled down to native ARM64 machine code. No nice decompilable Mono DLLs for us. But there's a trick.

IL2CppDumper takes the binary + the metadata file and spits out the original class structure:

```
Il2CppDumper.exe libil2cpp.so global-metadata.dat output/
```

And just like that, we got `dump.cs` — **600,000 lines** of C# class definitions. Every class, every method, every field, with their RVAs (addresses in the binary). This file became the map for everything that followed.

It immediately revealed the architecture: `NRpcNetwork` for TCP transport, `NRpcParams` with static Encrypt/Decrypt methods (hello encryption keys >>> not putting any here, it's on the apk released BTW not my fault), `PlayerInfo`, `OwnedCardInfo` with 55+ protobuf fields, `CommonUserDataResponse` with 51 fields... the entire game laid bare in type signatures.

Other stuff checked was... The assetbundles did contained A LOT of rewards and prize data... Meaning that, the game basically ran locally and checked here and there with the Authoritative server to know if the X transaction was okay.

<img width="1662" height="797" alt="image" src="https://github.com/user-attachments/assets/814c3eba-16c1-4d0b-8bdc-fe66c6a542b9" />

Examples above xd.
---

## 3. Sniffing the Wire

To see what the game actually says over the network, I set up Burp Suite as an HTTP proxy:

```powershell
adb shell settings put global http_proxy 192.168.0.5:8080
```

Had to install Burp's CA cert on the device, enable legacy TLS (the game used old ciphers), allow unsafe renegotiation — the usual dance with old mobile apps.

Fired up the game and caught the HTTP bootstrap sequence:

```
GET /notice.txt                                          → notice-hczz-oversea.oss-us-east-1.aliyuncs.com (empty)
GET /Addressables/.../catalog_2022.08.24.06.02.11.hash   → resource.hyperheroes.net (MD5: 79a50b6a...)
GET /Addressables/.../catalog_2022.08.24.06.02.11.json   → resource.hyperheroes.net (full asset catalog)
GET /serverlist/api/queryList                             → serverlist.hyperheroes.net (THIS IS THE KEY)
GET /login/guest/guestlogin                              → guest login
GET /globalservice/account/accountList                   → XD SDK account list
GET /myip                                                → myip.hyperheroes.net:1053
```

The `queryList` response was gold — it told the client exactly where to connect: `loginProxy` (TCP address for the game server), `resourceUrl` (CDN for assets), `noticeUrl`, etc. Control this endpoint and you control where the game connects.

Inside the APK's `catalog.json`, I found the game downloads **35 Addressable bundles** from `http://resource.hyperheroes.net/Addressables/ServerData/P21/Android/` and **126 update AssetBundles** from a versioned path. The update manifest (`NBundleInfos.bytes`) was a dead-simple text format, same as I saw previously:

```
-1@ElementName@FileName@MD5Hash@Extension@FileBytes
```

I don't know why even though the game servers were dead they kept the CDN alive. Amazon S3/CloudFront doesn't care if the game studio stopped paying for game servers — the static assets stay up until someone explicitly removes them. I had a window to mirror everything before it disappeared YAYYY! (So I did it obviously... Game does it to the client, I just did it to my PC LoL).

But here's the thing — Burp only showed HTTP traffic. After `queryList`, the game opened raw TCP connections to ports 9150, 9151, 9152. That traffic? Invisible to Burp. That was the game's **NRpc protocol** — protobuf payloads wrapped in DES-CBC encryption... Saw a bunch of bytes here and there, hex code going up and down... but in my mind I was thinking (Is the game downloading anything yet? I didn't know they had this protocol going on early in this stage).

How could you know exactly the data? WIRESHARK THAT BITCH!

<img width="1569" height="905" alt="image" src="https://github.com/user-attachments/assets/063b9dfb-9108-42cd-86f3-c59d787aa5e3" />

Basically I've enforced my local server to be at 05 and my phone to be at 04. (NOTE THIS)
---

## 4. Cracking the Protocol

Switched to Wireshark for full packet capture:

```
adb shell tcpdump -w /sdcard/capture.pcap
```

The PCAPs showed the raw TCP stream. Cross-referencing with `dump.cs`, the NRpc framing fell out:

```
[varint32(len+1)] [Segment 0: Header — protobuf, plaintext]
[varint32(len+1)] [Segment 1: Body — protobuf, DES-CBC encrypted]
[varint32(len+1)] [Segment 2: CommonResponse — DES-CBC encrypted]  ← responses only
[0x00]            ← packet terminator
```

Each segment length is a varint32 encoding `actual_size + 1`. The `0x00` terminator works because a zero-length segment would encode as `1`, so `0` is unambiguous as "end of packet". Elegant.

The encryption? DES-CBC. Found the key and IV straight in `dump.cs` under `NRpcParams`:

```
Key: 7e9ac962  (8 bytes UTF-8)
IV:  01c20de0  (8 bytes UTF-8)
```

Later confirmed these with Frida hooks on the actual Encrypt/Decrypt calls. They matched.

The header (segment 0) is always plaintext protobuf:

```
field 1 (varint) = error code (0 = OK)
field 2 (string) = RPC method name — "XDLogin", "HeartBeat", "CommonUserData", etc.
field 3 (string) = request token (UUID, ties request to response)
```

The body (segment 1) is method-specific protobuf, encrypted. Every method's fields map directly to the classes in `dump.cs`.

To decode the PCAPs, I wrote a series of Node.js scripts:

```javascript
// decode_pcap.js  — basic NRpc decoder, generic protobuf + DES-CBC
// decode_pcap2.js — full varint32 framing parser, processes entire PCAP files
// decode_pcap3.js — tuned for the second capture session
// decode_lotto.js — focused on gacha response dissection
```

These became the Rosetta Stone. I could see exactly what the original server sent for every single RPC call. Every field, every value. This was the blueprint for the replacement server.

---

## 5. The IL2CPP Dump — Mapping the Game's Brain

The 600k-line `dump.cs` was basically a decompiled class reference for the entire game. Here's what mattered:

```csharp
// === Network layer ===
// NRpcNetwork / NRpcNetworkEx — TCP transport + serialization
// NRpcParams — DES-CBC (static Encrypt/Decrypt with hardcoded key/IV)
// DelegatePushRPC — server-to-client push notifications

// === Data structures (all protobuf-serialized) ===
// PlayerInfo — userId, nick, level, rmb (diamonds), gold
// OwnedCardInfo — 55+ fields: actorId, quality, star, stats, equipped items
// CommonUserDataResponse — 51 fields: the EVERYTHING response after login
// LevelEndResponse — battle result, drops, XP, updated cards
// LottoResponse / OpenLottoHmiResponse — gacha system

// === Gameplay lifecycle ===
// NContainerLobby — Awake → Start → InitFunctionMap (lobby boot sequence)
// UIGuide — Open(introId) / Close / CloseAndCompleteIntro (tutorial state machine)
// NManagerGame — global singleton, holds playerInfo, SubPlayerRmb (spend diamonds)
// NManagerChannel — billing layer: Pay, PayOver, IsGuest
```

Every method in the dump had its RVA — the address in `libil2cpp.so` where that method lives. These became Frida hook targets:

```
NRpcParams.Encrypt              → 0x26CBA54
NRpcParams.Decrypt              → 0x26CBC5C
NRpcNetworkEx.RequestRpc        → 0x210AE98
NRpcNetworkEx.OnPackedReceived  → 0x210C354
NContainerLobby.Awake           → 0x21830CC
UIGuide.Open                    → 0x298D510
NDebug.LogError                 → 0x237A4AC
```

I also ran Ghidra with IL2CppDumper's scripts (`ghidra.py`, `ghidra_with_struct.py`) to import the type definitions and do static analysis on the binary. This revealed the IL2CPP object memory layout:

```c
// Il2CppString: klass(8) + monitor(8) + length(i32@0x10) + chars(utf16@0x14)
// Il2CppArray:  klass(8) + monitor(8) + bounds(ptr@0x10) + max_length(i32@0x18) + data(@0x20)
// List<T>:      klass(8) + monitor(8) + _items(ptr@0x10) + _size(i32@0x18)
```

---

## 6. The Emulator Problem (Spoiler: It Doesn't Work)

First attempt: hook `libil2cpp.so` on an x86_64 Android emulator (BlueStacks, Android Studio AVD). Seemed reasonable — run the game, attach Frida, read traffic.

Tried it. Instant crash:

```
SIGSEGV (Bad access due to invalid address)
```

Every. Single. Time.

Here's why: `libil2cpp.so` is compiled for ARM64. On an x86_64 emulator, it runs through **houdini** (Intel's ARM-to-x86 binary translation layer). Frida's Interceptor works by rewriting instructions at the target address — but houdini is translating those instructions on the fly. The memory layout isn't what Frida expects. The rewrite corrupts the translated instruction stream and the process dies.

I tried everything. Direct hook on `libil2cpp.so` — SIGSEGV. `Module.findExportByName` — IL2CPP doesn't use standard ELF exports. Java-layer hooks — can't reach NRpc, it's all native. ARM64 system image on AVD — painfully slow, still unstable.

Dead end. No workaround exists. If you want to hook IL2CPP, you need real ARM64 hardware. Period.

---

## 7. Going Physical — Motorola One Vision

Pulled out a **Motorola One Vision** (ARM64, Android 9). Native ARM64 — no translation layer, no houdini nonsense.

Set up Frida:

```powershell
adb push fridaserver/frida-server-17.8.2-android-arm64/frida-server /data/local/tmp/
adb shell chmod 755 /data/local/tmp/frida-server
adb shell su -c "/data/local/tmp/frida-server &"
```

```powershell
pip install frida-tools==17.8.2
frida -U -l  # verify — lists device processes
```

First hook on the real device — `NRpcParams.Encrypt` at RVA `0x26CBA54`:

```javascript
Interceptor.attach(base.add(0x26CBA54), {
    onEnter: function(args) {
        var data = readByteArr(args[0]);
        console.log("ENCRYPT plaintext: " + toHex(data));
    }
});
```

Worked. First try. Clean attach, no crash, plaintext NRpc data streaming across the console. The black box was open.

With a physical device on WiFi, the network setup changed. PC (server) at `192.168.0.5`, Motorola at `192.168.0.X` via DHCP. Had to set `PUBLIC_HOST` on the server and update build scripts to use the LAN IP instead of `10.0.2.2` (the emulator's host alias). Small price for actually working hooks.

---

## 8. Frida Instrumentation — hhproxy.js

The instrumentation script went through three generations.

`hhproxy_aggressive.js` — first attempt. Hooked everything: libc, Java, IL2CPP, Unity engine internals. Caused random crashes and instability. Learned the hard way that less is more.

`hhproxy_full.js` — intermediate. Narrowed down to game-relevant targets, still too many hooks.

`hhproxy.js` — the final version. **21 hooks + 4 diagnostic**, all purely passive (read-only). No libc hooks. No Java replacements. No Unity engine hooks. Just silent observation.

Here's what it watches:

```
Game logger      NDebug.LogError (throttled + deduplicated, no spam)
Requests         RequestRpc + RequestRpc(List) — method name + token
Crypto           NRpcParams.Encrypt/Decrypt — plaintext pre-encrypt, post-decrypt
Responses        OnPackedReceived — header parse + segment count
Callbacks        ResponseCallback — which handler gets which response
Push             DelegatePushRPC.Invoke + InitPushCallBack
Connection       SendConnect + OnLog
HeartBeat        throttled: logs #1, #2, then 1 every 30
Tutorial         UIGuide.Open/Close/CloseAndCompleteIntro — every step, every introId
Lobby lifecycle  NContainerLobby.Awake/Start/InitFunctionMap
Billing          IsGuest (forced false) + PayOver (forced success)
Java HTTP        URL.openConnection — every URL the game touches
LevelEnd debug   4 hooks on the b__0 callback for crash diagnosis
```

This was the single most valuable tool in the entire project. Decrypted NRpc in real time. Logged every tutorial step by introId so I could replicate the exact sequence. Showed which field/index caused `ArgumentOutOfRangeException` in LevelEnd callbacks. Revealed the exact lifecycle order: `Awake → Start → InitFunctionMap → [RPCs] → UIGuide.Open(introId=2) → ...`

```powershell
frida -U -f com.nkm.kp.hh -l MOD/hhproxy.js
frida -U -f com.nkm.kp.hh -l MOD/hhproxy.js > execution.log  # persist everything
```

---

## 9. Building the Resource Server

With the original servers dead, the game can't do anything. Can't download the Addressables catalog. Can't fetch asset bundles. Can't complete the HTTP bootstrap. Can't establish NRpc sessions. Total brick.

So I built a unified Node.js server (`resource-server/server.js`) that replaces everything. One process, 6+ ports:

```
Port 3000               HTTP    Main — assets, catalogs, HTTP bootstrap
Ports 80/9050/1888/1053  HTTP    Auxiliary bootstrap (replicating original host ports)
Port 9150               TCP     NRpc Lobby — login, player data, battles
Port 9151               TCP     NRpc Game — real-time battle zone
Port 9152               TCP     NRpc Chat
```

**First priority: mirror the assets before the CDN dies.**

```powershell
cd resource-server
node download-assets.js    # reads catalog.json, downloads all 35 Addressable bundles
node sync-updates.js       # downloads NBundleInfos manifest, fetches 126 update AssetBundles (8 parallel workers)
```

Everything saved to `mirror/`. CDN goes down tomorrow? Don't care.

The HTTP side handles every endpoint the game expects:

```javascript
// /notice.txt                          → empty (no announcements)
// /activities*.html                    → local notice page
// /serverlist/api/queryList            → THE KEY: redirect loginProxy + resourceUrl to our server
// /serverlist/api/checkServerStatus    → always online
// /login/guest/guestlogin              → code=0&ext=<MD5> (guest account)
// /globalservice/account/accountList   → XD SDK account listing
// /myip                                → client's IP
// /login/check/getExt                  → SHA-256 hash
// /Addressables/.../*.json             → patched catalog (URLs rewritten to point at us)
// /Addressables/.../*.hash             → static hash (forces catalog refresh)
// /Addressables/.../*.bundle           → serves 35 mirrored bundles
// /<ver>/<ch>/Android/*.assetbundle    → serves 126 update bundles
// /<ver>/<ch>/Android/*.bytes          → update manifest
// ANY *                                → catch-all 200 empty (never 404, game chokes on 404s)
```

The catalog patching is the slick part. The original `catalog.json` has URLs pointing to `resource.hyperheroes.net`. When the game requests it, the server intercepts and rewrites every URL on the fly to `http://<PUBLIC_HOST>:3000/...`. No need to touch the APK's copy (though `build.ps1` does that too as a fallback).

---

## 10. Implementing the NRpc Protocol

The core of the whole project. Had to reimplement the game's proprietary binary protocol from scratch.

**The codec** (`nrpc-codec.js`):

```javascript
const { encodePacket, NRpcStreamDecoder } = require('./nrpc-codec');

// Build and send a 3-segment response
const packet = encodePacket([headerBuf, bodyBuf, commonResponseBuf]);
socket.write(packet);

// Incremental stream decoder (handles TCP fragmentation gracefully)
const decoder = new NRpcStreamDecoder();
socket.on('data', (chunk) => {
    const packets = decoder.push(chunk);
    for (const segments of packets) { /* process */ }
});
```

Handles varint32 framing (`value = segmentLength + 1`), `0x00` terminators, and TCP stream reassembly (chunks can split mid-varint, mid-segment, anywhere).

**Encryption:**

```javascript
// DES-CBC — ancient, insecure, don't care. It's what the game uses.
// Node.js v17+ blocks DES by default → --openssl-legacy-provider
const key = Buffer.from('7e9ac962', 'utf8');  // 8 bytes
const iv  = Buffer.from('01c20de0', 'utf8');  // 8 bytes
```

**Protobuf — done by hand.** No `.proto` files exist anywhere. Rebuilt the serialization manually:

```javascript
function pbFieldVarint(fieldNum, value) { /* ... */ }
function pbFieldBytes(fieldNum, buf) { /* ... */ }
function pbFieldString(fieldNum, str) { /* ... */ }
```

Every field was mapped by triangulating three sources: field names from `dump.cs`, actual values from decoded PCAPs, and live decrypted traffic from Frida hooks. Tedious but it works.

**Implemented methods** (grew organically — each one added when the tutorial hit it):

```
XDLogin                — PlayerInfo + token + zone addresses + UserGuide (the big one)
HeartBeat              — header-only ACK, no body
CommonUserData         — 51-field mega-response: config, heroes, inventory, levels, chat, lotto
LevelData              — empty for new players
StartLevel             — CardInfo + OwnedCardInfo + SyncHeart
LevelEnd               — compound RPC: result + drops + XP + inventory + MapData (3 segments)
UpdateUserGuide        — persists the tutorial bitmask
UpdatePlayerTeam       — saves team composition
PlayerTeam             — returns saved team
LoginChat              — canLogin=true
OnePieceEquipItem      — equip 6 item slots + consume from inventory
QualityUp              — promote hero quality (10→20→21→30...)
OpenLottoHmi           — gacha pool: 8 heroes + LottoRank
Lotto                  — ShowNewCard + OwnedCardInfo + hero persistence
GetMonthOperActivityDot — activity indicator (field 1 = false, NOT field 127)
PlayerActorListOwnedCardAfter — full updated card list
+18 stubs              — header OK + minimal body (enough to not crash)
```

Server-side state, in-memory per guest account (no database, no persistence needed):

```javascript
const accountState = new Map();
// Per UUID:
// {
//   ownedCards:  Map<cardId, { actorId, quality, star, equippedSlots, stats }>,
//   inventory:   Map<itemId, { stuffId, count }>,
//   userGuide:   bitmask (tutorial progress),
//   team:        [cardId, cardId],
//   nextCardId:  auto-increment,
//   nextStuffId: auto-increment
// }
```

---

## 11. Patching the APK

The game has hostnames hardcoded in two places. `catalog.json` / `settings.json` have the Addressables URLs in plain text — easy to patch. But `global-metadata.dat` has the bootstrap hostnames baked into the IL2CPP string table as binary data. That's the tricky one.

`build.ps1` handles the whole pipeline.

**Steps 1-2: patch the text files** (trivial string replace):

```
http://resource.hyperheroes.net/Addressables/ServerData/P21/Android
→ http://192.168.0.5:3000/Addressables/ServerData/P21/Android
```

**Step 3: patch `global-metadata.dat`** (the hard part):

Strings in `global-metadata.dat` have fixed length. Replace `serverlist.hyperheroes.net` (26 bytes) with `192.168.0.5` (11 bytes) and you corrupt the entire metadata file. The replacement MUST be byte-for-byte the same length.

The trick: **sslip.io**. A DNS service that resolves any IP embedded in the hostname:

```
serverlist.hyperheroes.net  (26 bytes) → serve.192-168-0-5.sslip.io  (26 bytes)  ✓
server-kp.hyperheroes.net   (25 bytes) → serv.192-168-0-5.sslip.io   (25 bytes)  ✓
myip.hyperheroes.net        (20 bytes) → 192-168-0-5.sslip.io        (20 bytes)  ✓
```

`New-FixedLengthRedirectHost` generates a prefix padded to exactly the right byte count. `Replace-ByteSequence` does byte-by-byte binary search and swap in the metadata file.

**Steps 4-6: rebuild, align, sign:**

```powershell
apktool b hyper-heroes-original -o hyper-heroes-patched.apk
zipalign -f 4 hyper-heroes-patched.apk hyper-heroes-aligned.apk
apksigner sign --ks debug.keystore --ks-pass pass:android hyper-heroes-aligned.apk
```

v2 signature required for Android 7+ (API 24+). The original APK used v1+v2.

There's also `build-cash.ps1` — a variant that keeps original CDN URLs for assets but redirects only the billing host (`39.107.237.64:8080 → 192.168.0.5:3001`). Useful when you want real CDN assets but fake billing.

---

## 12. Tutorial Hell

The Hyper Heroes tutorial is a nightmare. Rigid, strictly server-orchestrated state machine. Every step depends on the previous one. Server sends wrong data at any point? Client freezes, crashes, or loops forever.

The full sequence I mapped through Frida (every line was pain):

```
 1. HTTP bootstrap: queryList → guestlogin → accountList → checkServerStatus → myip
 2. TCP Lobby (9150): SendConnect → XDLogin → CommonUserData
 3. TCP Chat (9152): SendConnect → LoginChat
 4. Lobby RPCs: OpenTreasurePage, GetFriends, GetBlackList, SyncRankingAward,
    LevelData×2, SyncEvent, SyncOperActivity, GetPlayerEvent (stubs OK)
 5. Continuous HeartBeats (Lobby 9150 + Chat 9152)
 6. Tutorial guides: introId 4899→4913 (intro cutscene)
 7. Lobby init: Awake → Start → InitFunctionMap
 8. GetMonthOperActivityDot, GetCeremonyExchange, etc.
 9. UIGuide introId=2 → UpdateUserGuide → introId=3 → PlayerTeam → UpdatePlayerTeam
10. LevelStart (levelId=300101) → Tutorial battle guides (1001→1012)
11. LevelEnd (compound RPC, 3 segments) → drops + inventory + XP
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

To make this work, I had to extract config data from the game's own CSV tables (pulled from AssetBundles using UABEA):

```
DataPlayerActorItemConfig.csv  — actorId_quality → [6 itemIds] per hero/tier
DataItem.csv                   — item table (id, name, type, stackSize)
DataLevel.csv                  — level config (id, drops, energy)
DataLevelBossDrop.csv          — drop tables: levelId → [{itemId, count}]
DataEquipment.csv              — equipment stats
```

**The inventory pre-seed trap.** The original server didn't start you with an empty inventory. It pre-loaded the items needed to equip both starter heroes at quality 10:

```
Hero 100003: [800010×2, 800015×2, 800004×1, 800110×1]
Hero 100017: [800009×2, 800008×2, 800000×1, 800006×1]
```

Without these, the tutorial hard-locks at the equipment step. It tells you to equip items you don't have. Took a while to figure out these items weren't supposed to come from drops — they're just... there, from the start.

**The gacha config trap.** `CommonUserData` has to include a `SyncLottoTimeResponse` inside `SyncConfigResponse` (field 1). Without it, the gacha module's `m_lottoArray` stays null and the lotto callback throws `NullReferenceException`. The response needs:

```
SyncLottoTimeResponse {
  F1 = SecsToGoldFree (0 = free pull ready)
  F2 = SecsToRmbFreeSecs (0 = free pull ready)
  F3 = repeated LottoAwards [Gold (type=1), RMB (type=2)]
  F4 = repeated AllLottoPoolActors [8 heroes with actorId]
}
```

---

## 13. The Cash Server

The game's billing went through a separate server at `39.107.237.64:8080`. Dead now, obviously.

`cash-server/server.js` — standalone Express server in permanent FORCE_SUCCESS mode. Never contacts upstream. Every request gets a fake success with maxed values:

```javascript
const MAX = 2147483647; // INT_MAX

function makeFakeSuccess(pathname) {
    const p = pathname.toLowerCase();

    // Balance check? You're rich.
    if (p.includes('balance') || p.includes('diamond')) {
        return { code: 0, data: { balance: MAX, rmb: MAX, diamond: MAX } };
    }

    // Purchase? Done. Still rich.
    if (p.includes('topup') || p.includes('purchase')) {
        return { code: 0, data: { orderId: 'LOCAL_' + Date.now(), status: 1, rmb: MAX } };
    }

    // VIP? Level 10, no expiration.
    // Any field named rmb/diamond/crystal/gem/coin/balance → INT_MAX
    // ...
}
```

Client-side Frida hooks complete the picture:

```javascript
// NManagerChannel.Pay()      → auto-triggers PayOver(EProductResult.Success=1)
// NGooglePay.purchase()      → blocked via Java hook (no Play Store UI)
// NManagerChannel.IsGuest    → forced false (unlocks purchase flow)
// PlayerInfo.get_Rmb()       → always returns INT_MAX
// NManagerGame.SubPlayerRmb  → no-op (diamonds never decrease)
```

Port 3001. `node cash-server/server.js`. Done.

---

## 14. The Bug Graveyard

Each of these took hours to find and minutes to fix. Classic.

**#1 — SIGSEGV: The Missing Third Segment**

Server sends `[header, body]` → client dies instantly. Turns out NRpc ALWAYS sends 3 segments for responses: `[header, body, commonResponse]`. The third is an encrypted protobuf blob: `{1: {1:1, 2:0, 4:0}}`. Without it, deserialization reads past the segment array. Unmapped memory. SIGSEGV.

Fix: `COMMON_RESPONSE_RAW` constant + `buildCommonResponseSeg()`. Every RPC response (except XDLogin, HeartBeat, Chat RPCs) now sends all three.

**#2 — LevelEnd: "Index was out of range"**

Complete level 300101, callback `b__0` crashes with `ArgumentOutOfRangeException`. The callback does `for i in GetDropItem: GetOwnedStuffInfo(i)`. Level 300101 drops `[{800010, ×2}, {800010, ×2}]` — two entries for the same item. I merged them into one inventory entry. Callback expected two.

Rule: `count(StuffInfo[]) == count(LevelDropItem[])`. Always. One StuffInfo per drop line. Not per unique item. If two drops reference the same itemId, emit two StuffInfo entries pointing to the same stuffId.

**#3 — GetMonthOperActivityDot: Wrong Field**

Lobby crashes on load. Generic stub returns `pbFieldVarint(127, 0)`. Client expects `pbFieldVarint(1, 0)` — field 1, isHasActivity = false. Field 127 means nothing to the parser.

One line fix. Separated from the stub group.

**#4 — OnePieceEquipItem: Where Are My Items?**

Tutorial freezes at equipment step. Inventory's empty. The quality-10 equip items aren't drops — they're supposed to be pre-seeded at account creation. Nobody told me.

Fix: pre-seed the inventory with both starter heroes' equipment items on account creation.

**#5 — Double Quality Promotion**

Hero goes from quality 10 straight to 21, skipping 20. `OnePieceEquipItem` was doing an internal promotion (10→20), then `QualityUp` promoted again (20→21). They're separate RPCs. Separate responsibilities.

Fix: strip promotion from `OnePieceEquipItem`. It only equips now. `QualityUp` owns all promotions.

**#6 — Gacha NullReferenceException (Triple Kill)**

Open gacha, client dies. Three things wrong simultaneously:

1. `OpenLottoHmi` returned stub instead of hero pool
2. `Lotto` returned bad StuffInfo without `ShowNewCard`
3. `CommonUserData` missing `SyncLottoTimeResponse` → `m_lottoArray` = null

All three had to be fixed together. `OpenLottoHmi` now returns 8 heroes. `Lotto` got rewritten with proper `ShowNewCard` + `OwnedCardInfo` + persistence. `CommonUserData` got the lotto config stuffed into `SyncConfigResponse`.

---

## 15. What Works Now

Full guest login. 35 Addressables + 126 AssetBundles served locally. Patched catalog with dynamic URL rewriting. 15+ HTTP bootstrap endpoints. Full NRpc protocol across Lobby, Game, and Chat. Complete tutorial from intro cutscene through battles, equipment, quality promotion, gacha. Equipment system. Quality promotion. Working gacha. Infinite diamonds and gold. Purchases without Google Play. Fake billing server. 25 passive Frida hooks. Automated build pipeline — patches, rebuilds, signs the APK in one script.

```
┌──────────────────────────────────────────────┐
│           Motorola (ARM64, Android)          │
│  ┌────────────────────────────────────────┐  │
│  │  Hyper Heroes (patched APK)           │  │
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
│  │  └── HTTP :3001 (fake billing)         │  │
│  └────────────────────────────────────────┘  │
└──────────────────────────────────────────────┘
```

---

## 16. What I Learned

**Emulators are useless for IL2CPP hooking.** The houdini/ndk_translation layer doesn't just slow things down — it makes Frida's Interceptor corrupt memory. Real ARM64 silicon or nothing.

**The third NRpc segment exists and it's mandatory.** Without the `commonResponse` blob, SIGSEGV. Didn't show up obviously in the PCAPs — looked like "just another few bytes." Hours of staring at hex dumps to figure out.

**Array counters must match exactly.** LevelEnd callback does `for i in dropItems: stuffInfos[i]`. Merged duplicate drops into fewer StuffInfo entries thinking I was being smart? Congrats — out-of-bounds crash. One StuffInfo per drop line. Always.

**PCAPs + IL2CPP dump + Frida = the holy trinity of mobile RE.** PCAPs show what the wire looks like. The dump shows what the code expects. Frida shows what actually happens at runtime. You need all three. Two out of three and you're guessing.

**The tutorial is an unforgiving server-side state machine.** Each `UpdateUserGuide` persists a bitmask. Get it wrong, tutorial loops or soft-locks. The server must track every guide completion exactly as the original did.

**Document everything.** The 800+ line `README.md` and this write-up exist because past-me kept forgetting why things were done a certain way. Next NRpc method that needs implementing? Pattern's already here.

**Mirror assets while you still can.** Game servers die first. CDNs (S3, CloudFront) survive longer — just static file hosting, nobody remembers to turn it off. That window closes eventually. Mirror early, mirror everything.

---

*Phyper — March 2026*
