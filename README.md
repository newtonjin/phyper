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
666. [GGS!](#GGs!-I-can-play-locally-now!)
14. [What I Learned](#14-what-i-learned)

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

The encryption? DES-CBC. Found the key and IV straight in `dump.cs` (Plaintext btw):

Censoring those okay? just the tip so you know we real!

```
Key: 7e9xxxxxxxxxx  (8 bytes UTF-8)
IV:  01xxxxxxxxxxxx (8 bytes UTF-8)
```

Later confirmed these with Frida hooks on the actual Encrypt/Decrypt calls. They matched.

The header (segment 0) is always plaintext protobuf:

```
field 1 (varint) = error code (0 = OK)
field 2 (string) = RPC method name — "XDLogin", "HeartBeat", "CommonUserData", etc.
field 3 (string) = request token (UUID, ties request to response)
```

The body (segment 1) is method-specific protobuf, encrypted. Every method's fields map directly to the classes in `dump.cs`.

To decode the PCAPs, I wrote a series of Node.js scripts (Node cuz I'm lazy and to speed up asked some AI to do a .js for me, btw the AI was lazier and couldn't understand the technical requirements.... It was a nightmare):

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

## 6. The Emulator INCIDENT (Spoiler: It Doesn't Work)

First attempt: hook `libil2cpp.so` on an x86_64 Android emulator (BlueStacks, Android Studio AVD). Seemed reasonable — run the game, attach Frida, read traffic.

Tried it. Instant crash:

```
SIGSEGV (Bad access due to invalid address)
```

Every. Single. Time.

Here's why: `libil2cpp.so` is compiled for ARM64. On an x86_64 emulator, it runs through **houdini** (Intel's ARM-to-x86 binary translation layer). Frida's Interceptor works by rewriting instructions at the target address — but houdini is translating those instructions on the fly. The memory layout isn't what Frida expects. The rewrite corrupts the translated instruction stream and the process dies.

I tried everything. Direct hook on `libil2cpp.so` — SIGSEGV. `Module.findExportByName` — IL2CPP doesn't use standard ELF exports. Java-layer hooks — can't reach NRpc, it's all native. ARM64 system image on AVD — painfully slow, still unstable.

Dead end. No workaround exists. If you want to hook IL2CPP, you need real ARM64 hardware. Period.
My solution for it?
I bought a trash phone from the times where this game was just released and root it... GOOOD LOOORD!!!! Probably that was the hardest part in this entire process... U guys have no Idea (or maybe do) of HOW MUCH TIME AND EFFORT IT IS NOWADAYS TO ROOT A PHONE!!!!!! Last time I did this on a Android device we had to download idk, superRoot.apk or shit like this, it was 4 minutes and BOOM, roooot! Now I had to redo the boot.img!!!!!!

<img width="1200" height="1600" alt="image" src="https://github.com/user-attachments/assets/b1da95ff-f7e6-4bb8-bc04-b2c14a06d7d2" />


---

## 7. Going Physical — Motorola One Vision

Pulled out the craziest sexiest mf 2016ish **Motorola** (ARM64, Android 9). Native ARM64 — no translation layer, no houdini nonsense.

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

With a physical device on WiFi, the network setup changed. PC (server) at `192.168.0.5`, Motorola at `192.168.0.X` via DHCP. Had to set `PUBLIC_HOST` on the server and update build scripts to use the LAN IP instead of the emulator's host alias. Small price for actually working hooks.

20$ USD btw, the phone was 20$.

---

## 8. Frida Instrumentation — hhproxy.js

The instrumentation script went through three generations.

`hhproxy_aggressive.js` — first attempt. Hooked everything: libc, Java, IL2CPP, Unity engine internals. Caused random crashes and instability. Learned the hard way that less is more.

```
Log result example:
[HHPROXY/GLOG] [WARN] \u542f\u52a8 Lua \u865a\u62df\u673a
[HHPROXY/GLOG] [ERROR] [Effect/vfx_activity_slotsmachine] load error!!!
[HHPROXY/GLOG] [ERROR] [UIMaterial/AlphaTexture] load error!!!
[HHPROXY/GLOG] [ERROR] [UI/Lobby/UILobbyPlayer] load error!!!
[HHPROXY/GLOG] [ERROR] [UI/Lobby/UILobbyMenu] load error!!!
[HHPROXY/GLOG] [ERROR] [UI/Lobby/Lua/UILuaChat] load error!!!
[HHPROXY/GLOG] [ERROR] [UI/Lobby/UILobbyMap] load error!!!
[HHPROXY/GLOG] [ERROR] [Actor/Barrier] load error!!!
[HHPROXY/GLOG] [ERROR] [GuildBattle/FogQuad] load error!!!
[HHPROXY/GLOG] [ERROR] [Model/aideha] load error!!!
[HHPROXY/GLOG] [ERROR] [Game/AttackRadiusUI] load error!!!
[HHPROXY/GLOG] [ERROR] [SceneAssemble/Effects/level00_fog] load error!!!
[HHPROXY/HTTP] >> http://server-kp.hyperheroes.net:9050/login/guest/guestlogin?guest=a163dad3-b3bd-472e-9e99-48ce980b4f4e&channel=600001
[HHPROXY/GLOG] [ERROR] [ModelFBX/aideha] load error!!!
[HHPROXY/HTTP] >> http://activation.hyperheroes.net:1888/globalservice/account/accountList?xdId=a163dad3-b3bd-472e-9e99-48ce980b4f4e&md5Str=B327F475087FBAD14EF2A07C9DBE27E4
[HHPROXY/HTTP] >> http://serverlist.hyperheroes.net//serverlist/api/checkServerStatus?channel=600001&platform=2&version=1.1.80&serverId=1110&xdId=a163dad3-b3bd-472e-9e99-48ce980b4f4e
[HHPROXY/HTTP] >> http://resource.hyperheroes.net/2021120801/600001/Android/NBundleInfos.bytes?v=34
[HHPROXY/HTTP] >> http://myip.hyperheroes.net:1053/myip
[HHPROXY/HTTP] >> http://server-kp.hyperheroes.net:9050/login/check/getExt?xdId=a163dad3-b3bd-472e-9e99-48ce980b4f4e&channelId=600001
[HHPROXY/CONN] >> SendConnect(game-na.hyperheroes.net:9150)
[HHPROXY/NETLOG] SendConnect host = game-na.hyperheroes.net port = 9150
[HHPROXY/PUSH] >> InitPushCallBack delegate=0x6e502a4300
[HHPROXY/PUSH]    method_ptr = 0x6e93442c00
[HHPROXY/PUSH]    -> libil2cpp.so+0x2702c00
[HHPROXY/PUSH]    -> HOOKED push handler at 0x6e93442c00
[HHPROXY/SEND] >>> XDLogin token=46b6bbb7-3a7b-4490-89a5-9895de894f1b
[HHPROXY/SEND] >>> XDLogin (prebuilt)
[HHPROXY/ENC] >>> [XDLogin] plain 130B: 0a 24 61 31 36 33 64 61 64 33 2d 62 33 62 64 2d 34 37 32 65 2d 39 65 39 39 2d 34 38 63 65 39 38 30 62 34 66 34 65 12 20 38 36 34 38 46 46 41 32 35 45 41 42 46 33 39 46 42 44 41 39 31 45 36 45 30 42 42 42 32 44 42 34 1a 34 08 02 12 06 31 2e 31 2e 38 30 1a 24 62 64 36 30 37 36 65 65 2d 36 63 62 35 2d 34 38 65 33 2d 38 61 65 65 2d 31 31 64 64 38 36 35 66 62 38 36 66 30 eb fd 3f 28 01 30 1c
[HHPROXY/GLOG] [WARN] SendConnect host = game-na.hyperheroes.net port = 9150
[HHPROXY/NETLOG] OnConnected Finish \u8fde\u63a5\u6210\u529f!
[HHPROXY/GLOG] [WARN] OnConnected Finish \u8fde\u63a5\u6210\u529f!
[HHPROXY/HB] >> HeartBeat #1
[HHPROXY/RECV] <<< ? err=0 token=46b6bbb7-3a7b-4490-89a5-9895de894f1b segs=2
[HHPROXY/DEC] <<< [?] 334B: 0a cb 02 0a 68 08 fd e0 02 12 06 34 35 31 38 31 26 18 00 22 04 63 30 30 32 2a 06 63 32 62 67 6b 31 40 01 48 00 50 00 58 00 60 00 70 00 78 b9 17 80 01 00 88 01 00 90 01 00 98 01 00 b0 01 00 b8 01 00 c0 01 00 c8 01 f5 03 d0 01 00 d8 01 00 e0 01 00 e8 01 00 f0 01 00 f8 01 00 80 02 00 88 02 01 90 02 00 98 02 00 a0 02 00 a8 02 00 12 12 08 3c 10 02 18 00 20 fc 81 d9 cd 06 28 00 30 3c 38 00 1a 0e 08 0a 10 02 18 00 20 fc 81 d9 cd 06 28 00 22 0a 08 32 10 00 18 00 28 00 30 32 28 b0 f3 fe ff ff ff ff ff ff 01 78 00 8a 01 15 67 77 2d 6e 61 2e 68 79 70 65 72 68 65 72 6f 65 73 2e 6e 65 74 90 01 b8 49 98 01 d6 08 a2 01 2b 31 31 31 30 5f 34 35 31 38 31 5f 37 65 63 63 65 61 62 32 64 35 65 62 34 33 64 61 38 61 66 65 36 30 39 66 33 34 32 39 36 63 66 34 aa 01 00 b0 01 a0 86 da cd 06 ba 01 1b 63 68 61 74 2d 63 6c 69 65 6e 74 2e 68 79 70 65 72 68 65 72 6f 65 73 2e 6e 65 74 c0 01 d2 47 d2 05 20 35 42 35 33 41 42 36 41 37 41 34 34 41 46 43 33 31 39 39 41 42 44 36 32 39 41 35 32 30 34 37 46 a0 06 fb 81 d9 cd 06
[HHPROXY/CB] <<< CALLBACK: ?(raw=0x6e502a4080) error=0
[HHPROXY/HTTP] >> https://e.tapdb.net/identify
[HHPROXY/HTTP] >> https://e.tapdb.net/event
[HHPROXY/SEND] >>> CommonUserData token=f0669b5f-6371-4734-9842-ff5cfb0b0358
[HHPROXY/SEND] >>> CommonUserData (prebuilt)
[HHPROXY/ENC] >>> [CommonUserData] plain 2B: 08 01
[HHPROXY/RECV] <<< ? err=0 token=f0669b5f-6371-4734-9842-ff5cfb0b0358 segs=3
[HHPROXY/DEC] <<< [?] 6495B: 0a de 08 0a b7 05 08 00 10 00 1a 16 08 01 10 8b db 35 18 00 20 05 28 a0 9c 01 30 a0 fe 0a 38 01 40 09 1a 16 08 02 10 8c db 35 28 f0 01 30 f0 10 38 01 40 09 48 00 50 00 58 00 1a 10 08 03 10 8d db 35 28 a0 06 30 90 1c 38 01 40 09 1a 13 08 04 10 f4 db 35 28 50 30 80 05 38 01 40 08 60 00 68 00 1a 12 08 05 10 f4 db 35 28 c8 01 30 a0 06 38 01 40 08 70 00 22 0a 08 84 8e 06 10 e0 c6 88 e0 05 22 0a 08 a0 8d 06 10 e0 ab 98 b4 05 22 0a 08 b8 8d 06 10 e0 ab 98 b4 05 22 0a 08 f5 8d 06 10 e0 c6 88 e0 05 22 0a 08 a7 8d 06 10 e0 ab 98 b4 05 22 0a 08 83 8e 06 10 e0 ab 98 b4 05 22 0a 08 f0 8d 06 10 e0 c6 88 e0 05 22 0a 08 f8 8d 06 10 e0 c6 88 e0 05 22 0a 08 e6 8d 06 10 e0 ab 98 b4 05 22 0a 08 b4 8d 06 10 e0 ab 98 b4 05 22 0a 08 b2 8d 06 10 e0 ab 98 b4 05 22 0a 08 d7 8d 06 10 e0 c6 88 e0 05 22 0a 08 b7 8d 06 10 e0 ab 98 b4 05 22 0a 08 c7 8e 06 10 e0 ab 98 b4 05 22 0a 08 e5 8d 06 10 e0 ab 98 b4 05 22 0a 08 dc 8d 06 10 e0 ab 98 b4 05 22 0a 08 bf 8d 06 10 e0 ab 98 b4 05 22 0a 08 a2 8d 06 10 e0 ab 98 b4 05 22 0a 08 bd 8d 06 10 e0 ab 98 b4 05 22 0a 08 f2 8d 06 10 e0 c6 88 e0 05 22 0a 08 bb 8d 06 10 e0 ab 98 b4 05 22 0a 08 d9 8d 06 10 e0 c6 88 e0 05 22 0a 08 ba 8d 06 10 e0 ab 98 b4 05 22 0a 08 be 8d 06 10 e0 ab 98 b4 05 22 0a 08 b0 8d 06 10 e0 ab 98 b4 05 22 0a 08 f4 8d 06 10 e0 ab 98 b4 05 22 0a 08 da 8d 06 10 e0 ab 98 b4 05 22 0a 08 a8 8d 06 10 e0 ab 98 b4 05 22 0a 08 81 8e 06 10 e0 ab 98 b4 05 22 0a 08 ad 8d 06 10 e0 ab 98 b4 05 22 0a 08 ff 8d 06 10 e0 ab 98 b4 05 22 0a 08 a1 8d 06 10 e0 ab 98 b4 05 22 0a 08 ab 8d 06 10 e0 ab 98 b4 ...(6495B)
[HHPROXY/DEC] <<< [?] 8B: 0a 06 08 00 10 00 20 00
[HHPROXY/CB] <<< CALLBACK: ?(raw=0x6e503c6a80) error=0
[HHPROXY/CONN] >> SendConnect(chat-client.hyperheroes.net:9170)
[HHPROXY/NETLOG] SendConnect host = chat-client.hyperheroes.net port = 9170
[HHPROXY/PUSH] >> InitPushCallBack delegate=0x6e4f827f00
[HHPROXY/PUSH]    method_ptr = 0x6e93442c74
[HHPROXY/PUSH]    -> libil2cpp.so+0x2702c74
[HHPROXY/PUSH]    -> HOOKED push handler at 0x6e93442c74
[HHPROXY/SEND] >>> LoginChat token=ad160fec-be7c-4209-a6ea-de37c6ad63fa
[HHPROXY/SEND] >>> LoginChat (prebuilt)
[HHPROXY/ENC] >>> [LoginChat] plain 682B: 0a 0a 31 31 31 30 5f 34 35 31 38 31 12 9b 05 7b 22 63 6f 6e 74 65 6e 74 22 3a 22 22 2c 20 22 75 73 65 72 49 64 22 3a 22 34 35 31 38 31 22 2c 20 22 75 73 65 72 4c 65 76 65 6c 22 3a 22 31 22 2c 20 22 75 73 65 72 4e 61 6d 65 22 3a 22 34 35 31 38 31 26 22 2c 20 22 76 69 70 4c 65 76 65 6c 22 3a 22 30 22 2c 20 22 61 6c 6c 46 69 67 68 74 69 6e 67 4e 75 6d 22 3a 22 32 32 33 32 22 2c 20 22 67 75 69 6c 64 49 64 22 3a 22 30 22 2c 20 22 67 75 69 6c 64 4e 61 6d 65 22 3a 22 22 2c 20 22 7a 6f 6e 65 49 64 22 3a 22 31 31 31 30 22 2c 20 22 69 63 6f 6e 22 3a 22 63 30 30 32 22 2c 20 22 69 63 6f 6e 4b 75 61 6e 67 22 3a 22 63 32 62 67 6b 31 22 2c 20 22 72 6f 6f 6d 49 64 22 3a 22 22 2c 20 22 6d 73 67 54 79 70 65 22 3a 22 30 22 2c 20 22 74 69 6d 65 53 74 65 6d 70 22 3a 22 22 2c 20 22 6d 61 78 5f 66 69 67 68 74 5f 61 63 74 6f 72 22 3a 22 7b 5c 22 41 63 74 6f 72 31 5c 22 3a 5c 22 7b 5c 5c 5c 22 61 63 74 6f 72 49 64 5c 5c 5c 22 3a 5c 5c 5c 22 31 30 30 30 31 37 5c 5c 5c 22 2c 20 5c 5c 5c 22 61 63 74 6f 72 4c 65 76 65 6c 5c 5c 5c 22 3a 5c 5c 5c 22 31 5c 5c 5c 22 2c 20 5c 5c 5c 22 61 63 74 6f 72 53 74 61 72 5c 5c 5c 22 3a 5c 5c 5c 22 31 5c 5c 5c 22 2c 20 5c 5c 5c 22 61 63 74 6f 72 51 75 61 6c 69 74 79 5c 5c 5c 22 3a 5c 5c 5c 22 31 30 5c 5c 5c 22 7d 5c 22 2c 20 5c 22 41 63 74 6f 72 32 5c 22 3a 5c 22 7b 5c 5c 5c 22 61 63 74 6f 72 49 64 5c 5c 5c 22 3a 5c 5c 5c 22 31 30 30 30 30 33 5c 5c 5c 22 2c 20 5c 5c 5c 22 61 63 74 6f 72 4c 65 76 65 6c 5c 5c 5c 22 3a 5c 5c 5c 22 31 5c 5c 5c 22 2c 20 5c 5c 5c 22 61 63 74 6f 72 53 74 61 72 5c 5c 5c 22 3a 5c ...(682B)
[HHPROXY/SEND] >>> OpenTreasurePage token=a5482458-e467-4272-bdda-45e4c8e461fa
[HHPROXY/SEND] >>> OpenTreasurePage (prebuilt)
[HHPROXY/GLOG] [WARN] SendConnect host = chat-client.hyperheroes.net port = 9170
[HHPROXY/NETLOG] OnConnected Finish \u8fde\u63a5\u6210\u529f!
[HHPROXY/GLOG] [WARN] OnConnected Finish \u8fde\u63a5\u6210\u529f!
[HHPROXY/RECV] <<< ? err=0 token=a5482458-e467-4272-bdda-45e4c8e461fa segs=3
[HHPROXY/DEC] <<< [?] decrypt returned null (enc was 8B)
```

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
frida -U -f com.nkm.kp.hh -l MOD/hhproxy.js > your.log  # persist everything
```

---

## 9. Building the Resource Server

With the original servers almost dead, the game can't do anything. Can't have proper support. Can't fetch account data. Can't complete the HTTP bootstrap (Yep, a lot of issues on the old servers). Can't establish legal requirements. MANY, hear me, MANY spend money in this game on the past, by 2019 the game was already dying when the company simply decided to abadon their FACEBOOK ID LOGIN... What does it mean? The game had only 2 options for logins, guest account or a Facebook SSO. Facebook ID dead, means no one could log in in their account anymore, THOUSANDS AND THOUSANDS WENT TO SPACE to a game with no support. 

So I built a unified Node.js server (`resource-server/server.js`) that replaces everything. One process, 6+ ports:

```
Port 3000               HTTP    Main — assets, catalogs, HTTP bootstrap
Ports 80/9050/1888/1053  HTTP    Auxiliary bootstrap (replicating original host ports)
Port 9150               TCP     NRpc Lobby — login, player data, battles
Port 9151               TCP     NRpc Game — real-time battle zone
Port 9152               TCP     NRpc Chat
```

Everything saved to `mirror/`. CDN goes down tomorrow? Don't care.

The HTTP side handles every endpoint the game expects:

```javascript
// /notice.txt                          → empty (no announcements) We don't actually need this bro...
// /activities*.html                    → local notice page The only news I've added was "not found"
// /serverlist/api/queryList            → THE KEY: redirect loginProxy + resourceUrl to our server LOCAL FOREVER
// /serverlist/api/checkServerStatus    → always online
// /login/guest/guestlogin              → code=0&ext=<MD5> (guest account)
// /globalservice/account/accountList   → XD SDK account listing (Dead at this point since you can't login anyway)
// /myip                                → client's IP (Yours to yourself)
// /login/check/getExt                  → SHA-256 hash
// /Addressables/.../*.json             → patched catalog (URLs rewritten to point at us)
// /Addressables/.../*.hash             → static hash (forces catalog refresh)
// /Addressables/.../*.bundle           → serves 35 mirrored bundles
// /<ver>/<ch>/Android/*.assetbundle    → serves 126 update bundles
// /<ver>/<ch>/Android/*.bytes          → update manifest
// ANY *                                → catch-all 200 empty (never 404, game chokes on 404s)
```

The catalog patching is the slick part. The original `catalog.json` has URLs pointing to `resource.hyperheroes.net`. When the game requests it, the server intercepts and rewrites every URL on the fly to `http://<PUBLIC_HOST>:3000/...`. No need to touch the APK's copy (though I have a recompiled and signed .apk that I changed the IPs manually, you will see just scrolling down a little).

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
const key = CENSORED ALREADY; // 8 bytes
const iv  = ALSO CENSORED ALREADY; // 8 bytes
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


**Steps 1-2: patch the text files** (trivial string replace):

```
http://GAMESSERVERDOMAINNAMEBLABLABLA/Addressables/ServerData/P21/Android
→ http://192.168.0.5:3000/Addressables/ServerData/P21/Android
```

**Step 3: patch `global-metadata.dat`** (the hard part):

Strings in `global-metadata.dat` have fixed length. Replace `serverlist.hyperheroes.net` (26 bytes) with `192.168.0.5` (11 bytes) and you corrupt the entire metadata file. The replacement MUST be byte-for-byte the same length.

The trick: **sslip.io**. A DNS service that resolves any IP embedded in the hostname (This one is cool right? U liked it huh? :D ):

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

WITH ALL THIS DONE>>>>> SERVER STAAARTED>>>>

<img width="913" height="560" alt="image" src="https://github.com/user-attachments/assets/5d8f05a7-f147-4b96-a018-97e0027ddf5c" />

Opening the installed APK>>>>>

<img width="1200" height="1600" alt="image" src="https://github.com/user-attachments/assets/6516821c-5fbe-456e-bd6b-3fabf4fabe75" />

Not found worked....

<img width="1200" height="1600" alt="image" src="https://github.com/user-attachments/assets/49b4c992-2a78-4fc9-a311-3e27fea90209" />

Server is also on>>>

<img width="1200" height="1600" alt="image" src="https://github.com/user-attachments/assets/c027ac12-ea91-4f39-9c83-10685cc0cc30" />



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

The game's billing went through a separate server.... Dead now, obviously.

`cash-server/server.js` — standalone Express server in permanent FORCE_SUCCESS mode. Never contacts upstream. Every request gets a fake success with maxed values:


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

## GGs! I can play locally now!
<img width="1200" height="1600" alt="image" src="https://github.com/user-attachments/assets/94b065fa-951f-42d8-9b7f-4c1f963e35a8" />


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

## 14. What I Learned

**Emulators are useless for IL2CPP hooking.** The houdini/ndk_translation layer doesn't just slow things down — it makes Frida's Interceptor corrupt memory. Real ARM64 silicon or nothing. (Hard to learn)

**Array counters must match exactly.** LevelEnd callback does `for i in dropItems: stuffInfos[i]`. Merged duplicate drops into fewer StuffInfo entries thinking I was being smart? Congrats — out-of-bounds crash. One StuffInfo per drop line. Always.

**PCAPs + IL2CPP dump + Frida = the holy trinity of mobile RE.** PCAPs show what the wire looks like. The dump shows what the code expects. Frida shows what actually happens at runtime. You need all three. Two out of three and you're guessing.

**The tutorial is an unforgiving server-side state machine.** Each `UpdateUserGuide` persists a bitmask. Get it wrong, tutorial loops or soft-locks. The server must track every guide completion exactly as the original did.

**Document everything.** The 800+ line `phypershit.txt` and this write-up exist because past-me kept forgetting why things were done a certain way. Next NRpc method that needs implementing? Pattern's already here.

**Mirror assets while you still can.** Game servers die first. CDNs (S3, CloudFront) survive longer — just static file hosting, nobody remembers to turn it off. That window closes eventually. Mirror early, mirror everything.

---

Newton - Cybersecurity BLABALBALABALABAL AGAIN!

*Phyper — March 2026*
