# Api-CA

# AdminConsole — Complete API Reference

Comprehensive documentation of all system modules, functions, and commands.

---

## 1. Architecture

```
AdminCommandHandler
└── Modules
├── AdminAuth          -- calculates a player's access level
├── CommandLevels       -- required level per internal command
├── CommandRegistry     -- engine: tools + command declaration
├── AdminCommandsPack   -- all commands (built-in)
└── Commands            -- entry point, parsing ":cmd arg1 arg2"
```

Execution flow for a command typed in-game:

`player types ":kill me"` → `Commands.Run(admin, ":kill me")` → parses name (`kill`) and args (`{"me"}`) → looks up function in `Registry.GetCommands()` → executes → checks level via `AdminAuth.GetLevel` before calling the actual handler.

---

## 2. `AdminAuth`

Calculates the numerical access level of a `Player`.

### `AdminAuth.DefaultLevel`
`number`. Level assigned to any player who does not match any `Config` entry. Defaults to `0`.

### `AdminAuth.Config.Groups`
List of roles. Each entry:

```lua
{
Role = "CustomName",   -- informational only, never read by the code
Level = 200,          -- level granted if the player has this role
Ranks = {
{ GroupId = 0, RoleId = 1234 },   -- one or more Roblox groups/roles
},
}
```

- `RoleId` = the actual Roblox role ID (not the 0-255 rank), retrievable via `GroupService:GetGroupInfo(groupId)`.
- A single `Role` (same `Level`) can cover multiple `{ GroupId, RoleId }` pairs—meaning multiple different groups. ### `AdminAuth.Config.Players`
List of individual players:

```lua
{ UserId = 9424236449, Role = "Owner", Level = 9999999 }
```

`Role` is purely informational; only `UserId` and `Level` are used.

### `AdminAuth.GetLevel(player: Player) → number`
Returns the highest applicable level for `player`, combining `Config.Players` and `Config.Groups`. Queries `player:GetRankInGroup` and caches the rank-to-RoleId mapping per group (`GroupService:GetGroupInfo`; one network call per group, not per player).

### `AdminAuth.CanUse(player: Player, requiredLevel: number) → boolean`
Shortcut: `AdminAuth.GetLevel(player) >= requiredLevel`.

---

## 3. `CommandLevels`

Static configuration table; contains no functions.

```lua
CommandLevels.DefaultLevel   -- number, level used if a command has no entry (0)
CommandLevels.Commands       -- { [commandName] = { Level = number }, ... }
```

Automatically used by `Registry.Command` when no level is explicitly passed. Applies only to internal commands (`AdminCommandsPack`); an external pack without access to this module must pass its level directly to `Registry.Command`/`Registry.TargetedCommand`.

---

## 4. `CommandRegistry`

The engine. Contains no commands itself. Can be `require`d by any other module to declare commands.

### Constants

| Field | Value | Description |
|---|---|---|
| `Registry.Prefix` | `":"` | Prefix for all commands. |
| `Registry.Target` | `"[player|me|all|others]"` | Standard usage text for a target. | ### Tools

#### `Registry.Notify(admin: Player, message: string)`
Sends `message` to the `admin`'s client via the `AdminCommandEvent` `RemoteEvent`. This is the only way to communicate a result or error to the admin.

#### `Registry.GetHumanoid(plr: Player) → Humanoid | nil`
Returns `nil` if `plr.Character` does not exist or does not have a `Humanoid`.

#### `Registry.GetHRP(plr: Player) → BasePart | nil`
Returns `nil` if `plr.Character` does not exist or does not have a `HumanoidRootPart`.

#### `Registry.FindPlayer(admin: Player, arg: string) → Player | nil`
Resolves a **single** player:
- `arg == "me"` (case-insensitive) → returns `admin`.
- Otherwise, searches by `Name` or `DisplayName` (exact match first, then partial prefix).
- Returns `nil` if `arg` is empty/nil or if there is no match.

#### `Registry.GetTargets(admin: Player, arg: string) → {Player}`
Resolves a **list** of players:
- `"me"` → `{admin}`
- `"all"` → all players on the server
- `"others"` → everyone except `admin`
- otherwise → searches by name like `FindPlayer`; returns a list containing a single element or an empty list.

#### `Registry.ForTargets(admin: Player, arg: string, fn: (plr: Player) → boolean?) → number | nil`
1. Resolves `arg` via `GetTargets`.
2. If empty → automatically sends `"SYSTEM Error: Player not found..."` to `admin` and returns `nil`.
3. Otherwise, calls `fn(plr)` for each target. If `fn` explicitly returns `false`, that player is **not** counted. 4. Returns the number of players for whom `fn` did not return `false`.

Used internally by `Registry.TargetedCommand`, but can be used directly for more specific needs.

#### `Registry.FindTool(name: string) → Tool | nil`
Searches for a `Tool` descendant named `name` (case-insensitive) in `ReplicatedStorage.Tools` and then `ServerStorage`.

#### `Registry.IsNumber2(args: {string}) → boolean`
Validation shortcut: `tonumber(args[2]) ~= nil`. To be passed as `check`.
