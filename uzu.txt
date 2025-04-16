getgenv().config = {}

local players = game:GetService("Players")
local http_service = game:GetService("HttpService")
local replicated_storage = game:GetService("ReplicatedStorage")
local teleport_service = game:GetService("TeleportService")
local player = players.LocalPlayer
local data_remote_event = replicated_storage.BridgeNet2.dataRemoteEvent

local ctrl_char = {"\a", "\b", "\f", "\n", "\r", "\t", "\v", "\z", "\0", "\1", "\2", "\3", "\4", "\5", "\6", "\7", "\8", "\9"}

local folder = "Uzu"
local name = ("%s - %s Arise Dungeon.lua"):format(player.UserId, game.GameId)
local path = ("%s/%s"):format(folder, name)

repeat task.wait() until player:GetAttribute("Loaded") and workspace.__Extra:FindFirstChild("__Spawns")

function save()
    if not isfolder(folder) then makefolder(folder) end
    writefile(path, http_service:JSONEncode(config))
end

function load()
    if not isfile(path) then return end
    getgenv().config = http_service:JSONDecode(readfile(path))
end

function teleport(position)
    local character = player.Character
    if not character then return end

    character:SetAttribute("InTp", true)
    character:PivotTo(position)
end

function get_distance(position)
    return player.Character and player:DistanceFromCharacter(position) or math.huge
end

function float()
    local character = player.Character
    local root_part = character and character:FindFirstChild("HumanoidRootPart")
    if not root_part then return end
    root_part.Velocity = Vector3.zero
end

function get_nearest_mob()
    local dist = math.huge
    local target = nil

    for _, v in workspace.__Main.__Enemies.Server:GetDescendants() do
        local mag = v:IsA("Part") and not v:GetAttribute("Dead") and get_distance(v:GetPivot().p)

        if mag and mag < dist then
            dist = mag
            target = v
        end
    end
    return target
end

function get_weapons()
    local weapons = {}

    for _, v in player.leaderstats.Inventory.Weapons:GetChildren() do
        local weapon_name = v:GetAttribute("Name")
        local weapon_rank = v:GetAttribute("Level")

        if v:GetAttribute("Locked") then continue end
        if player.leaderstats.Equips:GetAttribute("Weapon") == v.Name then continue end
        if weapon_name == "DualCrystalSword" then continue end

        weapons[weapon_name] = weapons[weapon_name] or {}
        weapons[weapon_name][weapon_rank] = weapons[weapon_name][weapon_rank] or {}
        table.insert(weapons[weapon_name][weapon_rank], v.Name)
    end
    return weapons
end

function replay_dungeon()
    local ticket = player.leaderstats.Inventory.Items:FindFirstChild("Ticket")
    local ticket_amount = ticket and ticket:GetAttribute("Amount")
    getgenv().old_ticket = old_ticket or ticket_amount or 0

    if old_ticket ~= ticket_amount or not replicated_storage:GetAttribute("Dungeon") then
        for _, v in ctrl_char do
            data_remote_event:FireServer({{Type = "Gems", Event = "DungeonAction", Action = "BuyTicket"}, v})
            data_remote_event:FireServer({{Event = "DungeonAction", Action = "Create"}, v})
            data_remote_event:FireServer({{Dungeon = player.UserId, Event = "DungeonAction", Action = "Start"}, v})
        end
        task.wait(60)
    end
end

function auto_dungeon()
    while task.wait() and config.auto_dungeon do
        replay_dungeon()

        local mob = get_nearest_mob()
        if not mob then continue end

        float()

        if get_distance(mob:GetPivot().p) > 10 then
            teleport(mob:GetPivot() * CFrame.new(0, 2, 0.1))
            task.wait(0.3)
        end

        data_remote_event:FireServer({{Event = "PunchAttack", Enemy = mob.Name}, "\4"})
    end
end

function auto_upgrade_weapon()
    while task.wait() and config.auto_upgrade_weapon do
        local weapon_table = get_weapons()

        for weapon, v in weapon_table do
            for rank, v2 in v do
                if #v2 < 3 then continue end
                if rank == 7 then continue end
                data_remote_event:FireServer({{
                    Type=weapon,
                    BuyType="Gems",
                    Weapons={v2[1], v2[2], v2[3]},
                    Event="UpgradeWeapon",
                    Level=rank + 1
                }, "\n"})
                task.wait()
            end
        end
    end
end

function auto_farm()
    while task.wait() and config.auto_farm do
        local mob = get_nearest_mob()
        if not mob then continue end

        float()

        if get_distance(mob:GetPivot().p) > 10 then
            teleport(mob:GetPivot() * CFrame.new(0, 2, 0.1))
            task.wait(0.3)
        end

        data_remote_event:FireServer({{Event = "PunchAttack", Enemy = mob.Name}, "\5"})
        task.wait(config.auto_farm_speed or 0.2)
    end
end

function auto_rejoin()
    player.OnTeleport:Connect(function(State)
        if State == Enum.TeleportState.Failed or State == Enum.TeleportState.InProgress then
            teleport_service:Teleport(game.PlaceId, player)
        end
    end)

    game:GetService("CoreGui").RobloxPromptGui.promptOverlay.ChildAdded:Connect(function(child)
        if child.Name == "ErrorPrompt" then
            wait(2)
            teleport_service:Teleport(game.PlaceId, player)
        end
    end)
end

task.wait(.1)
load()

if config.auto_rejoin then
    auto_rejoin()
end

local library = loadstring(game:HttpGet("https://raw.githubusercontent.com/uzu01/public/main/ui/uwuware"))()
local window = library:CreateWindow("Arise Crossover | Uzu")

local main_folder = window:AddFolder("Main")
local misc_folder = window:AddFolder("Misc")

main_folder:AddToggle({text = "Auto Dungeon", state = config.auto_dungeon, callback = function(v)
    config.auto_dungeon = v
    save()
    task.spawn(auto_dungeon)
end})

main_folder:AddToggle({text = "Auto Farm", state = config.auto_farm, callback = function(v)
    config.auto_farm = v
    save()
    task.spawn(auto_farm)
end})

main_folder:AddSlider({
    text = "Farm Speed (secs)",
    min = 0,
    max = 2,
    value = config.auto_farm_speed or 0.2,
    callback = function(v)
        config.auto_farm_speed = v
        save()
    end
})

misc_folder:AddToggle({text = "Auto Upgrade Weapon", state = config.auto_upgrade_weapon, callback = function(v)
    config.auto_upgrade_weapon = v
    save()
    task.spawn(auto_upgrade_weapon)
end})

misc_folder:AddToggle({text = "Auto Execute", state = config.auto_execute, callback = function(v)
    config.auto_execute = v
    save()
    if not v then return end
    queue_on_teleport('loadstring(game:HttpGet("https://raw.githubusercontent.com/uzu01/uzu/refs/heads/main/arise.lua"))()')
end})

misc_folder:AddToggle({text = "Auto Rejoin", state = config.auto_rejoin, callback = function(v)
    config.auto_rejoin = v
    save()
    if v then auto_rejoin() end
end})

misc_folder:AddBind({text = "Toggle GUI", key = "LeftControl", callback = function()
    library:Close()
end})

library:Init()
