# FlashForwardTest

Unreal Engine 5.8 Blueprint-only template with a Steam session loop: start menu, host or join a session, a pre-game lobby, then the match.

```
StartMenuLevel (UI_StartMenu)                     # boot map
├── Host        → CreateAdvancedSession → L_Lobby?listen
├── Find / Join → JoinSession → host's L_Lobby
│
L_Lobby (3D room + UI roster)
├── each player = Mannequin display on a slot + roster entry (avatar, name, ready)
├── all ready   → 1.5 s → seamless ServerTravel → Lvl_ThirdPerson
└── Leave       → DestroySession → StartMenuLevel
│
Lvl_ThirdPerson (BP_ThirdPersonGameMode)
```

## Requirements

- Unreal Engine 5.8
- Steam client running and logged in (needed for real names, avatars and invites)
- Plugins:
  - `OnlineSubsystemSteam` and `SteamSockets` (engine plugins, enabled in `FlashForwardTest.uproject`)
  - `AdvancedSessions` and `AdvancedSteamSessions` (bundled in `Plugins/`)

## Setup

### 1. Steam configuration

Put the following code in `Config/DefaultEngine.ini`:

```ini
[/Script/Engine.GameEngine]
!NetDriverDefinitions=ClearArray
+NetDriverDefinitions=(DefName="GameNetDriver",DriverClassName="/Script/SteamSockets.SteamSocketsNetDriver",DriverClassNameFallback="/Script/SteamSockets.SteamNetSocketsNetDriver")

[OnlineSubsystem]
DefaultPlatformService=Steam

[OnlineSubsystemSteam]
bEnabled=true
SteamDevAppId=480
bInitServerOnClient=true

[/Script/OnlineSubsystemSteam.SteamNetDriver]
NetConnectionClassName="OnlineSubsystemSteam.SteamNetConnection"
```

- `SteamDevAppId=480` is Valve's public test app (Spacewar). Replace it with your own App ID before shipping.
- Restart the editor after editing `DefaultEngine.ini`.

### 2. Lobby size

In the same file, cap the session at the number of lobby slots (4):

```ini
[/Script/Engine.GameSession]
MaxPlayers=4
```

### 3. Maps and classes

These are already set in `Config/DefaultEngine.ini`. Check them if you copy the system into another project:

- `GameDefaultMap` and `EditorStartupMap` = `/Game/Steam/StartMenuLevel`
- `GameInstanceClass` = `/Game/Steam/SteamGameInstance.SteamGameInstance_C`
- `L_Lobby` World Settings > GameMode Override = `BP_LobbyGameMode`

### 4. Blueprint RPC flags (manual)

RPC flags on Blueprint custom events have to be set by hand in the editor:

- `BP_LobbyPlayerController` > custom event `Server_ToggleLobbyReady`: Replicates = **Run on Server**, **Reliable**. Without this, a client's Ready click only runs on that client.
- `SteamSeshComp` > custom event `Server_CheckPassword`: Replicates = **Run on Server**, **Reliable**.

## Content

```
/Game/Steam
├── SteamGameInstance            # Host / Join; Host opens L_Lobby?listen
├── SteamSeshComp                # pause menu + session helpers, OnPauseMenuToggled dispatcher
├── StartMenu/                   # UI_StartMenu, UI_SessionButton, start menu GameMode / Controller
├── PauseMenu/                   # UI_PauseMenu (Continue, Invite, Main Menu, LeaveToMainMenu), UI_FriendButton
└── Lobby/
    ├── L_Lobby                  # lobby map: 4 BP_LobbySlot, LobbyCamera (tag "LobbyCamera")
    ├── BP_LobbyGameMode         # slots, displays, ready check, seamless ServerTravel
    ├── BP_LobbyGameState        # bTravelPending (RepNotify), roster / travel dispatchers
    ├── BP_LobbyPlayerState      # bIsReady (RepNotify), LobbySlotIndex (Replicated)
    ├── BP_LobbyPlayerController # lobby camera, lobby UI, input mode, Server_ToggleLobbyReady
    ├── BP_LobbySlot             # placement marker with SlotIndex (left to right from the camera)
    ├── BP_LobbyDisplay          # replicated Mannequin idle + name / ready labels
    ├── UI_Lobby                 # roster, status, Ready / Invite Friends / Leave
    └── UI_LobbyPlayerEntry      # avatar, name, ready state
```

## Tuning

Values are Blueprint variables (no literals in the graphs):

| Blueprint | Variable | Default |
|---|---|---|
| `BP_LobbyGameMode` | `MinPlayersToTravel` | 1 |
| `BP_LobbyGameMode` | `TravelDelaySeconds` | 1.5 |
| `BP_LobbyGameMode` | `MatchMapPath` | `/Game/ThirdPerson/Lvl_ThirdPerson` |
| `BP_LobbyGameMode` | `LobbyDisplayClass` | `BP_LobbyDisplay` |
| `BP_LobbyPlayerController` | `CameraTag` | `LobbyCamera` |
| `BP_LobbyPlayerController` | `CameraBlendSeconds` | 0 |
| `BP_LobbyDisplay`, `UI_LobbyPlayerEntry` | `LabelRefreshInterval` | 0.5 |
| `UI_Lobby` | `WaitingStatusText`, `StartingStatusText` | status messages |

To add slots, place more `BP_LobbySlot` actors in `L_Lobby` with the next `SlotIndex`, and raise `MaxPlayers` to match.

## Testing

- **PIE, one player:** open `L_Lobby`, Play as Listen Server, click **READY**. The match map loads after 1.5 s.
  - Seamless travel in PIE needs `net.AllowPIESeamlessTravel 1` in the console.
- **PIE, two players:** Number of Players = 2, Net Mode = Play As Listen Server. Travel waits until both are ready.
- **Steam:** launch Standalone (or a packaged build) with Steam running, then Host from the start menu and join from a second machine or Steam account.
- In the editor and PIE, Steam falls back to the NULL subsystem (`Unable to create OnlineSubsystem instance Steam`). This is expected; real names and avatars only appear in Standalone.

## Known limitations

- Ready state is not carried into the match.
- The session stays advertised after travel, so a late joiner lands directly in `Lvl_ThirdPerson`.