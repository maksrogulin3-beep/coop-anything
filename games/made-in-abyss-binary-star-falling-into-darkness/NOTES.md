# Made in Abyss: Binary Star Falling into Darkness — NOTES

Claim issue: https://github.com/cranebeaker/coop-anything/issues/186
Branch: game/made-in-abyss-bsfd

## Game identity
See local/notes-made-in-abyss.md (private). Summary:
- Steam AppID 1324340, buildid 10136809, Windows x64, Steam PC.
- Unreal Engine 4 shipping build; WindowsNoEditor pak naming; exact minor version unknown.
- Shipping exe: MadeInAbyss-Win64-Shipping.exe, 78310168 B,
  SHA256 5277EDFDEAF579C14355074F6D5E2E54CA14810DF1CA32899A7D7F3B4F7F1A80.
- Launcher exe: MadeInAbyss.exe (250 KB), SHA256 CEB2F2F313B8C4E979D7D05221D9FA433846A66313DE905F7FF8A44B44D72A97.

## Observations (checked)
- <game dir> contains Engine/, Licenses/, MadeInAbyss-BSFD/, MadeInAbyss.exe.
- MadeInAbyss-BSFD\ contains Binaries\Win64 (shipping exe + OpenAL32.dll), Content\Paks
  (MadeInAbyss-BSFD-WindowsNoEditor.pak ~19.2 GiB, _0_P ~9.0 GiB), usound\.
- No version resource and no ++UE4 string in ASCII scan of the shipping exe.

## Sources
- appmanifest_1324340.acf on <game dir>'s Steam library
- https://store.steampowered.com/app/1324340/ (game page, Steam AppID)
- https://www.nexusmods.com/games/madeinabyssbinarystarfallingintodarkness (21 mods, none co-op)
- https://github.com/Lequla/MiA-BSFDAccessoryArtifacts (pak-mod README: pak names with
  WindowsNoEditor suffix; base game versions 1.0.1-1.0.3+)
- https://www.reddit.com/r/MadeInAbyss/comments/1i7fz70/ (community co-op request, no answers)

## Derived
- WindowsNoEditor pak naming => UE4 packaged with a launcher-era UE4 (4.25 or earlier).
  Engine minor version pending check.
- UE4SS supports UE 4.7 through 5.8 per the repo playbook, so any of these minor versions
  should be moddable through UE4SS (Lua + SDK dump).

## Unknowns
- Exact UE4 minor version (pak footer or BuildVersion check).
- Save location/format (expect %LOCALAPPDATA%\MadeInAbyss\Saved\<store>\..., to confirm).
- Which mode to co-op first: DEEP IN ABYSS (custom delver, open-ended loop) is the
  likely first target vs HELLO ABYSS (story mode with Reg as companion).
- Whether the game has engine-native online plumbing (OnlineSubsystem) detectable in the
  shipping build.

## Next steps
- Python install (download in progress), then: MCPServer tools/, repo tool tests (test_all.py).
- Pak-footer check to pin engine version, then UE4SS experiment on a test copy.

## 2026-10-25 Stage 1 session 2 notes
- Licenses folder confirms integrated Angelscript (Hazelight plugin, MIT) and standard 'UnrealEngine.txt' copyright 1998-2022 => engine <= 4.27.
- ThirdParty layout: PhysX3 (<=4.26), Steamworks Steamv147, XAudio2_9 (>=4.24), BlastPlugin (>=4.24) => best guess UE 4.25/4.26. Ping again with real version.
- Pak footer mystery: no 0x5A6F5AE1 magic in last 64KB of either pak; q fields (929, 929) + index hash at start => looks like index-at-start layout; needs repak/UE-Viewer to confirm.
- Ice plan next: download repak + UE4SS (pinned, owner-approved), dump scripts, find .as files.

## 2026-10-25 Stage 1 session 3 notes (pak + Angelscript + UE4SS verdict)

### Tools (pinned, SHA-256 verified)
- repak v0.2.3, hash 6720d602...a73fa (GitHub API asset digest), installed to tools\repak.exe.
- UE4SS experimental-latest (v3.0.1-1181-g83330635), hash b9a166a7...d7b9 (GitHub API asset digest).

### Pak facts (checked)
- Both paks ARE standard UE paks readable by repak: version V9 / major "FrozenIndex",
  index NOT encrypted, no compression, mount `../../../`, 94877 entries (base) + 12023 (patch).
- Full file lists saved to %TEMP%\pak_base_list.txt / pak_patch_list.txt (94k/12k lines).

### Angelscript (checked — game logic is moddable as text)
- `MadeInAbyss-BSFD/Script/**` shipped INSIDE the pak as **plain text .as sources** (214 files:
  Core 24, Scenario/Map 53, Scenario/Deep 53, Scenario/Hello 24, Examples 18, DebugMenu 14, Test 2),
  plus Script/Binds.Cache + Binds.Cache.Headers (Hazelight binding caches).
- Engine/Plugins/Angelscript/Angelscript.uplugin: Hazelight plugin v0.5, runtime modules
  AngelscriptCode + AngelscriptLoader (Loader = Runtime => VM runs in shipping).
- DefaultGame.ini: `+DirectoriesToAlwaysStageAsUFS=(Path="../Script")` => scripts load from UFS/pak.
- Core scripts use the full plugin API (UPROPERTY/UFUNCTION/BlueprintOverride, Gameplay:: helpers,
  fibers). Config level: custom collision channels PawnPlayer/PawnCharacter already exist.
- Hazelight plugin networking: docs confirm UFUNCTION(NetMulticast/Client/Server) RPCs and
  UPROPERTY(Replicated) work in the plugin (current docs; v0.5-era support to verify by test).
- Hazelight's source repo went private (404 as of 2025-10 per reddit thread + our check).

### UE4SS verdict (checked, 6 launches)
- UE4SS loads, PS scan succeeds, detects engine 4.25/shipping, FName verified. Then the GAME
  process always dies 2-4s after `Locating KismetStringLibrary CDO...`.
- Crash is INSIDE UE4SS.dll at +0x4286DA (minidump parsed: c0000005 read at 0x1700040000,
  twice identical). Deterministic on repeat runs.
- Tried: bUseUObjectArrayCache true/false, bForceGUObjectArrayForIteration true/false,
  DefaultFNameToStringMethod Scan/Conv_NameToString. Same crash every time.
- Direct exe launch also exits with code 53 sometimes (Steam DRM context), but via
  `steam://rungameid/1324340` the injection happens too, so DRM exit is a separate nuisance.
- WITHOUT UE4SS: game runs fine (5+ min, healthy CPU/RAM; graceful close worked).
  CE: some launch runs with UE4SS left a crash_*.dmp in ue4ss/; only run-level symptom, game
  itself is not crashing unmodded.
- UE4SS upstream has a near-identical open issue for a customized-engine game (#1442, Little
  Nightmares II EE UE4.26, no workaround in thread).
- UEPseudo submodule (deps/first/Unreal of UE4SS @83330635) is private => cannot map fault
  offset to source ourselves.
- UE4SS parked at <owner coop-work>/ue4ss-parked (dwmapi.dll + ue4ss/); install dir clean again.

### Game run facts (checked)
- Launch: works via Steam URL and direct shipping exe (with -windowed). No %LOCALAPPDATA% game
  dir created yet; no UE4-style log found after 5 min runtime (log may be disabled in
  shipping config or written elsewhere; `unknown` until a longer full-map session).

### Conclusions / direction
- Stage 1 "code can run in the game" has TWO candidate mechanisms:
  (a) UE4SS — blocked by UE4SS.dll crash specific to this custom engine build;
  (b) Angelscript pak patch — REPLACES/EXTENDS game logic from a pak override, no injection
  needed, and the VM provably runs in shipping. (b) is the primary path going forward.
- Cheap decisive experiment next: build a test patch pak (e.g. _1_P.pak) overriding one
  Script/Core file to emit a log/print line, verify the game picks it up. Reversible:
  adding a NEW pak file, original paks untouched, deletable.

## Next steps
- Verify Angelscript pak override mechanism (repak pack a minimal patch pak).
- Determine save location/format (needs a longer in-game session, owner or scripted).
- Confirm plugin v0.5 script API surface (subsystem auto-load? net features?) via Examples scripts
  shipped in pak.
- Optional: re-run remaining C++ tool suites now that Build Tools is installed.

## 2026-10-25 Stage 1 session 4 notes (script compilation and network surface)

### Checked
- A deliberately invalid `CoopBootstrap.as` produced the game's own Angelscript compile
  diagnostic at startup. This proves the patch pak's `.as` source is discovered and compiled.
- The diagnostic was removed by deleting the test pak; a clean game launch was then verified.
- Valid source overrides of `ScenarioActor.as` and `ASEventLevelActor.as` compiled without the
  diagnostic, but their `BeginPlay` probes were not displayed in a live map. This is consistent
  with the map's pre-baked Blueprint/generated classes not being replaced merely by a source file
  with the same script name. Do not claim source override execution from this test.
- `Binds.Cache` contains real bindings for `Gameplay::SaveGameToSlot`, `UGameplayStatics`,
  `NewObject`, `UMIAGameInstance::GetGameInstance`, and the game's custom classes.
- `Binds.Cache` contains Steam session callback proxies (`CreateSession`, `FindSessions`,
  `JoinSession`, `DestroySession`) and `USteamNetDriver`; bindings also expose `SetReplicates`
  and `SetReplicateMovement`.
- `DefaultEngine.ini` selects `BP_MIAGameInstance` and `BP_AbyssPlayerController`, but no explicit
  `GameNetDriver`/Steam net-driver definition was found in the extracted config. This indicates
  Steam networking is packaged but not enabled for normal single-player startup; verify before
  building the session layer.

### Current decision
- Stop random `.as` source probes. The next implementation target is the real network/config path:
  Steam session discovery, net-driver configuration, then a replicated second pawn. Existing
  Blueprint assets/classes must be treated as the authoritative spawn path.
- No experimental pak is currently installed in the owner's game directory.

