<div align="center">

# Repcl

<img src="https://img.shields.io/badge/Repcl-v0.1.0-6C3EF4?style=for-the-badge&labelColor=0d0d0f" />
<img src="https://img.shields.io/badge/Roblox_Execution_and_Processing_Command_Library-6C3EF4?style=for-the-badge&labelColor=0d0d0f" />
<img src="https://img.shields.io/badge/Plinko_Labs-Built_by-6C3EF4?style=for-the-badge&labelColor=0d0d0f" />
<img src="https://img.shields.io/badge/Luau-strict-6C3EF4?style=for-the-badge&labelColor=0d0d0f" />
<img src="https://img.shields.io/badge/Iris-WWI_&_Repl-6C3EF4?style=for-the-badge&labelColor=0d0d0f" />

</div>

---

<br />

Repcl is a full-featured in-game command framework for Roblox. It handles command registration, typed argument parsing, server/client execution contexts, auth hooks, middleware, a chain expression language, and a built-in script editor called **Repl**. The main CLI terminal is a fully custom Roblox ScreenGui. WWI windows and the Repl editor are powered by [Iris](https://github.com/SirMallard/Iris).

---

## Compared to alternatives

| Feature | Repcl | Cmdr | BasicAdmin |
|---|---|---|---|
| Typed arg system | ✓ | ✓ | ✗ |
| Custom types + autocomplete | ✓ | ✓ | ✗ |
| Multi-value picker (Players) | ✓ | ✗ | ✗ |
| Server / Client / Bridge context | ✓ | ✗ | ✗ |
| Chain expressions (`& && \|\|`) | ✓ | ✗ | ✗ |
| Stack refs (`_ __ ___`) | ✓ | ✗ | ✗ |
| Conditionals (`~ ! = != > < >= <=`) | ✓ | ✗ | ✗ |
| Math in arg position | ✓ | ✗ | ✗ |
| Wrapped Window Interface (WWI) | ✓ | ✗ | ✗ |
| Built-in script editor (Repl) | ✓ | ✗ | ✗ |
| Iris-based UI | ✓ | ✗ | ✗ |
| Middleware hooks | ✓ | ✓ | ✗ |
| Rate limiting + cooldowns | ✓ | ✗ | ✗ |
| Fluent builder API | ✓ | ✗ | ✗ |

---

## Install

Place `Repcl` (ModuleScript) under `ReplicatedStorage`. 

---

## Getting Started 

**Server** (`Script` in `ServerScriptService `):

```lua
local Repcl = require(game.ReplicatedStorage.Repcl.Repcl)

Repcl.SetRoleHook(function(player)
    if player.UserId == YOUR_USER_ID then return "Owner" end
    if player:GetRankInGroup(GROUP_ID) >= 100 then return "Admin" end
    return "User"
end)

Repcl.Start({
    DefaultRole = "User",
    ReplEnabled = true,
})

Repcl.Register({
    Name        = "kick",
    Aliases     = { "k" },
    Description = "Kick a player",
    Context     = "Server",
    Permission  = "Admin",
    Args = {
        { Name = "target", Types = { "Player" }, Required = true },
        { Name = "reason", Types = { "String" }, Required = false, Default = "no reason" },
    },
    Attributes = {
        { Name = "silent", Short = "s", Description = "suppress broadcast" },
    },
    Run = function(ctx)
        ctx.Args.target:Kick(ctx.Args.reason)
        if not ctx.Attributes.silent then
            Repcl.Print(ctx.Args.target.Name .. " was kicked -- " .. ctx.Args.reason)
        end
    end,
})
```

**Client** (`LocalScript` in `StarterPlayerScripts`):

```lua
local Repcl = require(game.ReplicatedStorage.Repcl.Repcl)

game:GetService("UserInputService").InputBegan:Connect(function(input, gpe)
    if gpe then return end
    if input.KeyCode == Enum.KeyCode.Semicolon then
        Repcl.ToggleCLI()
    end
end)
```

The CLI auto-focuses input on open. Type `help` to see all registered commands.

---

## Command definition

### Table style

```lua
Repcl.Register({
    Name        = "ban",
    Aliases     = { "b" },
    Description = "Ban a player",
    Context     = "Bridge",
    Permission  = "Admin",

    Args = {
        { Name = "target",   Types = { "Player" },   Required = true },
        { Name = "duration", Types = { "Duration" },  Required = false, Default = 86400 },
        { Name = "reason",   Types = { "String" },    Required = false, Default = "no reason" },
    },

    Attributes = {
        { Name = "silent",    Short = "s", Description = "suppress broadcast" },
        { Name = "permanent", Short = "p", Description = "permanent ban" },
    },

    Client = function(ctx)
        ctx.Commit({
            TargetId  = ctx.Args.target.UserId,
            Duration  = ctx.Attributes.permanent and -1 or ctx.Args.duration,
            Reason    = ctx.Args.reason,
        })
    end,

    Server = function(ctx, payload)
        local target = game.Players:GetPlayerByUserId(payload.TargetId)
        if target then target:Kick("banned -- " .. payload.Reason) end
        ctx.Reply(tostring(payload.TargetId) .. " banned")
    end,
})
```

### Builder style

```lua
Repcl.Command("ban")
    :Alias("b")
    :Description("Ban a player")
    :Context("Bridge")
    :Permission("Admin")
    :Arg({ Name = "target",   Types = { "Player" }, Required = true })
    :Arg({ Name = "duration", Types = { "Duration" }, Required = false, Default = 86400 })
    :Attribute({ Name = "silent",    Short = "s" })
    :Attribute({ Name = "permanent", Short = "p" })
    :Client(function(ctx)
        ctx.Commit({ TargetId = ctx.Args.target.UserId })
    end)
    :Server(function(ctx, payload)
        local t = game.Players:GetPlayerByUserId(payload.TargetId)
        if t then t:Kick("banned") end
    end)
    :Register()
```

---

## Execution contexts

| Context | Behaviour |
|---|---|
| `"Client"` | Runs locally on client, never touches server |
| `"Server"` | Client sends invocation to server via RemoteEvent, server executes |
| `"Shared"` | Registered on both, runs wherever it is called from |
| `"Bridge"` | Client runs `Client` handler first, calls `ctx.Commit(payload)` to hand off to `Server` handler |

Bridge is for commands where the client does confirmation/UI work and the server does the authoritative action. Auth is checked on server before `Server` fires.

---

## Args

Each arg is a table inside the `Args` array:

```lua
{
    Name     = "target",
    Types    = { "Player", "String" },  -- union -- tries each in order
    Required = true,
    Default  = nil,
    Validate = function(value)
        if value == nil then return "player not found" end
        return true
    end,
}
```

`Types` is a union -- Repcl tries each type left to right and uses the first that parses and validates. If none pass, the command is rejected before `Run` fires. The dev never writes type checking inside handlers.

### Built-in types

`String` `Number` `Integer` `Boolean` `Player` `Players` `Team` `Color` `Vector3`

### Custom types

```lua
Repcl.DefineType("Duration", {
    Parse = function(raw)
        local n, unit = raw:match("^(%d+)([smhd])$")
        if not n then return nil end
        local mult = { s = 1, m = 60, h = 3600, d = 86400 }
        return tonumber(n) * (mult[unit] or 1)
    end,
    Validate = function(value)
        if value == nil then return "invalid duration -- use 30s 5m 2h 1d" end
        return true
    end,
    Autocomplete = function(raw)
        return { "30s", "5m", "1h", "6h", "1d", "7d" }
    end,
    Display = function(value)
        return tostring(value) .. "s"
    end,
    Multi      = false,
    Delimiter  = nil,
    Searchable = false,
})
```

Types with `Multi = true` trigger a floating picker frame in the CLI instead of inline ghost text. `Searchable = true` adds a filter input to that picker.

---

## Attributes

Flags attached after args with `-short` or `-name`:

```
kick PlayerTwo "grief" -s
ban PlayerTwo 1d "grief" -s -p
```

Declared on the command:

```lua
Attributes = {
    { Name = "silent",    Short = "s", Description = "suppress broadcast" },
    { Name = "permanent", Short = "p", Description = "permanent ban" },
}
```

Accessed in the handler via `ctx.Attributes.silent` -- always a boolean.

---

## Chain expression language

Multiple commands can be chained and made conditional in a single input.

### Chain operators

| Operator | Behaviour |
|---|---|
| `&` | Run next regardless of result |
| `&&` | Abort chain if current fails |
| `\|\|` | Run next only if current failed |

```
give PlayerTwo coins 500 && kick PlayerTwo "cleaned up"
```

### Object stack

Commands that return a value push onto a stack. Subsequent commands reference it:

| Ref | Meaning |
|---|---|
| `_` | Last returned object |
| `__` | Second to last |
| `___` | Third to last |
| `_.Property.Nested` | Property access on stack object |

```
GetPlayer coolwolf204 & GetItem sword
-- _ = sword, __ = player
give __ _.ItemId
```

If a link fails and was supposed to push, it does not pollute the stack. The next successful push takes its slot.

### Conditionals

```
GetPlayer coolwolf204 ~ ban _.UserId "cheating" -p
```

| Operator | Meaning |
|---|---|
| `~` | not nil -- continue if value exists |
| `!` | is nil -- continue if value is absent |
| `=` | equals |
| `!=` | not equals |
| `>` `<` `>=` `<=` | numeric comparison |

### Math in arg position

```
give PlayerTwo coins _.Cash + 500
heal PlayerTwo _.MaxHealth * 0.5
```

Math expressions referencing `_` are resolved before the arg is parsed and passed to the type system.

### Full example

```
GetPlayer coolwolf204 ~ && _.Health > 0 && ban _.UserId "cheating" -p & GetItem sword ~ give __ _.ItemId
```

---

## Auth

```lua
Repcl.SetAuthHook(function(player, commandName)
    return player:GetRankInGroup(GROUP_ID) >= 100
end)

Repcl.SetRoleHook(function(player)
    if player.UserId == OWNER_ID then return "Owner" end
    if player:GetRankInGroup(GROUP_ID) >= 200 then return "Admin" end
    return "User"
end)

Repcl.SetGroupHook(function(player)
    return player:GetRankInGroup(GROUP_ID)
end)

Repcl.AllowPlayer(USERID)
Repcl.DenyPlayer(USERID)
Repcl.SetDefaultRole("User")
```

Auth is always checked server-side before any server or bridge command executes. Client-only commands are not auth-gated by the framework -- restrict those via your own role checks inside the handler.

---

## Middleware

```lua
Repcl.PreExecute(function(ctx)
    if ctx.Player.AccountAge < 7 then
        ctx.Abort("account too new")
        return false
    end
end)

Repcl.PostExecute(function(ctx, result)
    if not ctx.Attributes.silent then
        Repcl.Print("[" .. ctx.Role .. "] " .. ctx.Player.Name .. " ran /" .. ctx.Command)
    end
end)

Repcl.OnErr(function(ctx, err)
    Repcl.Err(ctx.Command .. " failed -- " .. err)
end)

Repcl.OnConnect(function(player)
    Repcl.PrintTo(player, "welcome " .. player.Name)
end)
```

`PreExecute` returning `false` cancels the command. `ctx.Abort(msg)` cancels mid-handler.

---

## `ctx` reference

| Field | Type | Description |
|---|---|---|
| `ctx.Player` | `Player` | Who ran the command |
| `ctx.Command` | `string` | Command name |
| `ctx.Args` | `table` | Parsed and validated args |
| `ctx.Attributes` | `table` | Attribute flags (boolean) |
| `ctx.RawInput` | `string` | Original unparsed string |
| `ctx.IsServer` | `boolean` | |
| `ctx.IsClient` | `boolean` | |
| `ctx.ExecutedAt` | `number` | `os.clock()` at dispatch |
| `ctx.Role` | `string` | From `SetRoleHook` |
| `ctx.Rank` | `number` | From `SetGroupHook` |
| `ctx.Print(msg)` | | Write to caller's console |
| `ctx.Warn(msg)` | | |
| `ctx.Err(msg)` | | |
| `ctx.Reply(msg)` | | Alias for Print |
| `ctx.Abort(msg)` | | Cancel with message |
| `ctx.Commit(payload)` | | Bridge only -- hand off to server |

---

## Wrapped Window Interface (WWI)

Commands can optionally declare an Iris window. Opening the window dispatches through the same auth and `Run` pipeline as the CLI.

```lua
Repcl.Window("kick", function(win)
    win.Input("target", { "Player" })
    win.Input("reason", { "String" })
    win.Separator()
    win.Checkbox("silent", "Suppress broadcast")
    win.Button("Kick", function(values)
        -- dispatches through auth + Run
    end)
    win.Output()
end)

-- open programmatically
Repcl.ShowWindow("kick")
```

### Standalone windows (no auth)

```lua
Repcl.Window("dev_panel", function(win)
    win.Text("Server Stats")
    win.Separator()
    win.Label("Players", tostring(#game.Players:GetPlayers()))
    win.Slider("fov", 50, 120, 70)
    win.ColorPicker("ambient", "Ambient Light")
    win.Button("Apply", function(values)
        game.Workspace.CurrentCamera.FieldOfView = values.fov
    end)
end)

Repcl.ShowWindow("dev_panel")
```

If the window ID matches a registered command, auth is checked on button press. If the ID has no matching command, no auth check occurs.

### `WindowController` API

```
win.Input(name, types)
win.Dropdown(name, options)
win.Checkbox(name, label)
win.Slider(name, min, max, default)
win.ColorPicker(name, label)
win.MultiInput(name, types)
win.Text(content)
win.Label(key, value)
win.Separator()
win.Output()
win.Table(headers, rows)
win.Tree(label, fn)
win.Indent(fn)
win.Button(label, fn)
win.Tooltip(text)
win.Get(name)
win.Set(name, value)
win.OnChange(name, fn)
```

---

## Repl

Repl (Remediate Execution Processing Layer) is the built-in script editor. It is Repcl's nano -- an Iris window where you write and run multi-line Luau scripts against the live game state.

Open it from the CLI:

```
repl
repl -s     -- open in server context
```

Or programmatically:

```lua
Repcl.OpenRepl()
Repcl.SetReplContext("Server")

Repcl.OnReplRun(function(source, result)
    print("Repl executed:", source)
end)
```

Repl runs in a sandboxed environment on client by default. Switch to server context with `-s` or `Repcl.SetReplContext("Server")`.

---

## Rate limiting and cooldowns

```lua
Repcl.Start({
    RateLimit = { MaxCalls = 10, Window = 5 },
})

Repcl.SetCooldown("kick", 3)
Repcl.SetCooldown("ban", 10)
```

Rate limits are enforced per-player server-side. Cooldowns are per-command per-player.

---

## UI controls

```lua
Repcl.ShowCLI()
Repcl.HideCLI()
Repcl.ToggleCLI()
Repcl.LockCLI()
Repcl.UnlockCLI()
Repcl.SetTheme({})

Repcl.ShowWindow("id")
Repcl.HideWindow("id")

Repcl.OpenRepl()
Repcl.CloseRepl()
Repcl.SetReplContext("Server" | "Client")
```

---

## Output

```lua
Repcl.Print(msg)
Repcl.PrintTo(player, msg)
Repcl.Warn(msg)
Repcl.Err(msg)
Repcl.Clear()
Repcl.ClearFor(player)
Repcl.Log(tag, ctx)
```

---

## `Repcl.Parser`

The expression parser is exposed for power users who want to extend the chain language.

```lua
Repcl.Parser.DefineToken(symbol, config)
Repcl.Parser.DefineResolver(name, fn)

local ast = Repcl.Parser.Parse("kick PlayerTwo -s")
Repcl.Parser.Execute(ast, runFn)
Repcl.Parser.Tokenize(raw)
Repcl.Parser.ResolveStackRef(ref, stack)
Repcl.Parser.EvalMath(expr, stack)
```

Most developers will never touch this. It exists for adding custom operators or integrating Repcl's parser into other systems.

---

## Full API surface

```
Repcl.Start(config)
Repcl.Stop()
Repcl.Reload()
Repcl.IsStarted()

Repcl.Register(def)
Repcl.RegisterAll(defs)
Repcl.Unregister(name)
Repcl.UnregisterAll()
Repcl.Override(name, def)
Repcl.Enable(name)
Repcl.Disable(name)
Repcl.Alias(name, alias)
Repcl.GetCommand(name)
Repcl.GetAll()
Repcl.Command(name)  -- builder

Repcl.DefineType(name, def)
Repcl.OverrideType(name, def)
Repcl.GetType(name)

Repcl.SetAuthHook(fn)
Repcl.SetRoleHook(fn)
Repcl.SetGroupHook(fn)
Repcl.AllowPlayer(userId)
Repcl.DenyPlayer(userId)
Repcl.SetDefaultRole(role)
Repcl.GetRole(player)

Repcl.PreExecute(fn)
Repcl.PostExecute(fn)
Repcl.OnErr(fn)
Repcl.OnRegister(fn)
Repcl.OnUnregister(fn)
Repcl.OnConnect(fn)
Repcl.OnDisconnect(fn)

Repcl.SetCooldown(name, seconds)
Repcl.SetRateLimit(config)

Repcl.Print(msg)
Repcl.PrintTo(player, msg)
Repcl.Warn(msg)
Repcl.Err(msg)
Repcl.Clear()
Repcl.ClearFor(player)
Repcl.Log(tag, ctx)

Repcl.ShowCLI()
Repcl.HideCLI()
Repcl.ToggleCLI()
Repcl.LockCLI()
Repcl.UnlockCLI()
Repcl.SetTheme(theme)

Repcl.Window(id, fn)
Repcl.ShowWindow(id)
Repcl.HideWindow(id)

Repcl.OpenRepl()
Repcl.CloseRepl()
Repcl.SetReplContext(ctx)
Repcl.OnReplRun(fn)

Repcl.Parser
Repcl.Version
```

---

<div align="center">

Built by Plinko Labs

</div>
