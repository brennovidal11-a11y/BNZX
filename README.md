--// =========================================================
--// BZNX FUGA DO TECLADO — V31.0 PRECISION ROUTE EDITOR EDITION
--// Tema: BZNX Dark Graphite & Neon Red
--// Features: Editor de Rota Cirúrgico (Remover Último / Remover por Índice Exato) +
--//           Auto Event + Auto-Buy + Aproximação 2.5s + Pulo/Coleta 1s + Retorno Ponto #1 + Holo 3D
--// =========================================================

-- [GUARD] Limpeza de instâncias anteriores
local env = (typeof(getgenv) == "function" and getgenv()) or _G
if type(env.BZNX_CLEANUP) == "function" then pcall(env.BZNX_CLEANUP) end

-- SERVIÇOS
local Players          = game:GetService("Players")
local RunService       = game:GetService("RunService")
local UserInputService = game:GetService("UserInputService")
local VirtualUser      = game:GetService("VirtualUser")
local HttpService      = game:GetService("HttpService")

local LocalPlayer = Players.LocalPlayer
local PlayerGui   = LocalPlayer:WaitForChild("PlayerGui")
local IS_MOBILE   = UserInputService.TouchEnabled and not UserInputService.KeyboardEnabled

-- MAID (Gerenciamento de Recursos)
local Maid = { Tasks = {} }
function Maid:Add(item) self.Tasks[#self.Tasks + 1] = item; return item end
function Maid:Clean()
    for _, item in ipairs(self.Tasks) do
        local kind = typeof(item)
        if kind == "RBXScriptConnection" then pcall(function() item:Disconnect() end)
        elseif kind == "Instance" then pcall(function() item:Destroy() end)
        elseif kind == "thread" then if item ~= coroutine.running() then pcall(task.cancel, item) end
        elseif kind == "function" then pcall(item) end
    end
    self.Tasks = {}
end

local function safeCall(fn, ...)
    if type(fn) ~= "function" then return false, "not a function" end
    return pcall(fn, ...)
end

-- CARREGAMENTO WINDUI
local WindUI
local function loadWindUI()
    local sources = {
        "https://raw.githubusercontent.com/Footagesus/WindUI/main/dist/main.lua",
    }
    for _, url in ipairs(sources) do
        local ok, result = safeCall(function() return loadstring(game:HttpGet(url))() end)
        if ok and result then return result end
        task.wait(0.3)
    end
    return nil
end

WindUI = loadWindUI()
if not WindUI then
    warn("[BZNX] Falha ao carregar WindUI.")
    return
end

-- PALETA & CONFIGURAÇÕES
local RED   = Color3.fromRGB(225, 35, 50)
local DARK  = Color3.fromRGB(20, 20, 25)

local Config = {
    AnchorHeight   = 2.6,
    MAX_TROPHY     = 1e18,

    RouteMode      = "TP",     -- "TP" ou "Walk"
    TPSpeed        = 2000,     -- Velocidade do Modo TP
    WalkRouteSpeed = 120,      -- Velocidade do Modo Caminhada
    
    HoloVisuals    = true,     -- Desenho 3D da Rota

    TrophyKeywords  = { "win", "wins", "victory", "troph", "trofeu", "troféu", "reward", "gain" },
    EventKeywords   = { "event", "cone", "drop", "token", "gift", "present", "coin", "item", "aura", "trail" },
    ExcludeKeywords = { "req", "need", "door", "gate", "wall", "shop", "store", "buy", "speed", "pass" },
    CustomBlacklist = {},
}

-- SUFIXOS & CONVERSOR
local SUFFIXES = {
    k = 1e3, m = 1e6, b = 1e9, t = 1e12, qa = 1e15, qi = 1e18,
    mil = 1e3, milhao = 1e6, bilhao = 1e9, trilhao = 1e12
}

local function stripRich(text) return (string.gsub(tostring(text), "<[^>]+>", "")) end

local function parseNumber(text)
    if type(text) ~= "string" then return nil end
    local clean = string.lower(stripRich(text))
    local num = string.match(clean, "([%+%-]?%d[%d%.,_%s]*)")
    if not num then return nil end

    num = string.gsub(num, "%s+", "")
    if string.find(num, "[,%.][0-9][0-9][0-9]") then
        num = string.gsub(num, "[,%.%_](%d%d%d)", "%1")
        num = string.gsub(num, "[,]", ".")
    else
        num = string.gsub(num, ",", ".")
    end

    local val = tonumber(num)
    if not val then return nil end

    for suff, mult in pairs(SUFFIXES) do
        if string.find(clean, "%f[%a]" .. suff .. "%f[%A]") or string.find(clean, "%d%s*" .. suff) then
            val = val * mult
            break
        end
    end
    return math.floor(val + 0.5)
end

local function hasKeyword(str, list)
    str = string.lower(tostring(str))
    for _, k in ipairs(list) do
        if string.find(str, string.lower(k), 1, true) then return true end
    end
    return false
end

-- ESTADO GLOBAL & STORE
local Store = {
    Values = { 
        AutoWin = false, ESP = true, SelectedPart = nil, 
        LockHitbox = false, Godmode = true, RouteActive = false,
        SmartTarget = true, AntiAFK = true, AutoEvent = false, AutoBuy = false
    },
    Subs = {},
}
function Store:Get(k) return self.Values[k] end
function Store:Set(k, v)
    if self.Values[k] == v then return end
    self.Values[k] = v
    if self.Subs[k] then
        for _, fn in ipairs(self.Subs[k]) do safeCall(fn, v) end
    end
end
function Store:Subscribe(k, fn)
    self.Subs[k] = self.Subs[k] or {}
    table.insert(self.Subs[k], fn)
end

local State = { 
    Destroyed = false, Humanoid = nil, RootPart = nil, LockedCFrame = nil,
    SessionStart = os.clock(), WinsCollected = 0
}
local Cache = {}
local PadList = {}
local PadIndex = {}
local Waypoints = {}

local function refreshCharacter()
    local char = LocalPlayer.Character
    if not char then return false end
    local hum  = char:FindFirstChildOfClass("Humanoid")
    local root = char:FindFirstChild("HumanoidRootPart")
    if not hum or not root or hum.Health <= 0 then return false end
    State.Humanoid = hum
    State.RootPart = root
    return true
end

-- ANTI-AFK INTELIGENTE
local antiAfkConn = nil
local function updateAntiAFK(enable)
    if enable then
        if antiAfkConn then antiAfkConn:Disconnect() end
        antiAfkConn = LocalPlayer.Idled:Connect(function()
            VirtualUser:Button2Down(Vector2.new(0, 0), workspace.CurrentCamera.CFrame)
            task.wait(1)
            VirtualUser:Button2Up(Vector2.new(0, 0), workspace.CurrentCamera.CFrame)
        end)
        Maid:Add(antiAfkConn)
    else
        if antiAfkConn then antiAfkConn:Disconnect(); antiAfkConn = nil end
    end
end
Store:Subscribe("AntiAFK", updateAntiAFK)

-- GODMODE BLINDADO
local godmodeConn = nil
local function updateGodmode(enable)
    if enable then
        if godmodeConn then godmodeConn:Disconnect() end
        godmodeConn = RunService.Stepped:Connect(function()
            if State.Destroyed then return end
            local char = LocalPlayer.Character
            if char then
                local hum = char:FindFirstChildOfClass("Humanoid")
                if hum then
                    hum.BreakJointsOnDeath = false
                    hum.MaxHealth = 1e9
                    hum.Health = 1e9
                    hum:SetStateEnabled(Enum.HumanoidStateType.Dead, false)
                end
            end
        end)
        Maid:Add(godmodeConn)
    else
        if godmodeConn then godmodeConn:Disconnect(); godmodeConn = nil end
    end
end
Store:Subscribe("Godmode", updateGodmode)

-- SCANNER DE PAD
local function resolvePart(inst)
    if inst:IsA("BasePart") then return inst end
    if inst:IsA("Model") then return inst.PrimaryPart or inst:FindFirstChildWhichIsA("BasePart", true) end
    return nil
end

local function evaluateInstance(inst)
    if not inst or not inst.Parent then return end
    local fullName = string.lower(inst:GetFullName())
    if hasKeyword(fullName, Config.ExcludeKeywords) or hasKeyword(fullName, Config.CustomBlacklist) then return end

    if inst:IsA("TextLabel") or inst:IsA("TextButton") then
        local text = inst.Text
        if text and text ~= "" then
            local clean = string.lower(stripRich(text))
            if string.find(clean, "%d") then
                if hasKeyword(clean, Config.TrophyKeywords) or string.find(text, "🏆") then
                    local val = parseNumber(clean)
                    if val and val > 0 and val <= Config.MAX_TROPHY then
                        local gui = inst:FindFirstAncestorWhichIsA("BillboardGui") or inst:FindFirstAncestorWhichIsA("SurfaceGui")
                        if gui then
                            local holder = gui.Adornee or gui.Parent
                            local part = resolvePart(holder)
                            if part then Cache[part] = val end
                        end
                    end
                end
            end
        end
    end

    if inst:IsA("BasePart") or inst:IsA("Model") then
        local part = resolvePart(inst)
        if not part or hasKeyword(part:GetFullName(), Config.CustomBlacklist) then return end

        local ok, attrs = safeCall(function() return inst:GetAttributes() end)
        if ok and attrs then
            for k, v in pairs(attrs) do
                if hasKeyword(k, Config.TrophyKeywords) then
                    local n = type(v) == "number" and v or parseNumber(tostring(v))
                    if n and n > 0 and n <= Config.MAX_TROPHY then Cache[part] = n end
                end
            end
        end
    end
end

local function runFullScan()
    table.clear(Cache)
    local ok, list = safeCall(workspace.GetDescendants, workspace)
    if ok and list then
        for i = 1, #list do
            if State.Destroyed then break end
            evaluateInstance(list[i])
            if i % 400 == 0 then task.wait() end
        end
    end
end

local function fmt(n)
    if n >= 1e15 then return string.format("%.2fQa", n / 1e15) end
    if n >= 1e12 then return string.format("%.2fT", n / 1e12) end
    if n >= 1e9  then return string.format("%.2fB", n / 1e9) end
    if n >= 1e6  then return string.format("%.2fM", n / 1e6) end
    if n >= 1e3  then return string.format("%.2fK", n / 1e3) end
    return tostring(n)
end

local TrophyDropdown

local function refreshPadList()
    runFullScan()
    local list = {}
    for part, val in pairs(Cache) do
        if part and part.Parent then
            list[#list + 1] = { Part = part, Value = val, Name = part.Name }
        end
    end

    table.sort(list, function(a, b) return a.Value > b.Value end)
    PadList = {}
    PadIndex = {}
    local dropdownOptions = {}

    for i, item in ipairs(list) do
        local label = string.format("#%d | +%s Wins • %s", i, fmt(item.Value), item.Name)
        PadList[#PadList + 1] = item
        PadIndex[label] = item.Part
        dropdownOptions[#dropdownOptions + 1] = label
    end

    if #dropdownOptions == 0 then dropdownOptions = { "Nenhum troféu encontrado" } end
    if TrophyDropdown then safeCall(function() TrophyDropdown:Refresh(dropdownOptions) end) end
    
    if #list > 0 and (Store:Get("SmartTarget") or not Store:Get("SelectedPart")) then 
        Store:Set("SelectedPart", list[1].Part) 
    end
end

-- VISUALIZADOR HOLOGRÁFICO 3D
local HoloFolder = Instance.new("Folder")
HoloFolder.Name = "BZNX_HoloFolder"
HoloFolder.Parent = workspace
Maid:Add(HoloFolder)

local function rebuildHoloRoute()
    HoloFolder:ClearAllChildren()
    if not Config.HoloVisuals or #Waypoints == 0 then return end

    for i, cf in ipairs(Waypoints) do
        local p = Instance.new("Part")
        p.Size = Vector3.new(1.2, 1.2, 1.2)
        p.Shape = Enum.PartType.Ball
        p.Color = RED
        p.Material = Enum.Material.Neon
        p.Anchored = true
        p.CanCollide = false
        p.CFrame = cf
        p.Parent = HoloFolder

        local nextCF = Waypoints[i + 1]
        if nextCF then
            local dist = (nextCF.Position - cf.Position).Magnitude
            local line = Instance.new("Part")
            line.Size = Vector3.new(0.2, 0.2, dist)
            line.Color = RED
            line.Material = Enum.Material.Neon
            line.Anchored = true
            line.CanCollide = false
            line.CFrame = CFrame.new(cf.Position, nextCF.Position) * CFrame.new(0, 0, -dist/2)
            line.Parent = HoloFolder
        end
    end
end

-- PERSISTÊNCIA LOCAL
local function saveRouteToFile()
    if writefile then
        local rawData = {}
        for _, cf in ipairs(Waypoints) do
            table.insert(rawData, {cf.X, cf.Y, cf.Z})
        end
        safeCall(writefile, "BZNX_Route.json", HttpService:JSONEncode(rawData))
        WindUI:Notify({ Title = "BZNX File System", Content = "Rota salva com sucesso!", Duration = 3 })
    end
end

local function loadRouteFromFile()
    if readfile and isfile and isfile("BZNX_Route.json") then
        local ok, data = safeCall(readfile, "BZNX_Route.json")
        if ok and data then
            local parsed = HttpService:JSONDecode(data)
            table.clear(Waypoints)
            for _, pos in ipairs(parsed) do
                table.insert(Waypoints, CFrame.new(pos[1], pos[2], pos[3]))
            end
            rebuildHoloRoute()
            WindUI:Notify({ Title = "BZNX File System", Content = #Waypoints .. " pontos carregados!", Duration = 3 })
        end
    end
end

-- ESP SYSTEM
local ESPFolder = Instance.new("Folder")
ESPFolder.Name = "BZNX_ESPFolder"
ESPFolder.Parent = workspace
Maid:Add(ESPFolder)

local function rebuildESP()
    ESPFolder:ClearAllChildren()
    if not Store:Get("ESP") then return end

    for _, item in ipairs(PadList) do
        if item.Part and item.Part.Parent then
            local bb = Instance.new("BillboardGui")
            bb.Size = UDim2.fromOffset(150, 26)
            bb.AlwaysOnTop = true
            bb.Adornee = item.Part
            bb.Parent = ESPFolder

            local t = Instance.new("TextLabel")
            t.Size = UDim2.fromScale(1, 1)
            t.BackgroundTransparency = 1
            t.Font = Enum.Font.GothamBold
            t.TextColor3 = Color3.fromRGB(255, 215, 0)
            t.TextStrokeTransparency = 0.2
            t.TextScaled = true
            t.Text = string.format("🏆 +%s", fmt(item.Value))
            t.Parent = bb
        end
    end
end
Store:Subscribe("ESP", rebuildESP)

-- TOUCH SYSTEM
local function triggerTouch(part)
    if not refreshCharacter() or not part then return end
    if typeof(firetouchinterest) == "function" then
        safeCall(firetouchinterest, State.RootPart, part, 0)
        task.wait(0.01)
        safeCall(firetouchinterest, State.RootPart, part, 1)
    end
end

-- MOTOR DE EVENTOS (AUTO CONE EVENT & DROPS)
task.spawn(function()
    while not State.Destroyed do
        if Store:Get("AutoEvent") and refreshCharacter() then
            pcall(function()
                for _, obj in ipairs(workspace:GetDescendants()) do
                    if obj:IsA("BasePart") then
                        local name = string.lower(obj.Name)
                        if hasKeyword(name, Config.EventKeywords) then
                            local oldCF = State.RootPart.CFrame
                            State.Humanoid.PlatformStand = true
                            State.RootPart.CFrame = obj.CFrame + Vector3.new(0, 2, 0)
                            triggerTouch(obj)
                            task.wait(0.1)
                            State.RootPart.CFrame = oldCF
                            State.Humanoid.PlatformStand = false
                            break
                        end
                    end
                end
            end)
        end
        task.wait(1.5)
    end
end)

-- MOTOR DE AUTO BUY (LOJAS / AURAS / TRILHAS)
task.spawn(function()
    while not State.Destroyed do
        if Store:Get("AutoBuy") then
            pcall(function()
                for _, prompt in ipairs(workspace:GetDescendants()) do
                    if prompt:IsA("ProximityPrompt") then
                        local parentName = string.lower(prompt.Parent and prompt.Parent.Name or "")
                        if hasKeyword(parentName, { "shop", "store", "buy", "aura", "trail", "item" }) then
                            fireproximityprompt(prompt)
                        end
                    end
                end
            end)
        end
        task.wait(3)
    end
end)

-- MOTOR DE ROTA PRINCIPAL
local isRouting = false
local function runRouteEngine()
    if isRouting then return end
    isRouting = true

    task.spawn(function()
        while Store:Get("RouteActive") and not State.Destroyed do
            if #Waypoints > 0 and refreshCharacter() then
                State.Humanoid.PlatformStand = true

                for _, targetCF in ipairs(Waypoints) do
                    if not Store:Get("RouteActive") or State.Destroyed then break end

                    local startCF = State.RootPart.CFrame
                    local dist = (targetCF.Position - startCF.Position).Magnitude
                    local currentSpeed = (Config.RouteMode == "TP") and Config.TPSpeed or Config.WalkRouteSpeed
                    local duration = math.max(0.01, dist / math.max(10, currentSpeed))
                    local elapsed = 0

                    while elapsed < duration do
                        if not Store:Get("RouteActive") or State.Destroyed then break end
                        local dt = task.wait()
                        elapsed = elapsed + dt
                        
                        currentSpeed = (Config.RouteMode == "TP") and Config.TPSpeed or Config.WalkRouteSpeed
                        duration = math.max(0.01, dist / math.max(10, currentSpeed))
                        
                        local alpha = math.min(1, elapsed / duration)
                        if refreshCharacter() then
                            State.RootPart.CFrame = startCF:Lerp(targetCF, alpha)
                            State.RootPart.AssemblyLinearVelocity = Vector3.zero
                        end
                    end
                end

                if Store:Get("RouteActive") and not State.Destroyed then
                    local pad = Store:Get("SelectedPart")
                    if pad and pad.Parent and refreshCharacter() then
                        local targetPadCF = pad.CFrame + Vector3.new(0, Config.AnchorHeight, 0)
                        local startPadCF = State.RootPart.CFrame
                        
                        local approachDuration = 2.5
                        local elapsedApproach = 0
                        while elapsedApproach < approachDuration do
                            if not Store:Get("RouteActive") or State.Destroyed then break end
                            local dt = task.wait()
                            elapsedApproach = elapsedApproach + dt
                            local alpha = math.min(1, elapsedApproach / approachDuration)
                            if refreshCharacter() then
                                State.RootPart.CFrame = startPadCF:Lerp(targetPadCF, alpha)
                                State.RootPart.AssemblyLinearVelocity = Vector3.zero
                            end
                        end

                        if not State.Destroyed and refreshCharacter() then
                            State.Humanoid.PlatformStand = false
                            State.Humanoid.Jump = true

                            local collectStart = os.clock()
                            while os.clock() - collectStart < 1.0 do
                                if not Store:Get("RouteActive") or State.Destroyed then break end
                                triggerTouch(pad)
                                task.wait(0.05)
                            end
                            State.WinsCollected = State.WinsCollected + 1
                        end

                        if not State.Destroyed and #Waypoints > 0 and refreshCharacter() then
                            State.Humanoid.PlatformStand = true
                            State.RootPart.CFrame = Waypoints[1]
                            State.RootPart.AssemblyLinearVelocity = Vector3.zero
                            task.wait(0.05)
                        end
                    end
                end
            else
                task.wait(0.5)
            end
        end

        if refreshCharacter() then State.Humanoid.PlatformStand = false end
        isRouting = false
    end)
end
Store:Subscribe("RouteActive", function(v) if v then runRouteEngine() end end)

-- HEARTBEAT
Maid:Add(RunService.Heartbeat:Connect(function()
    if State.Destroyed then return end

    if Store:Get("LockHitbox") and State.LockedCFrame and not Store:Get("RouteActive") and refreshCharacter() then
        State.RootPart.CFrame = State.LockedCFrame
        State.RootPart.AssemblyLinearVelocity = Vector3.zero
    end

    if Store:Get("AutoWin") and not Store:Get("RouteActive") then
        local pad = Store:Get("SelectedPart")
        if pad and pad.Parent and refreshCharacter() then
            if not Store:Get("LockHitbox") then
                State.Humanoid.PlatformStand = true
                State.RootPart.AssemblyLinearVelocity = Vector3.zero
                State.RootPart.CFrame = pad.CFrame + Vector3.new(0, Config.AnchorHeight, 0)
            end
            triggerTouch(pad)
        end
    end
end))

-- INTERFACE WINDUI
safeCall(function()
    WindUI:AddTheme({
        Name = "BZNX", Accent = RED, Dialog = DARK, Outline = RED,
        Text = Color3.fromRGB(240, 240, 245), Background = DARK, Button = Color3.fromRGB(38, 38, 46),
    })
end)

local Window = WindUI:CreateWindow({
    Title        = "BZNX FUGA V31.0",
    Icon         = "solar:keyboard-bold",
    Author       = "PZNX / Brenno",
    Folder       = "BZNX_FUGA",
    Size         = IS_MOBILE and UDim2.fromOffset(460, 320) or UDim2.fromOffset(700, 520),
    Transparent  = true,
    Theme        = "BZNX",
    OpenButton   = { Title = "B", Enabled = true, Draggable = true },
})

local function Tab(title, icon) return Window:Tab({ Title = title, Icon = icon, IconColor = RED }) end

-- ABA TROFÉUS & TARGETING
local TrophyTab = Tab("Troféus", "solar:cup-star-bold")
local TrophySec = TrophyTab:Section({ Title = "Seleção Inteligente" })

TrophyDropdown = TrophySec:Dropdown({
    Title    = "Selecione o Pad Desejado",
    Values   = { "Escaneando o mapa..." },
    Callback = function(selected)
        if type(selected) == "table" then selected = selected[1] end
        local part = PadIndex[selected]
        if part then
            Store:Set("SelectedPart", part)
            WindUI:Notify({ Title = "BZNX", Content = "Alvo trocado!", Duration = 2 })
        end
    end,
})

TrophySec:Toggle({
    Title    = "Smart Target (Auto Maior Multiplicador)",
    Value    = Store:Get("SmartTarget"),
    Callback = function(v) Store:Set("SmartTarget", v) end,
})

TrophySec:Button({
    Title    = "Teleportar para o Pad Selecionado",
    Callback = function()
        local pad = Store:Get("SelectedPart")
        if pad and refreshCharacter() then State.RootPart.CFrame = pad.CFrame + Vector3.new(0, 4, 0) end
    end,
})

TrophySec:Button({
    Title    = "Re-escanear o Mapa",
    Callback = function()
        refreshPadList()
        rebuildESP()
        WindUI:Notify({ Title = "BZNX", Content = "Scanner concluído!", Duration = 2 })
    end,
})

TrophySec:Toggle({
    Title    = "ESP de Troféus",
    Value    = Store:Get("ESP"),
    Callback = function(v) Store:Set("ESP", v) end,
})

-- ABA AUTOMAÇÃO E ROTA
local GameTab = Tab("Automação", "solar:gamepad-bold")

local EventSec = GameTab:Section({ Title = "Auto Farm de Eventos & Lojas" })

EventSec:Toggle({
    Title    = "Auto Cone Event (Coletar Eventos / Drops no Mapa)",
    Desc     = "Caça e coleta automaticamente itens, moedas e cones de eventos que aparecem no mapa.",
    Value    = Store:Get("AutoEvent"),
    Callback = function(v) Store:Set("AutoEvent", v) end,
})

EventSec:Toggle({
    Title    = "Auto-Buy Lojas (Auras & Trilhas)",
    Desc     = "Interage e compra automaticamente itens cosméticos das lojas disponíveis.",
    Value    = Store:Get("AutoBuy"),
    Callback = function(v) Store:Set("AutoBuy", v) end,
})

local RouteSec = GameTab:Section({ Title = "Configuração de Rota de Hitbox" })

RouteSec:Button({
    Title    = "Adicionar Ponto (Com Snap ao Chão)",
    Callback = function()
        if refreshCharacter() then
            local ray = workspace:Raycast(State.RootPart.Position, Vector3.new(0, -15, 0))
            local targetCF = ray and CFrame.new(ray.Position + Vector3.new(0, 2.5, 0)) or State.RootPart.CFrame
            
            table.insert(Waypoints, targetCF)
            rebuildHoloRoute()
            WindUI:Notify({ Title = "BZNX Rota", Content = "Ponto #" .. #Waypoints .. " adicionado!", Duration = 2 })
        end
    end,
})

RouteSec:Button({
    Title    = "Remover o Último Ponto",
    Callback = function()
        if #Waypoints > 0 then
            table.remove(Waypoints, #Waypoints)
            rebuildHoloRoute()
            WindUI:Notify({ Title = "BZNX Editor", Content = "Último ponto removido! Total restante: " .. #Waypoints, Duration = 2 })
        else
            WindUI:Notify({ Title = "BZNX Editor", Content = "A rota já está vazia!", Duration = 2 })
        end
    end,
})

RouteSec:Input({
    Title    = "Remover Ponto Específico (Digite o Número)",
    Placeholder = "Ex: 110",
    Callback = function(text)
        local index = tonumber(text)
        if index and index >= 1 and index <= #Waypoints then
            table.remove(Waypoints, index)
            rebuildHoloRoute()
            WindUI:Notify({ Title = "BZNX Editor", Content = "Ponto #" .. index .. " removido com sucesso!", Duration = 3 })
        else
            WindUI:Notify({ Title = "BZNX Editor", Content = "Número inválido ou fora do limite!", Duration = 3 })
        end
    end,
})

RouteSec:Button({
    Title    = "Limpar Pontos da Rota Inteira",
    Callback = function()
        table.clear(Waypoints)
        Store:Set("RouteActive", false)
        rebuildHoloRoute()
        WindUI:Notify({ Title = "BZNX Rota", Content = "Rota limpa!", Duration = 2 })
    end,
})

RouteSec:Dropdown({
    Title    = "Modo de Deslocamento",
    Values   = { "Modo TP (Teletransporte)", "Modo Percorrer (Caminhada)" },
    Value    = "Modo TP (Teletransporte)",
    Callback = function(v)
        if type(v) == "table" then v = v[1] end
        Config.RouteMode = string.find(v, "TP") and "TP" or "Walk"
    end,
})

RouteSec:Slider({
    Title    = "Velocidade Modo TP",
    Step     = 100,
    Value    = { Min = 200, Max = 10000, Default = Config.TPSpeed },
    Callback = function(v) Config.TPSpeed = v end,
})

RouteSec:Slider({
    Title    = "Velocidade Modo Caminhada",
    Step     = 10,
    Value    = { Min = 10, Max = 500, Default = Config.WalkRouteSpeed },
    Callback = function(v) Config.WalkRouteSpeed = v end,
})

RouteSec:Toggle({
    Title    = "Visualizador Holográfico 3D da Rota",
    Value    = Config.HoloVisuals,
    Callback = function(v) Config.HoloVisuals = v; rebuildHoloRoute() end,
})

RouteSec:Toggle({
    Title    = "Ativar Rota da Hitbox",
    Value    = Store:Get("RouteActive"),
    Callback = function(v)
        if v and #Waypoints == 0 then
            WindUI:Notify({ Title = "Erro", Content = "Adicione pontos primeiro!", Duration = 3 })
            Store:Set("RouteActive", false)
            return
        end
        Store:Set("RouteActive", v)
    end,
})

local FileSec = GameTab:Section({ Title = "Arquivos & Persistência Local" })

FileSec:Button({
    Title    = "Salvar Rota Atual em Arquivo",
    Callback = saveRouteToFile,
})

FileSec:Button({
    Title    = "Carregar Rota Salva do Arquivo",
    Callback = loadRouteFromFile,
})

local SafetySec = GameTab:Section({ Title = "Proteção & AFK" })

SafetySec:Toggle({
    Title    = "Anti-AFK Inteligente",
    Value    = Store:Get("AntiAFK"),
    Callback = function(v) Store:Set("AntiAFK", v) end,
})

SafetySec:Toggle({
    Title    = "Godmode Blindado (Anti-Marshmallow)",
    Value    = Store:Get("Godmode"),
    Callback = function(v) Store:Set("Godmode", v) end,
})

local LockSec = GameTab:Section({ Title = "Controle Estático" })

LockSec:Toggle({
    Title    = "Travar Hitbox Posição Atual",
    Value    = Store:Get("LockHitbox"),
    Callback = function(v)
        if v and refreshCharacter() then State.LockedCFrame = State.RootPart.CFrame end
        Store:Set("LockHitbox", v)
    end,
})

LockSec:Toggle({
    Title    = "Auto Win Simples",
    Value    = Store:Get("AutoWin"),
    Callback = function(v) Store:Set("AutoWin", v) end,
})

-- DESTROY & CLEANUP
local function Destroy()
    State.Destroyed = true
    Store:Set("AutoWin", false)
    Store:Set("Godmode", false)
    Store:Set("LockHitbox", false)
    Store:Set("RouteActive", false)
    Store:Set("AutoEvent", false)
    Store:Set("AutoBuy", false)
    table.clear(Waypoints)
    HoloFolder:ClearAllChildren()
    Maid:Clean()
    safeCall(function() Window:Destroy() end)
    env.BZNX_CLEANUP = nil
end

env.BZNX_CLEANUP = Destroy

-- INICIALIZAÇÃO
task.defer(function()
    refreshPadList()
    updateGodmode(Store:Get("Godmode"))
    rebuildESP()
end)
