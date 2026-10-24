# Crime Scene Cleaner - notes

## Identity (observed, read-only)
- Title: Crime Scene Cleaner (President Studio, PlayWay publisher), Steam appid 1040200
- Steam buildid: 26046462 (appmanifest, LastUpdated epoch 1792525833)
- CrimeCleaner.exe SHA-256: e9167a329f7c8b1c9eef4d0a8dee6d6b08acdee0ce265f1e78ca474f4d560173
- GameAssembly.dll SHA-256: 8af483d77daca17387f4875638f180072195e0f0c8541e9f1e7e8ae999fa3847
- Engine: Unity 2021.3.45f2, IL2CPP (GameAssembly.dll + il2cpp_data/Metadata/global-metadata.dat), 64-bit Windows
- Plugins seen: Rewired, Wwise (AkSoundEngine), Burst, steam_api64.dll (no obvious netcode lib), Easy Save 3 (.es3 saves)
- OS: Windows 10 Pro 10.0.19045

## Observations
- No Mono Managed/ folder: scripts are native IL2CPP, need Il2CppDumper/Cpp2IL-style metadata dump.
- Web search (standard, 1 query) found no co-op mod. Only a Crowd Control (BepInEx) streamer mod, which shows BepInEx IL2CPP runs on the Steam build. Not yet verified by me.

## Unknown
- Any existing co-op attempt (need to search Discord/Nexus/Workshop/GitHub per research.md)
- Whether game has any shipped multiplayer code (expect none)
- Anti-cheat / online service: none seen in install listing

## Next
- Phase 1 research, then Phase 2 workspace (needs owner OK: copy of 53 GB install, tool downloads)

## Workspace log
- Test copy: testbed/game (robocopy from the Steam install; git-ignored territory, never commit).
- Cpp2IL 2022.1.0-pre-release.21 Windows exe: SHA-256 663fb432...e4e04c3 matches the value published by GitHub for the release asset.
- BepInEx 6.0.0-pre.2 IL2CPP win-x64 zip: SHA-256 616ec7eb06cf11b2a0000e8fcef04d1b12bb58e84a2e0bdac9523234fc193ceb. GitHub publishes no digest, so this is hash-on-first-download, not verified against an upstream value.
- Tools in tools_dl/ (not installed anywhere else).

## Phase 3 findings (observed)
- Test copy verified: exe and GameAssembly.dll hashes identical to the Steam install; 249 entries in CrimeCleaner_Data on both sides.
- Cpp2IL (dummydll output) works: IL2CPP metadata v31, Unity 2021.3.45f2, 127383 methods mapped. Dump in dump/cpp2il (not committed).
- Libraries: Steamworks.NET (com.rlabrecque.steamworks.net.dll), Rewired, Wwise, DOTween, Easy Save 3, PixelCrushers Dialogue System, Odin, Cinemachine, UniRx, Addressables, Unity Localization.
- NO networking library (no Mirror/Photon/Netcode/UNet types in metadata). Networking must be built from scratch.
- Game logic assemblies: MainGame.dll (quests, cleaning targets, ozonator, lockpicking, mission report, dialogue events) and Assembly-CSharp.dll (FirstPersonController, GrabController, GrabableItemController, PlayerInventory, Grabber, BodyRagdollController, GarbageCollector, Missions/MissionController, SaveSystem/PlayerSave...).
- Saves live inside the install (CrimeCleaner_Data/Saves, SavesBackups0), so a copy isolates them. The copy contains the owner's save folder (named by Steam id): local only, never commit.
- Baseline SHA-256 of the real save files taken before any launch: local/real_saves_baseline.json.

## Unknown
- Whether BepInEx 6 pre.2 loads on this build (not launched yet).
- Where Player.log / registry PlayerPrefs go (Unity default: AppData\LocalLow\<company>\<product>, HKCU).

## First launch results (test copy, observed)
- BepInEx 6.0.0-pre.2 zip (reports itself as be.697) FAILS: Cpp2IL inside it supports IL2CPP metadata 23-29, this game has 31. The game process exited at once.
- BepInEx 6.0.0-be.791+b782da4 (builds.bepinex.dev, SHA-256 bb1c5109...6f9a, no upstream hash published) WORKS: interop assemblies generated (11606 methods unstripped, 1037 failed, IL bodies 6267 ok / 1170 failed), chainloader started, game stayed alive 90 s. 0 plugins loaded (none written yet). Not yet checked: that the game reached the menu and ran correctly with the hook.
- Launch recipe: env SteamAppId/SteamGameId=1040200, args -screen-fullscreen 0 -screen-width 800 -screen-height 600 -logFile local/player.log, run from testbed/game.
- Real-save check after the run: 34 of 35 baseline files identical. The one changed file is Steam's own <Steam dir>\userdata\<account>\config\librarycache\1040200.json (Steam client library cache, rewritten when a game starts), not a game save. Save folder CrimeCleaner_Data/Saves of the real install was unchanged.

## Probe plugin (observed)
- src/CoopMod (BepInEx IL2CPP plugin, net6.0, builds with `dotnet build -c Release`, deploys to testbed/game/BepInEx/plugins). Loads on be.791; reflection over interop types works.
- Game reached SplashScreen -> MenuScreen_Nightmare (main menu scene) with the plugin running; probe logs every 5 s. No player or grabbables at the menu (expected).
- Public API of key types dumped to dump/api-dump.txt (CleaningTarget, BloodTarget, BodyRagdollController, FirstPersonController, GrabableItemController, GrabController, PlayerInventory, MissionController, Missions, SaveSystem, PlayerSave, Ozonator...).
- Missions.AcceptMission(int) exists: candidate way for a bot to start a mission without menu clicks. Unverified.

## Next (unverified plan)
1. Bot: load a mission via the game's own API, log player position and grabbable counts in a mission scene.
2. Hook points for sync: FirstPersonController (player pose), GrabController/GrabableItemController (items), CleaningTarget/BloodTarget (stains), BodyRagdollController (bodies), MissionController/quest events.
3. Transport: Steamworks.NET already in the game (Steam relay P2P), host authority, up to 4 players.

## Bot reaches a mission (observed)
- Saves are RELATIVE TO THE INSTALL (CrimeCleaner_Data/Saves/<steamid>/Slot1.es3, ES3), not %LocalLow%. persistentDataPath is AppData/LocalLow/President Studio/CrimeCleaner, which holds only Player.log. So the test copy is fully isolated. Copy-side save backup kept in local/copy_saves_backup.
- Bot (src/CoopMod, active only if BepInEx/bot.flag exists): after 40 s on the menu scene calls SaveSystem.Instance.StartGame(1). Result: LoadingScreenScene -> HubResidence-GameIntro (18 grabbables) -> "Dealer's Apartment" mission scene with FirstPersonController at (6.70, 0.04, 28.99) and 554 GrabableItemController objects. Whole run needs ~2 min.
- Real-install save hashes re-checked after these runs (see below).

## Network core, game-free (ran; no game launched)
- src/CoopCore: wire format (pose packet, 26 bytes, sequence numbers, newest wins), RemoteWorld, ITransport, loopback UdpTransport (host relays between clients).
- src/CoopCore.Tests: 1 host + 3 clients on loopback with 30% receive loss, 100 steps of movement plus 40 resend ticks. Result: ALL PASS (15 checks: codec, garbage rejected, stale pose ignored, all 12 peer-to-player final poses converge). This tests the sync core only; nothing is proven yet inside the real game, over Steam, or with items/stains/bodies.
- Not yet done: Plugin integration (read FirstPersonController pose, draw ghost avatars), Steam relay ITransport, item/stain/body sync.

## Two game instances talking (observed, localhost)
- testbed/game2 = hard-linked copy (2410 files linked, 269 real copies: BepInEx, Saves, PlayerPrefs, KeyBindings, doorstop ini). A second process CAN run alongside the first on the same linked files; no mutex/single-instance problem seen. Both used SteamAppId=1040200.
- coop.cfg (BepInEx/coop.cfg) sets role=host|client, id, port. Plugin sends the local FirstPersonController position at 20 Hz over CoopCore UDP; host relays; remote players get a collider-less capsule "ghost".
- Result after ~150 s: both instances reached "Dealer's Apartment" via the bot, and each logged the other's pose (host rx 682 packets, client rx 1128, sequence numbers rising, remote position equal to the other's player position).
- LIMITS: both players stood still at the same spawn point, so this proves the packet path in both directions, not movement tracking. Ghost capsule rendering was not checked on screen. No world sync (items, stains, bodies, quests) yet. Localhost only; Steam relay not written.

## Walking test (observed, localhost, 2 instances)
- Host bot (BepInEx/walk.flag) teleports its player on a circle 3 m in front of player 1 (CharacterController disabled around the write). Client log over ~6 samples: remote position changed every sample (x 7.14 -> 6.87 -> 6.64 -> 6.38 -> 6.20 -> 6.19, z 30.98 -> 31.47). So movement of a remote player IS tracked, on the client side numbers.
- Screenshot from the client (shot.flag, ScreenCapture) shows the mission INTRO CUTSCENE (player sitting in a car), not free movement. So the ghost capsule was NOT seen on screen: the client's camera is a cutscene camera and its player position stays at the car/spawn point. Ghost visibility remains unverified.
- Both instances stay in this intro until it ends; the bot needs to skip/finish the intro (or wait for gameplay control) before ghost and item tests mean anything. Unknown how to detect "intro over" (FirstPersonController.CanMove is a candidate).

## Step results: intro, items, Steam (observed unless marked)
### Walk/ghost
- CanMove turned True after the intro; with the walk bot gated on it, the client log shows the host's ghost at viewport (0.25, 0.19, z 2.4) with Renderer.isVisible = True. A screenshot (local/shot_client_walk.png) was taken but the camera faced a fence/wall and the ghost was not in that frame, so "ghost rendered" rests on the isVisible/viewport log, not on a picture.
### Items (src/CoopMod/ItemSync.cs, src/CoopCore/Items.cs)
- Game-free: item codec, host-stamped versions, holder exclusivity, 50 items converging on 4 replicas at 30% loss via snapshot resend, late-join snapshot: all PASS (ran).
- Live, 2 instances, localhost: item tables identical (n=554, hash 5962476C on both). Host moved #100 (BoxBigOpen) +2 m; both sides then show it at the same place after it fell (-4.14, 0.22, 24.79). Client grabbed #200 (Crate 23) with GrabController.GrabItem, carried it to its player; host saw it at (6.60, 1.75, 29.98) = client value while held; after release the host simulated the fall and both ended at (6.60, 0.22, 29.98).
- Limits: one host item and one client item tested; the carry was a teleport of the held item, not a human hand movement; no conflict test (two players grabbing the same item) in the live game; client owner id is taken from the packet (spoofable, fine for friends); throwing, stacking, containers, garbage bags, bodies, stains, doors, quest state NOT synced.
### Steam transport (src/CoopMod/SteamTransport.cs)
- Uses the steam_api64.dll flat API already initialised by the game (real app id 1040200, so every player owns the game; no Spacewar needed). No callbacks (game's own RunCallbacks keeps running); sessions are accepted up front for every SteamID in coop.cfg (peers=... on host, host=... on client).
- Single-account smoke test inside the real game (ran): interface getters resolved, own SteamID read, SendMessageToUser to self returned EResult 1, the message came back through ReceiveMessagesOnChannel with the right length (3). This confirms calling convention, send flags and message layout (data @0, size @8).
- NOT proven: delivery between two real accounts, whether SteamID decoded from the identity at msg+16 is right (not logged), the accept-up-front assumption (playbook says the first message from a stranger raises callback 1251; if messages are held, the fix is to poll callbacks), lobbies/invites (none written: players exchange SteamID64 by hand), packet size limits (reads cap at 4096).
- Scripts: scripts/run_two.ps1 launches host+client copies and prints logs.

## Two-account Steam test kit (prepared, NOT yet run)
- package/CscCoop-friend.zip (35.5 MB, SHA-256 727a9f2a...091a, built by scripts/make_package.py from package/stage): BepInEx 6 be.791 + CscCoop.dll + CoopCore.dll + coop.cfg (client, id 1, host SteamID64 filled in) + README-FRIEND.txt. No game files. Contains the host's SteamID64 in coop.cfg: do not commit or publish.
- scripts/launch_host.ps1 starts the host test copy (testbed/game, steam transport, no bots).
- Expected evidence: both logs show "[net] steam transport ready", host log gets rx counts from the friend's SteamID, ghost visible, items move. If the friend's packets never arrive, suspect the unaccepted-session case (callback 1251).

## Two-account Steam test (owner + friend, reported by owner, not observed by me)
- Steam transport worked between two real accounts: remote player position updated (seen as a moving shadow) and grabbed items synced.
- Bug 1: ghost capsule invisible, only its shadow drawn. Cause (observed): CreatePrimitive uses the built-in 'Standard' shader; Unity reports it supported=True but this URP game draws only its shadow pass. Fix: material from Shader.Find("Universal Render Pipeline/Unlit") (found in the build), cyan _BaseColor; fallback copies a URP material from the scene. Verified on localhost: screenshot local/shot_ghost_urp.png shows a solid cyan capsule in the client's view.
- Bug 2: a thrown item just dropped on the other PC. Cause: release packet had no velocity and OnPacket zeroed velocity; release was also sent once, unreliably. Fix: ItemState carries linear + angular velocity (61-byte packet); clients send a proposal counter (clock-seeded) in Version and resend the release 4 times; the host drops duplicate/late proposals per sender. Game-free tests: velocity roundtrip, resent release ignored, late held packet ignored, host simulates flight: all PASS.
- Live localhost throw: client threw Crate (23) through GrabController.TryRelease(force); release sent v=(-0.62, 2.84, 5.97); host and client then logged the same trajectory sample by sample (first sample (6.59, 0.22, 30.65), bounce back to (6.91, 0.21, 30.36) on both). Previous build dropped it straight down at (6.60, 0.22, 29.98). The crate hit something about 0.7 m ahead (fence near spawn), so a long free flight was not observed.
- Still not verified: these two fixes over real Steam between two accounts.

## Second Steam session (reported by owner)
- Cyan capsules visible on both PCs; thrown items land in the same place on both.
- Bug 3: client sees a little stutter when IT throws. Bug 4: ghost stays after the host disconnects.

## Fixes (observed on localhost, two copies; not yet re-tested over Steam)
- Leaving: new Bye message (sent 3x from OnApplicationQuit, relayed by host) plus a 5 s silence timeout. On leave: ghost destroyed, RemoteWorld.Forget (a restarted player's seq 1 is accepted again), host frees every item the leaver held (ItemAuthority.ReleaseAll, broadcast). Live: normal host quit -> client logged "player 0 left (bye)", ghost gone; host process killed -> client logged "player 0 left (timeout)", ghost gone. Freed-items path covered only by game-free tests.
- Throw stutter. Measured cause (per-packet trace on the client, one throw): the host's copy of a client-thrown crate trails the client's by about a round trip, 0.05-0.17 m during flight; the client applied each of those 18 updates in 2.1 s, snapping its crate back each time. Fix: for 5 s after a local throw the client ignores host updates while either copy still moves (> 0.2 m/s); updates skipped this way are not recorded, so the host's final resting state (resent by the snapshot) still applies; updates within 5 cm / 3 degrees are not applied at all.
- Live after the fix: two real throws (release v ~ (-0.21, 5.4, 1.99)), 0 host snaps applied during either flight, final positions equal within 1 cm on both copies.
- Measurement mistakes found and fixed on the way (so earlier numbers in this session are invalid): the snap counter ran before the grace check; the walk bot kicked landed crates around; the bot's "carry" put crates on the car roof so some throws never launched (now carried to camera + 1.2 m).
- Game-free tests now 34 checks, ALL PASS.

## Third Steam session (reported by owner)
- Friend's own throws look smooth. Freed items after a player quits: owner could pick them up (ReleaseAll works over Steam). Capsule disappears on quit.
- Bug 5: on the client, a thrown item sometimes jumps a short distance after it stops. Cause: two PCs' physics do not land the item in exactly the same spot; after the throw grace period the client snapped to the host's resting state in one frame. Localhost gaps were < 5 cm (below the correction threshold), so local tests never showed it.
- Fix: when both copies are at rest and differ by 5 cm - 1.5 m, the client slides the item to the host's spot (smoothstep, 0.2-0.6 s, kinematic while sliding; a grab or newer state cancels the slide). Over 1.5 m it still snaps.
- Live localhost check with a forced gap (bot pushed the client's copy 0.4 m sideways mid-flight): "blend #201 done: 0.37 m over 0.29 s, 19 frames, largest step 3.1 cm"; final positions identical on both. Five other items that settled 5-11 cm differently at scene load also slid instead of jumping (largest step 2.4 cm).
