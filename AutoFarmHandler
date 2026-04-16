-- AutoFarmHandler.lua
-- Handles Fruit Summoning Macro and Auto Pickup
-- OPTIMIZED: Uses caching to prevent FPS drops
-- FIXED: "invalid key to next" crash by snapshotting table keys during pickup loop

local Players = game:GetService("Players")
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local RunService = game:GetService("RunService")
local Workspace = game:GetService("Workspace")
local HttpService = game:GetService("HttpService")
local GuiService = game:GetService("GuiService")
local VirtualInputManager = (function()
    local ok, vim = pcall(function() return Instance.new("VirtualInputManager") end)
    if ok then print("[AutoFarm] ✅ Using instanced VirtualInputManager") return vim end
    local ok2, vim2 = pcall(function() return game:GetService("VirtualInputManager") end)
    if ok2 then print("[AutoFarm] ✅ Using service VirtualInputManager") return vim2 end
    warn("[AutoFarm] ⚠️ VirtualInputManager unavailable!")
    return nil
end)()

-- ============================================================================
-- VIRTUAL MOUSE SYSTEM (shared with CoreManager pattern)
-- Uses VIM + GuiInset to properly simulate mouse input.
-- ============================================================================
local VirtualMouse = {}

function VirtualMouse.applyInset(x, y)
    local inset = GuiService:GetGuiInset()
    return x + inset.X, y + inset.Y
end

function VirtualMouse.move(x, y)
    if not VirtualInputManager then return false end
    local rx, ry = VirtualMouse.applyInset(x, y)
    VirtualInputManager:SendMouseMoveEvent(rx, ry, game)
    return true
end

function VirtualMouse.click(x, y, button)
    if not VirtualInputManager then return false end
    button = button or 0
    local rx, ry = VirtualMouse.applyInset(x, y)
    VirtualInputManager:SendMouseButtonEvent(rx, ry, button, true, game, 0)
    task.wait()
    VirtualInputManager:SendMouseButtonEvent(rx, ry, button, false, game, 0)
    return true
end

function VirtualMouse.clickAt(x, y, button)
    if not VirtualInputManager then return false end
    VirtualMouse.move(x, y)
    task.wait(0.15)
    VirtualMouse.click(x, y, button)
    return true
end

local LocalPlayer = Players.LocalPlayer
local PlayerGui = LocalPlayer:WaitForChild("PlayerGui")

local AutoFarmHandler = {}
AutoFarmHandler.__index = AutoFarmHandler

AutoFarmHandler.Settings = {
    AutoPickup = false,
    AutoSummon = false,
    PickupRange = 50,
    PickupDelay = 0.3,
    MouseMoveTime = 0.3, -- Time in seconds to smoothly move mouse to fruit
    ServerHopOnDanger = true, -- Master switch
    ServerHopOnChakraSense = true, -- Specific switch for Chakra Sense
    DangerRadius = 300
}

local GAME_ID = 10266164381
local HOP_MARKER_FILE = "PuppyHub_HopMarker.txt"

local State = {
    PickupCache = {},
    Connections = {},
    SummonThread = nil,
    PickupThread = nil,
    SafetyThread = nil,
    IsDefending = false,
    IsHopping = false,
} 

local function writeHopMarker(context)
    if not writefile then return false end

    local payload = tostring(tick()) .. "|" .. tostring(game.PlaceId) .. "|" .. tostring(game.JobId) .. "|" .. tostring(context or "unknown")
    for _ = 1, 3 do
        local ok = pcall(function()
            writefile(HOP_MARKER_FILE, payload)
        end)
        if ok and isfile and isfile(HOP_MARKER_FILE) then
            return true
        end
        task.wait(0.05)
    end

    warn("[AutoFarm] Failed to verify hop marker write for " .. tostring(context))
    return false
end

-- ============================================================================
-- EXPOSED FUNCTIONS (For CoreManager)
-- ============================================================================

-- Function to Instantly Sever All Connections/Threads
function AutoFarmHandler.emergencyStop()
    print("[AutoFarm] 🚨 EMERGENCY STOP: Severing all connections/threads...")
    
    -- 1. Cancel Tasks
    if State.SummonThread then task.cancel(State.SummonThread) State.SummonThread = nil end
    if State.PickupThread then task.cancel(State.PickupThread) State.PickupThread = nil end
    if State.SafetyThread then task.cancel(State.SafetyThread) State.SafetyThread = nil end
    
    -- 2. Disconnect Events
    for name, conn in pairs(State.Connections) do
        if conn and conn.Connected then 
            conn:Disconnect() 
        end
    end
    State.Connections = {}
    
    -- 3. Reset State Flags
    State.IsDefending = false
    State.IsHopping = false
    -- We do NOT clear Settings here, so we can resume later if needed
    -- But we clear cache to be safe
    State.PickupCache = {}
    
    print("[AutoFarm] All systems halted.")
end

function AutoFarmHandler.checkSafety()
    -- Returns: isSafe (bool), reason (string/nil)
    local myChar = LocalPlayer.Character
    local myRoot = myChar and myChar:FindFirstChild("HumanoidRootPart")
    if not myRoot then return true, nil end -- Can't check if not spawned
    local settingsFolder = ReplicatedStorage:FindFirstChild("Settings")

    for _, player in ipairs(Players:GetPlayers()) do
        if player ~= LocalPlayer then
            -- 1. Distance (Always checked if ServerHopOnDanger is master-enabled externally, 
            -- but effectively checking here implies danger)
            if player.Character and player.Character:FindFirstChild("HumanoidRootPart") then
                local dist = (player.Character.HumanoidRootPart.Position - myRoot.Position).Magnitude
                if dist <= AutoFarmHandler.Settings.DangerRadius then
                    return false, "Player Nearby (" .. player.Name .. ")"
                end
            end
            
            -- 2. Chakra Sense (Conditional Check)
            if AutoFarmHandler.Settings.ServerHopOnChakraSense then
                -- Replicated Settings Check
                if settingsFolder then
                    local pSettings = settingsFolder:FindFirstChild(player.Name)
                    if pSettings then
                        local sVal = pSettings:FindFirstChild("CurrentSkill")
                        if sVal and type(sVal.Value) == "string" and string.find(string.lower(sVal.Value), "sense") then
                            return false, "Chakra Sense (" .. player.Name .. ")"
                        end
                    end
                end
                
                -- Tool Check
                if player.Character then
                    for _, tool in ipairs(player.Character:GetChildren()) do
                        if tool:IsA("Tool") and string.find(string.lower(tool.Name), "sense") then
                            return false, "Chakra Sense Tool (" .. player.Name .. ")"
                        end
                    end
                end

                local backpack = player:FindFirstChildOfClass("Backpack")
                if backpack then
                    for _, tool in ipairs(backpack:GetChildren()) do
                        if tool:IsA("Tool") and string.find(string.lower(tool.Name), "sense") then
                            return false, "Chakra Sense Tool (" .. player.Name .. ")"
                        end
                    end
                end
            end
        end
    end
    return true, nil
end

-- ============================================================================
-- HTTP SERVER HOP
-- ============================================================================
local ServerHop = {}

function ServerHop.join(serverId)
    local events = ReplicatedStorage:FindFirstChild("Events")
    local dataEvent = events and events:FindFirstChild("DataEvent")
    if dataEvent then
        print("[AutoFarm] Firing ServerTeleport to: " .. tostring(serverId))
        writeHopMarker("AutoFarm.ServerHop.join")
        -- Standard Bloodlines teleport implementation
        dataEvent:FireServer("ServerTeleport", serverId, 14)
    else
        warn("[AutoFarm] DataEvent not found! Hop failed.") 
    end
end

function ServerHop.fetchServers()
    local allServers = {}
    local url = string.format("https://roproxy-production-0200.up.railway.app/games/v1/games/%d/servers/0?sortOrder=2&excludeFullGames=false&limit=100", GAME_ID)
    
    print("[AutoFarm-Debug] Fetching Server List from: " .. url)
    
    local success, response = pcall(function()
        local requestFunc = (syn and syn.request) or (http and http.request) or http_request or (fluxus and fluxus.request) or request
        if requestFunc then return requestFunc({Url = url, Method = "GET"}).Body else return game:HttpGet(url) end
    end)

    if success and response then
        local jsonSuccess, data = pcall(HttpService.JSONDecode, HttpService, response)
        if jsonSuccess and data and data.data then
            print("[AutoFarm-Debug] Servers found: " .. #data.data)
            for _, s in ipairs(data.data) do table.insert(allServers, s) end
        else
            warn("[AutoFarm-Debug] JSON Decode Failed or No Data:", response)
        end
    else
        warn("[AutoFarm-Debug] HTTP Request Failed.")
    end
    return allServers
end

function ServerHop.execute()
    if State.IsHopping then return end
    State.IsHopping = true
    print("[AutoFarm] Initiating Hop...")

    task.spawn(function()
        local servers = ServerHop.fetchServers()
        if #servers > 0 then
            local candidates = {}
            for _, s in ipairs(servers) do
                if s.playing and s.maxPlayers and s.id ~= game.JobId then
                    if s.playing >= 5 and s.playing < (s.maxPlayers - 2) then
                        table.insert(candidates, s)
                    end
                end
            end
            
            if #candidates > 0 then
                local target = candidates[math.random(1, #candidates)]
                print("[AutoFarm] Hopping to server: " .. target.id .. " (" .. target.playing .. " plrs)")
                ServerHop.join(target.id)
            else
                print("[AutoFarm] No suitable servers found.")
            end
        else
            warn("[AutoFarm] No servers returned from fetch.")
        end
        task.wait(2)
        State.IsHopping = false
    end)
end

AutoFarmHandler.executeServerHop = ServerHop.execute

-- ============================================================================
-- CLAY DOLL DEFENSE LOGIC
-- ============================================================================

local function getDataFunctionRemote()
    local events = ReplicatedStorage:FindFirstChild("Events")
    return events and events:FindFirstChild("DataFunction")
end

local function handleClayDoll(dollModel)
    if State.IsDefending then return end
    State.IsDefending = true
    
    print("[AutoFarm] ⚠️ CLAY DOLL DETECTED! Engaging Defense Protocol...")

    local character = LocalPlayer.Character
    local root = character and character:FindFirstChild("HumanoidRootPart")
    
    if not root then 
        State.IsDefending = false 
        return 
    end

    local originalAnchored = root.Anchored
    root.Anchored = true
    
    -- Start blocking via VirtualInputManager
    local VM = game:GetService('VirtualInputManager')
    pcall(function() VM:SendKeyEvent(true, Enum.KeyCode.F, false, game) end)
    print("[AutoFarm] 🛡️ Block activated via VirtualInputManager.")

    -- Face the doll while blocking
    local defenseLoop = RunService.Heartbeat:Connect(function()
        if not dollModel or not dollModel.Parent then return end
        local dollRoot = dollModel:FindFirstChild("HumanoidRootPart") or dollModel.PrimaryPart
        if dollRoot and root then
            local lookPos = Vector3.new(dollRoot.Position.X, root.Position.Y, dollRoot.Position.Z)
            root.CFrame = CFrame.lookAt(root.Position, lookPos)
        end
    end)

    -- Wait for doll to die or disappear
    while dollModel and dollModel.Parent do
        local hum = dollModel:FindFirstChild("Humanoid")
        if hum and hum.Health <= 0 then break end
        task.wait(0.1)
    end

    if defenseLoop then defenseLoop:Disconnect() end
    if root then root.Anchored = originalAnchored end

    -- Unblock via VirtualInputManager
    pcall(function() VM:SendKeyEvent(false, Enum.KeyCode.F, false, game) end)
    print("[AutoFarm] 🛡️ Unblocked via VirtualInputManager.")

    print("[AutoFarm] ✅ Clay Doll neutralized. Resuming farm.")
    State.IsDefending = false
end

local function startClayDollWatcher()
    if State.Connections.ClayDoll then return end
    State.Connections.ClayDoll = Workspace.ChildAdded:Connect(function(child)
        task.delay(0.5, function()
            if not child.Parent then return end
            if child:IsA("Model") and child:FindFirstChild("Humanoid") then
                if Players:GetPlayerFromCharacter(child) then return end
                local myRoot = LocalPlayer.Character and LocalPlayer.Character:FindFirstChild("HumanoidRootPart")
                if myRoot then
                    local targetRoot = child:FindFirstChild("HumanoidRootPart") or child.PrimaryPart
                    if targetRoot then
                        local dist = (myRoot.Position - targetRoot.Position).Magnitude
                        if dist < 30 then handleClayDoll(child) end
                    end
                end
            end
        end)
    end)
end

local function stopClayDollWatcher()
    if State.Connections.ClayDoll then
        State.Connections.ClayDoll:Disconnect()
        State.Connections.ClayDoll = nil
    end
end

-- ============================================================================
-- SUMMON MACRO (Hardcoded 16s Loop)
-- Tools in this game are virtual — no Tool instances exist.
-- We equip via DataEvent:FireServer("Item", "Selected", name) then click to use.
-- ============================================================================

local HOTBAR_MAX_SLOTS = 11
local FRUIT_SUMMON_TOOL_CANDIDATES = {
    "Fruit Summoning",
    "Fruit Summon",
    "Summon Fruit",
    "Fruit Summon Tool",
}

local HOTBAR_KEYBINDS = {
    [1] = Enum.KeyCode.One,
    [2] = Enum.KeyCode.Two,
    [3] = Enum.KeyCode.Three,
    [4] = Enum.KeyCode.Four,
    [5] = Enum.KeyCode.Five,
    [6] = Enum.KeyCode.Six,
    [7] = Enum.KeyCode.Seven,
    [8] = Enum.KeyCode.Eight,
    [9] = Enum.KeyCode.Nine,
    [10] = Enum.KeyCode.Zero,
    [11] = Enum.KeyCode.Minus,
}

local function getDataEvent()
    local events = ReplicatedStorage:FindFirstChild("Events")
    return events and events:FindFirstChild("DataEvent")
end

local function getLoadout()
    local clientGui = PlayerGui and PlayerGui:FindFirstChild("ClientGui")
    local mainframe = clientGui and clientGui:FindFirstChild("Mainframe")
    return mainframe and mainframe:FindFirstChild("Loadout")
end

local function normalizeName(name)
    if type(name) ~= "string" then return "" end
    return string.lower((name:gsub("%s+", " "):gsub("^%s+", ""):gsub("%s+$", "")))
end

local function isFruitSummonName(name)
    local normalized = normalizeName(name)
    if normalized == "" then return false end

    for _, candidate in ipairs(FRUIT_SUMMON_TOOL_CANDIDATES) do
        if normalized == normalizeName(candidate) then
            return true
        end
    end

    return normalized:find("fruit", 1, true) and normalized:find("summon", 1, true)
end

local function findFruitSummonSlot()
    local loadout = getLoadout()
    if not loadout then return nil end

    for slotIndex = 1, HOTBAR_MAX_SLOTS do
        local slot = loadout:FindFirstChild("Slot" .. slotIndex)
        local slotText = slot and slot:FindFirstChild("SlotText")
        local itemName = slotText and slotText:IsA("TextLabel") and slotText.Text or ""
        if isFruitSummonName(itemName) then
            return {
                index = slotIndex,
                slot = slot,
                itemName = itemName,
            }
        end
    end
end

local function activateHotbarSlot(slotInfo)
    if not slotInfo or not slotInfo.slot then return false end

    local success = false

    local keyCode = HOTBAR_KEYBINDS[slotInfo.index]
    if keyCode and VirtualInputManager then
        pcall(function()
            VirtualInputManager:SendKeyEvent(true, keyCode, false, game)
            task.wait(0.05)
            VirtualInputManager:SendKeyEvent(false, keyCode, false, game)
            success = true
        end)
    end

    local function trySignal(instance, signalName)
        if success or not instance or not firesignal then return end
        local signal = instance[signalName]
        if typeof(signal) ~= "RBXScriptSignal" then return end

        local ok = pcall(firesignal, signal)
        if ok then
            success = true
        end
    end

    trySignal(slotInfo.slot, "MouseButton1Click")
    trySignal(slotInfo.slot, "Activated")

    for _, descendant in ipairs(slotInfo.slot:GetDescendants()) do
        if descendant:IsA("GuiButton") then
            trySignal(descendant, "MouseButton1Click")
            trySignal(descendant, "Activated")
            if success then break end
        end
    end

    if success then
        task.wait(0.15)
    end

    return success
end

local function selectFruitSummonTool(toolName)
    local dataEvent = getDataEvent()
    if not dataEvent then
        warn("[AutoFarm] DataEvent not found! Cannot equip tool.")
        return false
    end
    
    local ok, err = pcall(function()
        dataEvent:FireServer("Item", "Selected", toolName)
    end)
    
    if not ok then
        warn("[AutoFarm] Failed to equip fruit summon item (" .. tostring(toolName) .. "): " .. tostring(err))
        return false
    end
    
    print("[AutoFarm] Equipped: " .. tostring(toolName))
    return true
end

local function resolveFruitSummonTool()
    local slotInfo = findFruitSummonSlot()
    if slotInfo then
        return slotInfo.itemName, slotInfo
    end

    return FRUIT_SUMMON_TOOL_CANDIDATES[1], nil
end

local function clickEquippedTool()
    pcall(function()
        local x, y = 400, 400
        
        if VirtualInputManager then
            -- Right Click (Mastered Version) with inset
            VirtualMouse.click(x, y, 1)
            task.wait(0.1)
            -- Left Click (Standard Version) with inset
            VirtualMouse.click(x, y, 0)
        else
            -- Fallback: physical mouse clicks
            if mouse2click then mouse2click() end
            task.wait(0.1)
            if mouse1click then mouse1click() end
        end
    end)
end

local function useFruitSummonTool()
    local toolName, slotInfo = resolveFruitSummonTool()
    if not toolName then
        warn("[AutoFarm] Fruit summon item was not found in the hotbar.")
        return false
    end

    local activatedHotbar = activateHotbarSlot(slotInfo)
    if not selectFruitSummonTool(toolName) then return false end

    task.wait(activatedHotbar and 0.15 or 0.3)
    clickEquippedTool()
    
    return true
end

local function summonLoop()
    while AutoFarmHandler.Settings.AutoSummon do
        if State.IsDefending or State.IsHopping then 
            task.wait(0.5) 
            continue 
        end

        local success = useFruitSummonTool()
        
        if success then
            print("[AutoFarm] Summon used. Waiting 16s cooldown...")
            for i = 1, 160 do
                if not AutoFarmHandler.Settings.AutoSummon then return end
                task.wait(0.1)
            end
            print("[AutoFarm] Cooldown finished.")
        else
            warn("[AutoFarm] Summon failed, retrying in 2s...")
            task.wait(2)
        end
    end
end

-- ============================================================================
-- AUTO PICKUP LOGIC
-- Fruits spawn as MeshPart in workspace.
-- Inside has a StringValue with .Name == LocalPlayer.Name (ownership check)
-- Inside has a ClickDetector called "ItemDetector"
-- ============================================================================

local function isValidFruitPickup(obj)
    -- Must be a MeshPart (fruit) or BasePart
    if not obj:IsA("BasePart") then return false end
    
    -- Must have an ItemDetector ClickDetector
    local detector = obj:FindFirstChild("ItemDetector")
    if not detector or not detector:IsA("ClickDetector") then return false end
    
    -- Must have a StringValue with our player name (ownership check)
    local playerName = LocalPlayer.Name
    for _, child in ipairs(obj:GetChildren()) do
        if child:IsA("StringValue") and child.Name == playerName then
            return true
        end
    end
    
    return false
end

local function addToCache(child)
    if isValidFruitPickup(child) then
        State.PickupCache[child] = true
        print("[AutoFarm] 🍎 Fruit detected & cached: " .. child.Name)
    end
end

local function startTracking()
    -- Scan existing workspace children
    task.spawn(function()
        for _, child in ipairs(Workspace:GetChildren()) do
            addToCache(child)
        end
    end)
    
    -- Watch for new children (fruits spawn as MeshPart)
    State.Connections.Tracking = Workspace.ChildAdded:Connect(function(child)
        -- Small delay to let the StringValue and ClickDetector replicate
        task.delay(0.5, function()
            if child.Parent then
                addToCache(child)
            end
        end)
    end)
    
    State.Connections.Removing = Workspace.ChildRemoved:Connect(function(child)
        if State.PickupCache[child] then
            State.PickupCache[child] = nil
        end
    end)
end

local function stopTracking()
    if State.Connections.Tracking then State.Connections.Tracking:Disconnect() end
    if State.Connections.Removing then State.Connections.Removing:Disconnect() end
    State.Connections.Tracking = nil
    State.Connections.Removing = nil
    State.PickupCache = {}
end

-- ============================================================================
-- FRUIT PICKUP METHODS (Priority: VIM > firesignal > fireclickdetector > physical)
-- ============================================================================

-- Method 1: VirtualMouse move + click at fruit screen position (fully virtual)
local function tryVIMPickup(itemPos)
    if not VirtualInputManager then return false end
    local camera = Workspace.CurrentCamera
    if not camera then return false end
    
    local screenPos, onScreen = camera:WorldToScreenPoint(itemPos)
    if not onScreen then return false end
    
    -- Move cursor to fruit, then right-click (mastered) + left-click (standard)
    VirtualMouse.move(screenPos.X, screenPos.Y)
    task.wait(0.1)
    VirtualMouse.click(screenPos.X, screenPos.Y, 1) -- right-click
    task.wait(0.05)
    VirtualMouse.click(screenPos.X, screenPos.Y, 0) -- left-click
    return true
end

-- Method 2: firesignal with full hover sequence
local function tryFireSignalPickup(clickDetector)
    if not firesignal then return false end
    
    local ok = pcall(function()
        firesignal(clickDetector.MouseHoverEnter, LocalPlayer)
        firesignal(clickDetector.MouseHoverLeave, LocalPlayer)
        task.wait(0.15)
        firesignal(clickDetector.MouseClick, LocalPlayer)
    end)
    
    return ok
end

-- Method 3: fireclickdetector
local function tryFireClickDetector(clickDetector)
    if not fireclickdetector then return false end
    
    local ok = pcall(function()
        fireclickdetector(clickDetector)
    end)
    
    return ok
end

-- Method 4: Physical mouse fallback (last resort, needs window focused)
local function tryPhysicalClick(itemPos)
    if not (mouse1click and mousemoverel) then return false end
    local camera = Workspace.CurrentCamera
    if not camera then return false end
    
    local screenPos, onScreen = camera:WorldToScreenPoint(itemPos)
    if not onScreen then return false end
    
    local mouse = LocalPlayer:GetMouse()
    mousemoverel(screenPos.X - mouse.X, screenPos.Y - mouse.Y)
    task.wait(0.05)
    mouse1click()
    return true
end

-- Master pickup function: tries each method in priority order
local function pickupFruit(item)
    -- Priority 1: VirtualMouse with inset (fully virtual, works in background)
    if tryVIMPickup(item.Position) then
        return true, "VIM"
    end
    
    local detector = item:FindFirstChild("ItemDetector")
    if detector and detector:IsA("ClickDetector") then
        -- Priority 2: firesignal with hover sequence
        if tryFireSignalPickup(detector) then
            return true, "firesignal"
        end
        
        -- Priority 3: fireclickdetector
        if tryFireClickDetector(detector) then
            return true, "fireclickdetector"
        end
    end
    
    -- Priority 4: Physical mouse (needs window focused)
    if tryPhysicalClick(item.Position) then
        return true, "physical"
    end
    
    return false, "none"
end

local function doPickup()
    if State.IsDefending or State.IsHopping then return end
    local character = LocalPlayer.Character
    local root = character and character:FindFirstChild("HumanoidRootPart")
    if not root then return end
    local range = AutoFarmHandler.Settings.PickupRange
    
    -- Snapshot keys first to avoid "invalid key to next" during yields
    local items = {}
    for item, _ in pairs(State.PickupCache) do
        table.insert(items, item)
    end
    
    for _, item in ipairs(items) do
        if not AutoFarmHandler.Settings.AutoPickup then break end
        
        -- Double check item validity (it might have been removed during a previous wait)
        if State.PickupCache[item] and item.Parent then
            local itemPos = item.Position -- We know it's a BasePart
            
            if itemPos and (root.Position - itemPos).Magnitude <= range then
                local picked, method = pickupFruit(item)
                if picked then
                    print("[AutoFarm] ✅ Picked up fruit: " .. item.Name .. " (via " .. method .. ")")
                else
                    print("[AutoFarm] ⚠️ All pickup methods failed for: " .. item.Name)
                end
                
                task.wait(AutoFarmHandler.Settings.PickupDelay)
            end
        elseif not item.Parent then
             -- Cleanup if parent missing
             State.PickupCache[item] = nil
        end
    end
end

function AutoFarmHandler.togglePickup(val)
    AutoFarmHandler.Settings.AutoPickup = val
    if State.PickupThread then task.cancel(State.PickupThread) State.PickupThread = nil end
    if val then
        startTracking()
        State.PickupThread = task.spawn(function()
            while AutoFarmHandler.Settings.AutoPickup do doPickup() task.wait(0.3) end
            stopTracking()
        end)
    else stopTracking() end
end

function AutoFarmHandler.toggleSummon(val)
    AutoFarmHandler.Settings.AutoSummon = val
    if State.SummonThread then task.cancel(State.SummonThread) State.SummonThread = nil end
    if val then
        -- Force AutoPickup ON
        AutoFarmHandler.togglePickup(true)
        
        -- Start safety watcher
        if State.SafetyThread then task.cancel(State.SafetyThread) end
        State.SafetyThread = task.spawn(function()
            while AutoFarmHandler.Settings.AutoSummon do
                local safe, _ = AutoFarmHandler.checkSafety()
                if not safe and AutoFarmHandler.Settings.ServerHopOnDanger then
                    AutoFarmHandler.executeServerHop()
                    break
                end
                task.wait(1)
            end
        end)
        
        startClayDollWatcher()
        State.SummonThread = task.spawn(summonLoop)
    else
        if State.SafetyThread then task.cancel(State.SafetyThread) State.SafetyThread = nil end
        stopClayDollWatcher()
    end
end

return AutoFarmHandler
