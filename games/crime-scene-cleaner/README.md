# Crime Scene Cleaner (2024)

| | |
|---|---|
| Engine | Unity 2021.3.45f2, IL2CPP, URP (BepInEx 6 bleeding edge, build be.791) |
| Versions tested | Steam, app 1040200, buildid 26046462, Windows 10; `CrimeCleaner.exe` SHA-256 `e9167a32...4f560173`, `GameAssembly.dll` SHA-256 `8af483d7...ae999fa3847` (full hashes in [NOTES.md](NOTES.md)) |
| Projects | CSC Coop prototype (this write-up; source not published yet) |
| QA level | none (bot-driven two-copy runs read by hand, plus the owner's two-account sessions; no pass/fail live suite) |
| Last updated | 2026-10-23 |

## Where things stand

A BepInEx plugin lets 2 players (designed for up to 4) cleaning the same job see each other and share every grabbable item over Steam. The owner played three sessions with a friend on two real Steam accounts. Players appear as cyan capsules, and grabbing, carrying and throwing items stays in sync. Stains, bodies, doors, quest progress and the job report are **not** synced. Each player still runs their own mission, so this is not yet co-op play of the game.

## Existing work

- No co-op or multiplayer mod was found. Only one general web search was run (2026-10-23), and the owner reports no prior attempt. GitHub, Nexus, Steam Workshop and Discord were **not** searched.
- [Crowd Control's Crime Scene Cleaner guide](https://crowdcontrol.live/guides/crimescenecleaner) shows that a BepInEx-based mod for the Steam build exists. That was the only hint that BepInEx loads on this game.

## How the game works (for co-op)

- **Engine:** Unity 2021.3.45f2, compiled with IL2CPP (`GameAssembly.dll` plus `il2cpp_data/Metadata/global-metadata.dat`, metadata version 31). Rendering is URP. Libraries in the build: Steamworks.NET, Rewired, Wwise, DOTween, Easy Save 3, PixelCrushers Dialogue System, Odin, Cinemachine, UniRx, Addressables.
- **Modding:**
  - BepInEx 6.0.0-pre.2 does **not** load. Its Cpp2IL supports IL2CPP metadata 23-29, not 31, and the game exits at once.
  - BepInEx 6 bleeding-edge be.791 (`builds.bepinex.dev`) works. Its first start generates the interop assemblies (about a minute).
  - Cpp2IL 2022.1.0-pre-release.21 dumps the metadata on its own.
- **Netcode:** none. No Mirror, Photon, Netcode for GameObjects or UNet types are in the metadata. `steam_api64.dll` is loaded and initialised by the game itself, with the game's real app id.
- **Game logic:** split between `MainGame.dll` and `Assembly-CSharp.dll`. Classes useful for co-op:
  - `FirstPersonController`: `CanMove` becomes true when the mission intro cutscene ends, and `IsBlocked` also exists.
  - `GrabController`: `GrabbedRb`, `GrabItem(item)`, `TryRelease(Vector3 force)` (a throw), `TryRelease()`.
  - `GrabableItemController`: `Rb`, `startingWorldPosition`, `IsHeld`.
  - `BodyRagdollController`, `CleaningTarget`, `BloodTarget`, `MissionController`, `Missions`, `SaveSystem`, `PlayerSave`.
- **Saves:** Easy Save 3 files are written **inside the install** (`CrimeCleaner_Data/Saves/<steam id>/Slot1.es3` and `SavesBackups0/`), not under `%USERPROFILE%\AppData\LocalLow`. That folder (`President Studio/CrimeCleaner`) holds only `Player.log`. A copied install therefore has fully separate saves.
- **Starting a job without menus:** `SaveSystem.Instance.StartGame(1)` from the main menu (`MenuScreen_Nightmare` scene) loads slot 1. The test save went `LoadingScreenScene` → `HubResidence-GameIntro` → the "Dealer's Apartment" job (554 `GrabableItemController` objects).
- **Several copies on one PC:** a hard-linked second install runs alongside the first. No single-instance mutex or window check was seen. Only `BepInEx/`, `CrimeCleaner_Data/Saves`, `PlayerPrefs.es3`, `KeyBindingSaveFile.es3` and `doorstop_config.ini` need real copies.

## What we changed

The co-op layer is a BepInEx plugin (`CscCoop.dll`) plus a game-free sync core (`CoopCore.dll`) with its own test program.

1. **Transport.**
   - Players connect through `ISteamNetworkingMessages`, called with the flat C API on the `steam_api64.dll` the game has already initialised.
   - Every player owns the game, so the real app id is used and the [Spacewar](../../playbook/networking-steam-spacewar.md) trick is not needed.
   - Steam callbacks are not used: the game keeps calling `SteamAPI_RunCallbacks` itself. Sessions are accepted up front for every SteamID listed in `BepInEx/coop.cfg`: `peers=` on the host, `host=` on clients.
   - A loopback UDP transport with the same interface is used for local tests.
   - x64 message layout confirmed with a send-to-self test: data at offset 0, size at 8, sender identity at 16.
2. **Players.**
   - Each player sends their position and yaw at 20 Hz. The host relays poses to everyone, and the newest sequence number wins.
   - **Bug:** remote players drew only a shadow. `GameObject.CreatePrimitive` uses the built-in `Standard` shader. Unity reports it as supported, but this URP game renders only its shadow pass. Fix: a material from `Shader.Find("Universal Render Pipeline/Unlit")`, coloured cyan. If that shader is missing, a URP material is copied from the scene.
   - **Bug:** a player's capsule stayed after they left. Fix: a bye message sent on quit, plus a 5 s silence timeout. A restarted player is accepted again from sequence 1.
3. **Items.**
   - **Authority:** the host is authoritative. Items are identified by their index in a table sorted by name and `startingWorldPosition`. Both PCs logged the same table hash, `5962476C` for n=554.
   - **Holding:** the holder owns an item; the host simulates everything else.
   - **Versions:** the host stamps every update with a version. Clients send a clock-seeded proposal counter, so duplicate and late client packets are dropped.
   - **Bug:** thrown items dropped straight down on the other PC. Updates had no velocity, and the receiver zeroed it. Fix: updates carry linear and angular velocity, and the release packet is resent 4 times.
   - **Bug:** a client's own throw stuttered.
     - Cause, measured with a per-packet trace: the host's copy trails by about a round trip, 5-17 cm in flight, and the client snapped back to each of the host's updates.
     - Fix: for 5 s after a local throw, the client ignores the host while either copy still moves. Skipped updates are not recorded as known, so the host's final state still arrives.
   - **Bug:** after a throw, the item jumped a short distance once it stopped. Two PCs' physics land it in slightly different places. Fix: if both copies are at rest and 5 cm to 1.5 m apart, the client slides the item to the host's spot over 0.2-0.6 s (smoothstep, kinematic while sliding).
   - **Leaving:** a player who leaves has their held items freed by the host.

## QA

- **Game-free core tests:** `dotnet run -c Release` in the core's test project. 34 checks, ALL PASS (2026-10-23). They cover:
  - codecs, and rejection of bad packets and NaN values
  - 1 host + 3 clients on loopback with 30% receive loss: every peer converges on every other player's final pose
  - hold exclusivity, duplicate and late releases, host simulation after a throw
  - 50 items converging on 4 replicas at 30% loss
  - late-join snapshot, release-all on leave, rejoin with a restarted counter
- **Bot-driven two-copy runs on one PC** (loopback UDP; a script launches both and prints the logs; results read by hand, not asserted):
  - A bot starts the job through `SaveSystem.StartGame(1)` and waits for `CanMove`.
  - Movement: the host bot walks, and the client logs the moving pose.
  - Ghost: the capsule's viewport position is logged, and a screenshot is taken when it is centred. The screenshot shows a solid cyan capsule.
  - Items: the host moved an item and the client logged it in the same place. The client grabbed, carried and threw crates through the game's own `GrabItem`/`TryRelease`. During one throw the client applied 0 corrections from the host, and both copies ended within 1 cm. A forced 0.4 m landing gap slid closed in 0.29 s, with a largest per-frame step of 3.1 cm.
  - Leaving: a normal host quit logged "player 0 left (bye)", and a killed host logged "left (timeout)".
- **Real Steam, two accounts** (owner and a friend, reported by the owner, 2026-10-23):
  - capsules visible
  - items synced, throws land in the same place, the thrower's throw smooth
  - capsule removed on quit, held items free to take
  - landing jump gone after the slide fix

## How to play

There is no release yet. The test kit is BepInEx be.791 plus the two plugin DLLs plus a `coop.cfg`, unzipped into the game folder. Each player sets `transport=steam`. The host sets `role=host` and `peers=<friends' SteamID64s>`; each friend sets `role=client`, `id=1..3` and `host=<host SteamID64>`. A player's SteamID64 is printed to `BepInEx/LogOutput.log` on start. Everyone then starts the same job.

## Known issues

- Only player positions and grabbable items are shared. Stains, blood, bodies, doors, tools, garbage bags, the ozonator, quest progress, money and the job report are each player's own.
- Players see each other as plain capsules, with no character model or animation.
- No lobby or invite: SteamID64s are typed into a config file by hand.
- The client trusts its own claim of who sent a packet. That is fine between friends, but there is no protection against a cheating player.
- Every player must load the same job from a save that is in the same state. The item table only matches if the scene and its items are the same.

## Next steps

1. **Bodies:** `BodyRagdollController` and `BodyPartController`, which use their own grab path (`GrabController.TryGetBody`).
2. **Cleaning state:** `CleaningTarget` and `BloodTarget`. Every stain cleaned on one PC must disappear on the others. This is monotonic, so it fits numbered, acknowledged events (see [effect-capture-story-sync](../../playbook/effect-capture-story-sync.md)).
3. **Quest progress and the end-of-job report:** `MissionController`, the `MainGame` quest events.
4. **Lobbies and invites** instead of hand-typed SteamIDs. Then a real 3- and 4-player test.
5. **A pass/fail live suite:** turn the current log-reading runs into asserted checks, so the QA level can become `automated`.
