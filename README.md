# BNZX
BNZ_PAINEL
--// =========================================================
--// BZNX FPS — TRAINING / NPC AIM ASSIST
--// PZNX / Brenno
--// WindUI + Luau
--//
--// Destinado a NPCs/dummies do próprio jogo.
--// =========================================================

local WindUI = loadstring(game:HttpGet(
    "https://raw.githubusercontent.com/Footagesus/WindUI/main/dist/main.lua"
))()

local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local Workspace = game:GetService("Workspace")

local LocalPlayer = Players.LocalPlayer
local PlayerGui = LocalPlayer:WaitForChild("PlayerGui")

-- =========================================================
-- CORES
-- =========================================================

local RED = Color3.fromRGB(225, 35, 50)
local WHITE = Color3.fromRGB(245, 245, 245)
local GREY = Color3.fromRGB(155, 155, 165)
local BLUE = Color3.fromRGB(45, 145, 255)
local GREEN = Color3.fromRGB(45, 230, 145)
local YELLOW = Color3.fromRGB(255, 220, 45)

-- =========================================================
-- CONFIGURAÇÃO
-- =========================================================

local Config = {
    Aim = {
        Enabled = false,
        FOV = 15,
        Range = 100,
        Smoothness = 10,
        TargetPart = "Head",
        RequireLineOfSight = true,
    },

    Visual = {
        FOVEnabled = false,
        FOVRadius = 150,
        NPCHighlight = false,
        NPCName = false,
        NPCDistance = false,
    },

    General = {
        NPCFolderName = "TrainingNPCs",
    }
}

-- =========================================================
-- ESTADO
-- =========================================================

local CurrentTarget = nil
local FOVGui = nil
local FOVCircle = nil
local NPCVisuals = {}

-- =========================================================
-- NOTIFICAÇÃO
-- =========================================================

local function Notify(title, text)
    WindUI:Notify({
        Title = title,
        Content = text,
        Duration = 2,
        Icon = "solar:check-circle-bold",
    })
end

-- =========================================================
-- WINDOW
-- =========================================================

local Window = WindUI:CreateWindow({
    Title = "BZNX FPS",
    Icon = "solar:target-bold",
    Author = "PZNX / Brenno",
    Folder = "BZNX_FPS_TRAINING",

    Size = UDim2.fromOffset(700, 500),

    Transparent = true,
    Theme = "Dark",
    SideBarWidth = 180,

    OpenButton = {
        Title = "B",
        CornerRadius = UDim.new(0, 10),
        StrokeThickness = 2,
        Enabled = true,
        Draggable = true,
    },
})

-- =========================================================
-- UTILITÁRIOS
-- =========================================================

local function GetCharacter(model)
    if not model then
        return nil
    end

    if model:IsA("Model") then
        return model
    end

    return model:FindFirstAncestorOfClass("Model")
end

local function GetRoot(model)
    local character = GetCharacter(model)

    if not character then
        return nil
    end

    return character:FindFirstChild("HumanoidRootPart")
        or character.PrimaryPart
end

local function GetTargetPart(model)
    local character = GetCharacter(model)

    if not character then
        return nil
    end

    return character:FindFirstChild(Config.Aim.TargetPart, true)
        or character:FindFirstChild("Head", true)
        or GetRoot(character)
end

local function IsAliveNPC(model)
    if not model or not model:IsA("Model") then
        return false
    end

    local humanoid = model:FindFirstChildOfClass("Humanoid")

    if humanoid and humanoid.Health <= 0 then
        return false
    end

    return GetRoot(model) ~= nil
end

-- =========================================================
-- LOCALIZAÇÃO DOS NPCs
-- =========================================================

local function GetNPCContainer()
    return Workspace:FindFirstChild(
        Config.General.NPCFolderName
    )
end

local function GetNPCs()
    local container = GetNPCContainer()

    if not container then
        return {}
    end

    local result = {}

    for _, object in ipairs(container:GetChildren()) do
        if IsAliveNPC(object) then
            table.insert(result, object)
        end
    end

    return result
end

-- =========================================================
-- VISIBILIDADE
-- =========================================================

local function HasLineOfSight(targetPart)
    if not Config.Aim.RequireLineOfSight then
        return true
    end

    local camera = Workspace.CurrentCamera

    if not camera or not targetPart then
        return false
    end

    local origin = camera.CFrame.Position
    local direction = targetPart.Position - origin

    local params = RaycastParams.new()
    params.FilterType = Enum.RaycastFilterType.Exclude
    params.FilterDescendantsInstances = {
        LocalPlayer.Character
    }

    local result = Workspace:Raycast(
        origin,
        direction,
        params
    )

    if not result then
        return true
    end

    return result.Instance:IsDescendantOf(
        targetPart.Parent
    )
end

-- =========================================================
-- FOV VISUAL
-- =========================================================

local function DestroyFOV()
    if FOVGui then
        FOVGui:Destroy()
    end

    FOVGui = nil
    FOVCircle = nil
end

local function CreateFOV()
    DestroyFOV()

    if not Config.Visual.FOVEnabled then
        return
    end

    FOVGui = Instance.new("ScreenGui")
    FOVGui.Name = "BZNX_TrainingFOV"
    FOVGui.IgnoreGuiInset = true
    FOVGui.ResetOnSpawn = false
    FOVGui.DisplayOrder = 50
    FOVGui.Parent = PlayerGui

    FOVCircle = Instance.new("Frame")
    FOVCircle.Name = "Circle"

    FOVCircle.AnchorPoint = Vector2.new(0.5, 0.5)
    FOVCircle.Position = UDim2.fromScale(0.5, 0.5)

    FOVCircle.Size = UDim2.fromOffset(
        Config.Visual.FOVRadius * 2,
        Config.Visual.FOVRadius * 2
    )

    FOVCircle.BackgroundTransparency = 1
    FOVCircle.Parent = FOVGui

    local corner = Instance.new("UICorner")
    corner.CornerRadius = UDim.new(1, 0)
    corner.Parent = FOVCircle

    local stroke = Instance.new("UIStroke")
    stroke.Color = RED
    stroke.Thickness = 2
    stroke.Transparency = 0.1
    stroke.Parent = FOVCircle
end

local function UpdateFOV()
    if not FOVCircle then
        return
    end

    FOVCircle.Size = UDim2.fromOffset(
        Config.Visual.FOVRadius * 2,
        Config.Visual.FOVRadius * 2
    )
end

-- =========================================================
-- SELEÇÃO DO NPC
-- =========================================================

local function FindBestNPC()
    local camera = Workspace.CurrentCamera

    if not camera then
        return nil
    end

    local bestPart = nil
    local smallestAngle = math.huge

    local cameraPosition = camera.CFrame.Position
    local cameraDirection = camera.CFrame.LookVector

    for _, npc in ipairs(GetNPCs()) do
        local targetPart = GetTargetPart(npc)

        if targetPart then
            local offset =
                targetPart.Position - cameraPosition

            local distance = offset.Magnitude

            if distance > 0
                and distance <= Config.Aim.Range
            then
                local direction = offset.Unit

                local dot =
                    math.clamp(
                        cameraDirection:Dot(direction),
                        -1,
                        1
                    )

                local angle =
                    math.deg(math.acos(dot))

                if angle <= Config.Aim.FOV / 2
                    and angle < smallestAngle
                    and HasLineOfSight(targetPart)
                then
                    smallestAngle = angle
                    bestPart = targetPart
                end
            end
        end
    end

    return bestPart
end

-- =========================================================
-- MIRA SUAVE
-- =========================================================

local function AimAtTarget(targetPart, deltaTime)
    local camera = Workspace.CurrentCamera

    if not camera or not targetPart then
        return
    end

    local direction =
        targetPart.Position - camera.CFrame.Position

    if direction.Magnitude <= 0.001 then
        return
    end

    local desired =
        CFrame.lookAt(
            camera.CFrame.Position,
            targetPart.Position
        )

    local alpha =
        1 - math.exp(
            -Config.Aim.Smoothness * deltaTime
        )

    camera.CFrame =
        camera.CFrame:Lerp(
            desired,
            alpha
        )
end

-- =========================================================
-- VISUAIS DOS NPCs
-- =========================================================

local function RemoveNPCVisual(npc)
    local visual = NPCVisuals[npc]

    if visual then
        visual:Destroy()
        NPCVisuals[npc] = nil
    end
end

local function CreateNPCVisual(npc)
    RemoveNPCVisual(npc)

    if not npc then
        return
    end

    local root = GetRoot(npc)

    if not root then
        return
    end

    local folder = Instance.new("Folder")
    folder.Name = "BZNX_NPC_VISUAL"
    folder.Parent = npc

    NPCVisuals[npc] = folder

    if Config.Visual.NPCHighlight then
        local highlight = Instance.new("Highlight")

        highlight.Name = "Highlight"
        highlight.Adornee = npc

        highlight.FillColor = BLUE
        highlight.OutlineColor = WHITE

        highlight.FillTransparency = 0.85
        highlight.OutlineTransparency = 0

        highlight.Parent = folder
    end

    if Config.Visual.NPCName
        or Config.Visual.NPCDistance
    then
        local billboard = Instance.new("BillboardGui")

        billboard.Name = "Info"
        billboard.Adornee = root

        billboard.Size = UDim2.fromOffset(220, 40)
        billboard.StudsOffset = Vector3.new(0, 3, 0)

        billboard.AlwaysOnTop = true
        billboard.MaxDistance = 1000

        billboard.Parent = folder

        local label = Instance.new("TextLabel")

        label.Size = UDim2.fromScale(1, 1)
        label.BackgroundTransparency = 1

        label.TextColor3 = WHITE
        label.TextStrokeTransparency = 0.3

        label.Font = Enum.Font.GothamBold
        label.TextSize = 12

        label.Parent = billboard

        task.spawn(function()
            while billboard.Parent do
                local text = ""

                if Config.Visual.NPCName then
                    text = npc.Name
                end

                if Config.Visual.NPCDistance then
                    local myCharacter =
                        LocalPlayer.Character

                    local myRoot =
                        myCharacter
                        and myCharacter:FindFirstChild(
                            "HumanoidRootPart"
                        )

                    local targetRoot = GetRoot(npc)

                    if myRoot and targetRoot then
                        local distance =
                            math.floor(
                                (
                                    myRoot.Position
                                    - targetRoot.Position
                                ).Magnitude
                            )

                        if text ~= "" then
                            text = text .. " • "
                        end

                        text =
                            text
                            .. distance
                            .. " studs"
                    end
                end

                label.Text = text

                task.wait(0.1)
            end
        end)
    end
end

local function RefreshNPCVisuals()
    for npc in pairs(NPCVisuals) do
        RemoveNPCVisual(npc)
    end

    for _, npc in ipairs(GetNPCs()) do
        CreateNPCVisual(npc)
    end
end

local function ClearNPCVisuals()
    for npc in pairs(NPCVisuals) do
        RemoveNPCVisual(npc)
    end
end

-- =========================================================
-- INÍCIO
-- =========================================================

local HomeTab = Window:Tab({
    Title = "Início",
    Icon = "solar:home-2-bold",
    IconColor = RED,
})

local HomeSection = HomeTab:Section({
    Title = "BZNX FPS",
    Desc = "Centro de treinamento e debug.",
})

HomeSection:Paragraph({
    Title = "Jogador",
    Desc =
        LocalPlayer.DisplayName
        .. "  @"
        .. LocalPlayer.Name,
})

HomeSection:Paragraph({
    Title = "NPCs",
    Desc =
        tostring(#GetNPCs())
        .. " NPCs encontrados",
})

HomeSection:Paragraph({
    Title = "Status",
    Desc = "Painel carregado.",
})

-- =========================================================
-- AIM ASSIST
-- =========================================================

local AimTab = Window:Tab({
    Title = "Mira",
    Icon = "solar:target-bold",
    IconColor = RED,
})

local AimSection = AimTab:Section({
    Title = "NPC Aim Assist",
    Desc = "Assistência de mira para NPCs de treinamento.",
})

AimSection:Toggle({
    Title = "Aim Assist",
    Desc = "Acompanha suavemente o NPC dentro do FOV.",
    Value = false,

    Callback = function(value)
        Config.Aim.Enabled = value
        CurrentTarget = nil

        Notify(
            "Aim Assist",
            value and "Ativado." or "Desativado."
        )
    end,
})

AimSection:Slider({
    Title = "FOV da Mira",
    Step = 1,

    Value = {
        Min = 1,
        Max = 90,
        Default = Config.Aim.FOV,
    },

    Callback = function(value)
        Config.Aim.FOV = value
    end,
})

AimSection:Slider({
    Title = "Alcance",
    Step = 1,

    Value = {
        Min = 10,
        Max = 500,
        Default = Config.Aim.Range,
    },

    Callback = function(value)
        Config.Aim.Range = value
    end,
})

AimSection:Slider({
    Title = "Suavidade",
    Step = 1,

    Value = {
        Min = 1,
        Max = 30,
        Default = Config.Aim.Smoothness,
    },

    Callback = function(value)
        Config.Aim.Smoothness = value
    end,
})

AimSection:Dropdown({
    Title = "Parte do alvo",

    Values = {
        "Head",
        "Chest",
        "HumanoidRootPart",
    },

    Value = "Head",

    Callback = function(value)
        Config.Aim.TargetPart = value
    end,
})

AimSection:Toggle({
    Title = "Verificar obstáculos",
    Desc = "Evita selecionar NPC atrás de paredes.",
    Value = true,

    Callback = function(value)
        Config.Aim.RequireLineOfSight = value
    end,
})

-- =========================================================
-- FOV VISUAL
-- =========================================================

local FOVTab = Window:Tab({
    Title = "FOV",
    Icon = "solar:focus-bold",
    IconColor = RED,
})

local FOVSection = FOVTab:Section({
    Title = "Campo de visão",
    Desc = "Configuração visual da área de mira.",
})

FOVSection:Toggle({
    Title = "Mostrar FOV",
    Desc = "Mostra o círculo no centro da tela.",
    Value = false,

    Callback = function(value)
        Config.Visual.FOVEnabled = value
        CreateFOV()
    end,
})

FOVSection:Slider({
    Title = "Tamanho visual",
    Step = 1,

    Value = {
        Min = 40,
        Max = 400,
        Default = Config.Visual.FOVRadius,
    },

    Callback = function(value)
        Config.Visual.FOVRadius = value
        UpdateFOV()
    end,
})

-- =========================================================
-- VISUAL NPC
-- =========================================================

local VisualTab = Window:Tab({
    Title = "Visual",
    Icon = "solar:eye-bold",
    IconColor = RED,
})

local VisualSection = VisualTab:Section({
    Title = "NPC Debug",
    Desc = "Ferramentas visuais para o campo de treinamento.",
})

VisualSection:Toggle({
    Title = "Highlight NPC",
    Desc = "Destaca os NPCs.",
    Value = false,

    Callback = function(value)
        Config.Visual.NPCHighlight = value
        RefreshNPCVisuals()
    end,
})

VisualSection:Toggle({
    Title = "Nome NPC",
    Desc = "Mostra o nome dos NPCs.",
    Value = false,

    Callback = function(value)
        Config.Visual.NPCName = value
        RefreshNPCVisuals()
    end,
})

VisualSection:Toggle({
    Title = "Distância NPC",
    Desc = "Mostra a distância até o NPC.",
    Value = false,

    Callback = function(value)
        Config.Visual.NPCDistance = value
        RefreshNPCVisuals()
    end,
})

VisualSection:Button({
    Title = "Limpar visuais",
    Desc = "Remove os indicadores dos NPCs.",

    Callback = function()
        Config.Visual.NPCHighlight = false
        Config.Visual.NPCName = false
        Config.Visual.NPCDistance = false

        ClearNPCVisuals()

        Notify(
            "Visual",
            "Indicadores removidos."
        )
    end,
})

-- =========================================================
-- STATUS
-- =========================================================

local StatusTab = Window:Tab({
    Title = "Status",
    Icon = "solar:chart-2-bold",
    IconColor = RED,
})

local StatusSection = StatusTab:Section({
    Title = "BZNX STATUS",
    Desc = "Informações do sistema.",
})

local StatusText = StatusSection:Paragraph({
    Title = "Status",
    Desc = "Carregando...",
})

-- =========================================================
-- SOBRE
-- =========================================================

local AboutTab = Window:Tab({
    Title = "Sobre",
    Icon = "solar:info-circle-bold",
    IconColor = RED,
})

local AboutSection = AboutTab:Section({
    Title = "BZNX FPS",
    Desc = "Centro de treinamento.",
})

AboutSection:Paragraph({
    Title = "PZNX / Brenno",
    Desc = "Painel de treinamento e desenvolvimento.",
})

AboutSection:Paragraph({
    Title = "Interface",
    Desc = "WindUI",
})

AboutSection:Paragraph({
    Title = "Modo",
    Desc = "NPC / Training",
})

-- =========================================================
-- LOOP PRINCIPAL
-- =========================================================

RunService.RenderStepped:Connect(function(deltaTime)
    if Config.Aim.Enabled then
        CurrentTarget = FindBestNPC()

        if CurrentTarget then
            AimAtTarget(
                CurrentTarget,
                deltaTime
            )
        end
    else
        CurrentTarget = nil
    end
end)

-- =========================================================
-- ATUALIZAÇÃO DO STATUS
-- =========================================================

task.spawn(function()
    while task.wait(1) do
        local camera = Workspace.CurrentCamera

        local cameraFOV =
            camera
            and math.floor(camera.FieldOfView)
            or 0

        local targetName =
            CurrentTarget
            and CurrentTarget.Name
            or "Nenhum"

        if StatusText and StatusText.SetDesc then
            StatusText:SetDesc(
                "NPCs: "
                .. tostring(#GetNPCs())

                .. "\nFOV da câmera: "
                .. tostring(cameraFOV)

                .. "\nFOV da mira: "
                .. tostring(Config.Aim.FOV)

                .. "\nAlcance: "
                .. tostring(Config.Aim.Range)

                .. "\nAlvo atual: "
                .. targetName
            )
        end
    end
end)

-- =========================================================
-- EVENTOS DO CONTAINER DE NPCs
-- =========================================================

local function ConnectNPCContainer(container)
    if not container then
        return
    end

    container.ChildAdded:Connect(function(child)
        task.wait(0.2)

        if IsAliveNPC(child) then
            CreateNPCVisual(child)
        end
    end)

    container.ChildRemoved:Connect(function(child)
        RemoveNPCVisual(child)

        if CurrentTarget
            and CurrentTarget:IsDescendantOf(child)
        then
            CurrentTarget = nil
        end
    end)
end

ConnectNPCContainer(
    GetNPCContainer()
)

Workspace.ChildAdded:Connect(function(child)
    if child.Name ==
        Config.General.NPCFolderName
    then
        ConnectNPCContainer(child)
        RefreshNPCVisuals()
    end
end)

-- =========================================================
-- FINAL
-- =========================================================

CreateFOV()
RefreshNPCVisuals()

Notify(
    "BZNX FPS",
    "Painel de treinamento carregado."
)

print("======================================")
print("          BZNX FPS TRAINING")
print("       PZNX / Brenno")
print("          WINDUI EDITION")
print("======================================")
