--[[
    KRATOS HUB - FINAL EDITION
    Steal An Egg Edition
    Version: 2.2.0 (Bypass BAC-3518 + Bugfixes)

    - CORREÇÃO: Ovos agora aparecem corretamente na lista
    - NOVO: Lista minimizável (mostra apenas Top 5)
    - NOVO: Filtros iniciam DESATIVADOS
    - NOVO: Botão Único Loop/TeleGuiado (Lógica Alternada)
    - BYPASS: Movimentação via CFrame Interpolation para evitar Kick
]]

--==================================================
-- SERVICES
--==================================================

local Players = game:GetService("Players")
local Workspace = game:GetService("Workspace")
local RunService = game:GetService("RunService")
local UserInputService = game:GetService("UserInputService")
local TweenService = game:GetService("TweenService")
local VirtualUser = game:GetService("VirtualUser")

local LocalPlayer = Players.LocalPlayer

--==================================================
-- GUI PARENT
--==================================================

local function getGuiParent()
    local parent
    pcall(function()
        if gethui then parent = gethui() end
    end)
    if not parent then parent = game:GetService("CoreGui") end
    return parent
end

pcall(function()
    local old = getGuiParent():FindFirstChild("KratosHub_V2")
    if old then old:Destroy() end
end)

--==================================================
-- SETTINGS
--==================================================

local Settings = {
    AutoFarm = false,
    ESP = false,
    LoopMode = false, -- false = TeleGuiado, true = Loop
    AntiAFK = true,
    BypassSpeed = 45,
    ListMinimized = false,

    SelectedRarities = {
        Common = false, Uncommon = false, Rare = false, Epic = false,
        Legendary = false, Mythic = false, Cosmic = false, Secret = false,
        Eternal = false, Divine = false
    }
}

local RarityOrder = {"Divine", "Eternal", "Secret", "Cosmic", "Mythic", "Legendary", "Epic", "Rare", "Uncommon", "Common"}
local RarityColors = {
    Divine = Color3.fromRGB(210, 220, 0), Eternal = Color3.fromRGB(255, 70, 180),
    Secret = Color3.fromRGB(80, 0, 130), Cosmic = Color3.fromRGB(160, 70, 255),
    Mythic = Color3.fromRGB(255, 80, 80), Legendary = Color3.fromRGB(255, 170, 0),
    Epic = Color3.fromRGB(175, 80, 255), Rare = Color3.fromRGB(70, 150, 255),
    Uncommon = Color3.fromRGB(80, 220, 100), Common = Color3.fromRGB(160, 160, 160)
}

--==================================================
-- CHARACTER & BYPASS
--==================================================

local Character, Humanoid, RootPart

local function updateCharacter()
    Character = LocalPlayer.Character or LocalPlayer.CharacterAdded:Wait()
    Humanoid = Character:WaitForChild("Humanoid")
    RootPart = Character:WaitForChild("HumanoidRootPart")
    Humanoid:SetStateEnabled(Enum.HumanoidStateType.FallingDown, false)
    Humanoid:SetStateEnabled(Enum.HumanoidStateType.Ragdoll, false)
end

updateCharacter()
LocalPlayer.CharacterAdded:Connect(function()
    task.wait(1)
    updateCharacter()
end)

--==================================================
-- HELPERS
--==================================================

local function formatNumber(number)
    number = tonumber(number) or 0
    if number >= 1e12 then return string.format("%.2fT", number / 1e12)
    elseif number >= 1e9 then return string.format("%.2fB", number / 1e9)
    elseif number >= 1e6 then return string.format("%.2fM", number / 1e6)
    elseif number >= 1e3 then return string.format("%.2fK", number / 1e3)
    else return tostring(math.floor(number)) end
end

local function parseNumber(value)
    if value == nil then return nil end
    if typeof(value) == "number" then return value end
    if typeof(value) ~= "string" then return nil end
    local clean = value:gsub(",", ""):gsub("%$", ""):gsub("%s+", "")
    local multiplier = 1
    if clean:upper():find("T") then multiplier = 1e12 clean = clean:gsub("[Tt]", "")
    elseif clean:upper():find("B") then multiplier = 1e9 clean = clean:gsub("[Bb]", "")
    elseif clean:upper():find("M") then multiplier = 1e6 clean = clean:gsub("[Mm]", "")
    elseif clean:upper():find("K") then multiplier = 1e3 clean = clean:gsub("[Kk]", "") end
    local number = tonumber(clean)
    return number and (number * multiplier) or nil
end

local function getPosition(object)
    if not object then return nil end
    if object:IsA("BasePart") then return object.Position end
    if object:IsA("Model") then
        local p = object.PrimaryPart or object:FindFirstChildWhichIsA("BasePart", true)
        return p and p.Position
    end
    return nil
end

local function getAdornee(object)
    if object:IsA("BasePart") then return object end
    if object:IsA("Model") then return object.PrimaryPart or object:FindFirstChildWhichIsA("BasePart", true) end
    return nil
end

--==================================================
-- EGG LOGIC (FIXED)
--==================================================

local function getEggContainer()
    local names = {"Eggs", "Egg", "EggsFolder", "EggContainer", "EggsContainer", "Pets", "Drops", "Items"}
    for _, name in ipairs(names) do
        local found = Workspace:FindFirstChild(name, true)
        if found then return found end
    end
    return Workspace
end

local function getRarity(object)
    local attributes = {"Rarity", "rarity", "Tier", "tier", "Grade", "grade"}
    for _, attribute in ipairs(attributes) do
        local value = object:GetAttribute(attribute)
        if value then
            local text = tostring(value)
            for _, rarity in ipairs(RarityOrder) do
                if text:lower():find(rarity:lower(), 1, true) then return rarity end
            end
        end
    end
    for _, rarity in ipairs(RarityOrder) do
        if object.Name:lower():find(rarity:lower(), 1, true) then return rarity end
    end
    return "Common"
end

local ValueAttributeNames = {"Value", "value", "Price", "price", "Cost", "cost", "Worth", "worth", "Cash", "cash", "Money", "money", "EggValue", "SellValue"}

local function getEggValue(object)
    for _, attribute in ipairs(ValueAttributeNames) do
        local value = object:GetAttribute(attribute)
        if value ~= nil then
            local number = parseNumber(value)
            if number then return number end
        end
    end
    for _, descendant in ipairs(object:GetDescendants()) do
        if (descendant:IsA("NumberValue") or descendant:IsA("IntValue")) and table.find(ValueAttributeNames, descendant.Name) then
            return descendant.Value
        end
        if descendant:IsA("StringValue") and table.find(ValueAttributeNames, descendant.Name) then
            local number = parseNumber(descendant.Value)
            if number then return number end
        end
        if descendant:IsA("TextLabel") or descendant:IsA("TextButton") or descendant:IsA("TextBox") then
            local number = parseNumber(descendant.Text)
            if number and number > 0 then return number end
        end
    end
    return 0
end

local function isEgg(object)
    if not object or object:IsA("Terrain") then return false end
    local name = object.Name:lower()
    local keywords = {"egg", "imp", "rhino", "demon", "hound", "pet", "dragon", "cat", "dog"}
    for _, keyword in ipairs(keywords) do
        if name:find(keyword, 1, true) then return true end
    end
    if object:GetAttribute("Rarity") or object:GetAttribute("Value") or object:GetAttribute("Price") or object:GetAttribute("Worth") then
        return true
    end
    return false
end

local function getEggs()
    local container = getEggContainer()
    if not container then return {} end
    local eggs = {}
    for _, object in ipairs(container:GetChildren()) do
        if isEgg(object) then
            local rarity = getRarity(object)
            -- Se nenhum filtro estiver ativo, mostra todos. Se algum estiver, filtra.
            local anyFilterActive = false
            for _, v in pairs(Settings.SelectedRarities) do if v then anyFilterActive = true break end end
            
            if not anyFilterActive or Settings.SelectedRarities[rarity] then
                table.insert(eggs, {
                    Object = object, Name = object.Name, Rarity = rarity,
                    Value = getEggValue(object), Position = getPosition(object)
                })
            end
        end
    end
    table.sort(eggs, function(a, b) return a.Value > b.Value end)
    return eggs
end

local function getBestEgg()
    local eggs = getEggs()
    return eggs[1]
end

--==================================================
-- GUI CONSTRUCTION
--==================================================

local ScreenGui = Instance.new("ScreenGui")
ScreenGui.Name = "KratosHub_V2"
ScreenGui.ResetOnSpawn = false
ScreenGui.IgnoreGuiInset = true
ScreenGui.ZIndexBehavior = Enum.ZIndexBehavior.Sibling
ScreenGui.Parent = getGuiParent()

local Main = Instance.new("Frame")
Main.Name = "Main"
Main.Size = UDim2.fromOffset(335, 520)
Main.Position = UDim2.new(0.5, -167, 0.5, -260)
Main.BackgroundColor3 = Color3.fromRGB(8, 17, 12)
Main.BorderSizePixel = 0
Main.Parent = ScreenGui

local MainCorner = Instance.new("UICorner")
MainCorner.CornerRadius = UDim.new(0, 14)
MainCorner.Parent = Main

local MainStroke = Instance.new("UIStroke")
MainStroke.Thickness = 1.5
MainStroke.Color = Color3.fromRGB(20, 75, 35)
MainStroke.Parent = Main

local Header = Instance.new("Frame")
Header.Size = UDim2.new(1, 0, 0, 62)
Header.BackgroundTransparency = 1
Header.Parent = Main

local Logo = Instance.new("Frame")
Logo.Size = UDim2.fromOffset(38, 38)
Logo.Position = UDim2.fromOffset(10, 12)
Logo.BackgroundColor3 = Color3.fromRGB(25, 35, 20)
Logo.Parent = Header
Instance.new("UICorner", Logo).CornerRadius = UDim.new(1, 0)

local LogoText = Instance.new("TextLabel")
LogoText.BackgroundTransparency = 1
LogoText.Size = UDim2.fromScale(1, 1)
LogoText.Text = "K"
LogoText.Font = Enum.Font.GothamBold
LogoText.TextSize = 18
LogoText.TextColor3 = Color3.fromRGB(180, 220, 40)
LogoText.Parent = Logo

local Title = Instance.new("TextLabel")
Title.BackgroundTransparency = 1
Title.Position = UDim2.fromOffset(58, 10)
Title.Size = UDim2.fromOffset(190, 25)
Title.Text = "KRATOS HUB"
Title.Font = Enum.Font.GothamBold
Title.TextSize = 13
Title.TextXAlignment = Enum.TextXAlignment.Left
Title.TextColor3 = Color3.fromRGB(240, 245, 240)
Title.Parent = Header

local Subtitle = Instance.new("TextLabel")
Subtitle.BackgroundTransparency = 1
Subtitle.Position = UDim2.fromOffset(58, 32)
Subtitle.Size = UDim2.fromOffset(190, 18)
Subtitle.Text = "STEAL EGG SYSTEM"
Subtitle.Font = Enum.Font.Gotham
Subtitle.TextSize = 7
Subtitle.TextXAlignment = Enum.TextXAlignment.Left
Subtitle.TextColor3 = Color3.fromRGB(75, 110, 80)
Subtitle.Parent = Header

local MinButton = Instance.new("TextButton")
MinButton.Size = UDim2.fromOffset(25, 25)
MinButton.Position = UDim2.new(1, -45, 0, 18)
MinButton.BackgroundTransparency = 1
MinButton.Text = "−"
MinButton.TextColor3 = Color3.fromRGB(100, 120, 105)
MinButton.Font = Enum.Font.GothamBold
MinButton.TextSize = 20
MinButton.Parent = Header

local Featured = Instance.new("Frame")
Featured.Size = UDim2.new(1, -20, 0, 65)
Featured.Position = UDim2.fromOffset(10, 62)
Featured.BackgroundColor3 = Color3.fromRGB(9, 21, 14)
Featured.BorderSizePixel = 0
Featured.Parent = Main
Instance.new("UICorner", Featured).CornerRadius = UDim.new(0, 11)
local FeaturedStroke = Instance.new("UIStroke", Featured)
FeaturedStroke.Thickness = 1
FeaturedStroke.Color = Color3.fromRGB(30, 75, 40)

local EggIcon = Instance.new("Frame")
EggIcon.Size = UDim2.fromOffset(48, 48)
EggIcon.Position = UDim2.fromOffset(7, 8)
EggIcon.BackgroundColor3 = Color3.fromRGB(18, 31, 21)
EggIcon.Parent = Featured
Instance.new("UICorner", EggIcon).CornerRadius = UDim.new(0, 9)

local EggEmoji = Instance.new("TextLabel")
EggEmoji.BackgroundTransparency = 1
EggEmoji.Size = UDim2.fromScale(1, 1)
EggEmoji.Text = "🥚"
EggEmoji.Font = Enum.Font.Gotham
EggEmoji.TextSize = 23
EggEmoji.Parent = EggIcon

local FeaturedName = Instance.new("TextLabel")
FeaturedName.BackgroundTransparency = 1
FeaturedName.Position = UDim2.fromOffset(63, 8)
FeaturedName.Size = UDim2.fromOffset(160, 20)
FeaturedName.Text = "PROCURANDO..."
FeaturedName.Font = Enum.Font.GothamBold
FeaturedName.TextSize = 11
FeaturedName.TextXAlignment = Enum.TextXAlignment.Left
FeaturedName.TextColor3 = Color3.fromRGB(235, 240, 235)
FeaturedName.Parent = Featured

local FeaturedRarity = Instance.new("TextLabel")
FeaturedRarity.BackgroundTransparency = 1
FeaturedRarity.Position = UDim2.fromOffset(63, 30)
FeaturedRarity.Size = UDim2.fromOffset(100, 17)
FeaturedRarity.Text = "Common"
FeaturedRarity.Font = Enum.Font.Gotham
FeaturedRarity.TextSize = 8
FeaturedRarity.TextXAlignment = Enum.TextXAlignment.Left
FeaturedRarity.Parent = Featured

local FeaturedValue = Instance.new("TextLabel")
FeaturedValue.BackgroundTransparency = 1
FeaturedValue.Position = UDim2.new(1, -100, 0, 20)
FeaturedValue.Size = UDim2.fromOffset(90, 20)
FeaturedValue.Text = "$0"
FeaturedValue.Font = Enum.Font.GothamBold
FeaturedValue.TextSize = 12
FeaturedValue.TextXAlignment = Enum.TextXAlignment.Right
FeaturedValue.TextColor3 = Color3.fromRGB(75, 255, 145)
FeaturedValue.Parent = Featured

local List = Instance.new("ScrollingFrame")
List.Size = UDim2.new(1, -20, 0, 178)
List.Position = UDim2.fromOffset(10, 135)
List.BackgroundTransparency = 1
List.BorderSizePixel = 0
List.ScrollBarThickness = 2
List.CanvasSize = UDim2.new()
List.Parent = Main
Instance.new("UIListLayout", List).Padding = UDim.new(0, 5)

local ListToggle = Instance.new("TextButton")
ListToggle.Size = UDim2.fromOffset(80, 20)
ListToggle.Position = UDim2.new(1, -90, 0, 125)
ListToggle.BackgroundColor3 = Color3.fromRGB(15, 30, 20)
ListToggle.Text = "Minimizar"
ListToggle.TextColor3 = Color3.fromRGB(150, 160, 150)
ListToggle.Font = Enum.Font.Gotham
ListToggle.TextSize = 8
ListToggle.Parent = Main
Instance.new("UICorner", ListToggle).CornerRadius = UDim.new(0, 5)

local SectionTitle = Instance.new("TextLabel")
SectionTitle.BackgroundTransparency = 1
SectionTitle.Position = UDim2.fromOffset(11, 321)
SectionTitle.Size = UDim2.fromOffset(160, 22)
SectionTitle.Text = "MODO: TELEGUIDADO"
SectionTitle.Font = Enum.Font.GothamBold
SectionTitle.TextSize = 9
SectionTitle.TextXAlignment = Enum.TextXAlignment.Left
SectionTitle.TextColor3 = Color3.fromRGB(140, 180, 145)
SectionTitle.Parent = Main

local function createToggleButton(parent, position)
    local button = Instance.new("TextButton")
    button.Size = UDim2.fromOffset(38, 22)
    button.Position = position
    button.BackgroundColor3 = Color3.fromRGB(18, 31, 22)
    button.Text = ""
    button.AutoButtonColor = false
    button.Parent = parent
    Instance.new("UICorner", button).CornerRadius = UDim.new(1, 0)

    local knob = Instance.new("Frame")
    knob.Size = UDim2.fromOffset(14, 14)
    knob.Position = UDim2.fromOffset(4, 4)
    knob.BackgroundColor3 = Color3.fromRGB(145, 160, 150)
    knob.Parent = button
    Instance.new("UICorner", knob).CornerRadius = UDim.new(1, 0)

    local state = false
    local function setState(value)
        state = value
        if state then
            button.BackgroundColor3 = Color3.fromRGB(36, 90, 44)
            TweenService:Create(knob, TweenInfo.new(0.15), {Position = UDim2.new(1, -18, 0, 4), BackgroundColor3 = Color3.fromRGB(100, 255, 125)}):Play()
        else
            button.BackgroundColor3 = Color3.fromRGB(18, 31, 22)
            TweenService:Create(knob, TweenInfo.new(0.15), {Position = UDim2.fromOffset(4, 4), BackgroundColor3 = Color3.fromRGB(145, 160, 150)}):Play()
        end
    end
    button.MouseButton1Click:Connect(function() setState(not state) end)
    return button, function() return state end, setState
end

local LoopToggle, getLoopState, setLoopState = createToggleButton(Main, UDim2.new(1, -55, 0, 320))

local AutoFarmButton = Instance.new("TextButton")
AutoFarmButton.Size = UDim2.new(1, -20, 0, 35)
AutoFarmButton.Position = UDim2.fromOffset(10, 350)
AutoFarmButton.BackgroundColor3 = Color3.fromRGB(13, 27, 17)
AutoFarmButton.Text = "AUTO FARM : OFF"
AutoFarmButton.Font = Enum.Font.GothamBold
AutoFarmButton.TextSize = 10
AutoFarmButton.TextColor3 = Color3.fromRGB(140, 150, 145)
AutoFarmButton.Parent = Main
Instance.new("UICorner", AutoFarmButton).CornerRadius = UDim.new(0, 8)
local AutoStroke = Instance.new("UIStroke", AutoFarmButton)
AutoStroke.Thickness = 1
AutoStroke.Color = Color3.fromRGB(30, 65, 35)

local RarityTitle = Instance.new("TextLabel")
RarityTitle.BackgroundTransparency = 1
RarityTitle.Position = UDim2.fromOffset(11, 394)
RarityTitle.Size = UDim2.fromOffset(250, 18)
RarityTitle.Text = "FILTRE RARIDADE"
RarityTitle.Font = Enum.Font.GothamBold
RarityTitle.TextSize = 8
RarityTitle.TextXAlignment = Enum.TextXAlignment.Left
RarityTitle.TextColor3 = Color3.fromRGB(120, 165, 125)
RarityTitle.Parent = Main

local rarityNames = {"Cosmic", "Secret", "Eternal", "Divine"}
for index, rarity in ipairs(rarityNames) do
    local row = math.floor((index - 1) / 2)
    local column = (index - 1) % 2
    local button = Instance.new("TextButton")
    button.Size = UDim2.fromOffset(145, 28)
    button.Position = UDim2.fromOffset(10 + column * 152, 416 + row * 32)
    button.BackgroundColor3 = Color3.fromRGB(12, 24, 15)
    button.Text = ""
    button.Parent = Main
    Instance.new("UICorner", button).CornerRadius = UDim.new(0, 7)

    local indicator = Instance.new("Frame")
    indicator.Size = UDim2.fromOffset(15, 15)
    indicator.Position = UDim2.fromOffset(6, 6)
    indicator.BackgroundColor3 = Color3.fromRGB(35, 42, 35)
    indicator.Parent = button
    Instance.new("UICorner", indicator).CornerRadius = UDim.new(0, 4)

    local label = Instance.new("TextLabel")
    label.BackgroundTransparency = 1
    label.Position = UDim2.fromOffset(28, 0)
    label.Size = UDim2.new(1, -32, 1, 0)
    label.Text = rarity
    label.Font = Enum.Font.Gotham
    label.TextSize = 8
    label.TextXAlignment = Enum.TextXAlignment.Left
    label.TextColor3 = Color3.fromRGB(210, 215, 210)
    label.TextTransparency = 0.5
    label.Parent = button

    button.MouseButton1Click:Connect(function()
        Settings.SelectedRarities[rarity] = not Settings.SelectedRarities[rarity]
        indicator.BackgroundColor3 = Settings.SelectedRarities[rarity] and RarityColors[rarity] or Color3.fromRGB(35, 42, 35)
        label.TextTransparency = Settings.SelectedRarities[rarity] and 0 or 0.5
    end)
end

local Ball = Instance.new("TextButton")
Ball.Name = "ToggleBall"
Ball.Size = UDim2.fromOffset(52, 52)
Ball.Position = UDim2.new(0, 20, 0.5, -26)
Ball.BackgroundColor3 = Color3.fromRGB(9, 20, 12)
Ball.Text = "K"
Ball.TextColor3 = Color3.fromRGB(180, 235, 70)
Ball.Font = Enum.Font.GothamBold
Ball.TextSize = 19
Ball.Visible = false
Ball.Parent = ScreenGui
Instance.new("UICorner", Ball).CornerRadius = UDim.new(1, 0)
local BallStroke = Instance.new("UIStroke", Ball)
BallStroke.Thickness = 2
BallStroke.Color = Color3.fromRGB(80, 190, 75)

local function makeDraggable(frame, dragArea)
    local dragging, dragInput, dragStart, startPos
    dragArea.InputBegan:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
            dragging = true
            dragStart = input.Position
            startPos = frame.Position
            input.Changed:Connect(function()
                if input.UserInputState == Enum.UserInputState.End then dragging = false end
            end)
        end
    end)
    dragArea.InputChanged:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseMovement or input.UserInputType == Enum.UserInputType.Touch then
            dragInput = input
        end
    end)
    UserInputService.InputChanged:Connect(function(input)
        if input == dragInput and dragging then
            local delta = input.Position - dragStart
            frame.Position = UDim2.new(startPos.X.Scale, startPos.X.Offset + delta.X, startPos.Y.Scale, startPos.Y.Offset + delta.Y)
        end
    end)
end

makeDraggable(Main, Header)
makeDraggable(Ball, Ball)

MinButton.MouseButton1Click:Connect(function() Main.Visible = false Ball.Visible = true end)
Ball.MouseButton1Click:Connect(function() Main.Visible = true Ball.Visible = false end)

ListToggle.MouseButton1Click:Connect(function()
    Settings.ListMinimized = not Settings.ListMinimized
    ListToggle.Text = Settings.ListMinimized and "Expandir" or "Minimizar"
end)

--==================================================
-- BYPASS MOVEMENT
--==================================================

local function safeMoveTo(targetPosition)
    if not RootPart then return false end
    local distance = (RootPart.Position - targetPosition).Magnitude
    if distance <= 4 then return true end

    local startTime = tick()
    while Settings.AutoFarm and RootPart and (RootPart.Position - targetPosition).Magnitude > 4 do
        if tick() - startTime > 10 then break end
        local nextPos = RootPart.Position:Lerp(targetPosition, 0.1)
        RootPart.CFrame = CFrame.new(nextPos)
        RunService.Heartbeat:Wait()
    end
    return (RootPart.Position - targetPosition).Magnitude <= 5
end

--==================================================
-- AUTO FARM LOGIC (FIXED)
--==================================================

local farming = false
local OriginalPosition = nil

local function farmOnce()
    if farming or not Settings.AutoFarm or not RootPart then return end
    local best = getBestEgg()
    if not best or not best.Object or not best.Position then return end

    farming = true
    if not OriginalPosition then OriginalPosition = RootPart.Position end

    safeMoveTo(best.Position)
    task.wait(0.3)
    
    if RootPart then RootPart.CFrame = CFrame.new(best.Position + Vector3.new(0, 2, 0)) end
    task.wait(0.5)

    local timeout = tick() + 2
    while Settings.AutoFarm and best.Object and best.Object.Parent and tick() < timeout do
        task.wait(0.1)
    end

    if OriginalPosition and Settings.AutoFarm then
        safeMoveTo(OriginalPosition)
    end

    -- LÓGICA SOLICITADA: Se for TeleGuiado (LoopMode = false), desativa AutoFarm ao voltar
    if not Settings.LoopMode then
        Settings.AutoFarm = false
        updateAutoButton()
    end

    OriginalPosition = nil
    farming = false
end

function updateAutoButton()
    if Settings.AutoFarm then
        AutoFarmButton.Text = "AUTO FARM : ON"
        AutoFarmButton.TextColor3 = Color3.fromRGB(90, 255, 130)
        AutoFarmButton.BackgroundColor3 = Color3.fromRGB(13, 42, 22)
    else
        AutoFarmButton.Text = "AUTO FARM : OFF"
        AutoFarmButton.TextColor3 = Color3.fromRGB(140, 150, 145)
        AutoFarmButton.BackgroundColor3 = Color3.fromRGB(13, 27, 17)
    end
end

AutoFarmButton.MouseButton1Click:Connect(function()
    Settings.AutoFarm = not Settings.AutoFarm
    updateAutoButton()
    OriginalPosition = nil
end)

LoopToggle.MouseButton1Click:Connect(function()
    Settings.LoopMode = getLoopState()
    SectionTitle.Text = Settings.LoopMode and "MODO: LOOP" or "MODO: TELEGUIDADO"
end)

task.spawn(function()
    while task.wait(0.35) do
        if Settings.AutoFarm then
            farmOnce()
            if Settings.LoopMode then task.wait(0.2) end
        end
    end
end)

--==================================================
-- ESP & LIST (FIXED)
--==================================================

local ESPObjects = {}
local SelectedEgg = nil

local function updateFeatured()
    local best = getBestEgg()
    SelectedEgg = best
    if not best then
        FeaturedName.Text = "NENHUM OVO"
        FeaturedRarity.Text = "Nenhuma raridade"
        FeaturedRarity.TextColor3 = Color3.fromRGB(120, 130, 120)
        FeaturedValue.Text = "$0"
        return
    end
    FeaturedName.Text = best.Name
    FeaturedRarity.Text = best.Rarity
    FeaturedRarity.TextColor3 = RarityColors[best.Rarity] or Color3.new(1,1,1)
    FeaturedValue.Text = "$" .. formatNumber(best.Value)
end

local function createEggRow(data, index)
    local row = Instance.new("Frame")
    row.Size = UDim2.new(1, -5, 0, 38)
    row.BackgroundColor3 = Color3.fromRGB(11, 25, 16)
    row.LayoutOrder = index
    row.Parent = List
    Instance.new("UICorner", row).CornerRadius = UDim.new(0, 8)

    local rank = Instance.new("TextLabel")
    rank.BackgroundTransparency = 1
    rank.Size = UDim2.fromOffset(35, 38)
    rank.Text = tostring(index)
    rank.Font = Enum.Font.GothamBold
    rank.TextSize = 8
    rank.TextColor3 = Color3.fromRGB(95, 115, 100)
    rank.Parent = row

    local icon = Instance.new("Frame")
    icon.Size = UDim2.fromOffset(28, 28)
    icon.Position = UDim2.fromOffset(34, 5)
    icon.BackgroundColor3 = Color3.fromRGB(16, 31, 20)
    icon.Parent = row
    Instance.new("UICorner", icon).CornerRadius = UDim.new(0, 6)

    local emoji = Instance.new("TextLabel")
    emoji.BackgroundTransparency = 1
    emoji.Size = UDim2.fromScale(1, 1)
    emoji.Text = "🥚"
    emoji.Font = Enum.Font.Gotham
    emoji.TextSize = 14
    emoji.Parent = icon

    local name = Instance.new("TextLabel")
    name.BackgroundTransparency = 1
    name.Position = UDim2.fromOffset(68, 3)
    name.Size = UDim2.fromOffset(115, 18)
    name.Text = data.Name
    name.Font = Enum.Font.GothamBold
    name.TextSize = 8
    name.TextXAlignment = Enum.TextXAlignment.Left
    name.TextColor3 = Color3.fromRGB(225, 230, 225)
    name.Parent = row

    local rarity = Instance.new("TextLabel")
    rarity.BackgroundTransparency = 1
    rarity.Position = UDim2.fromOffset(68, 20)
    rarity.Size = UDim2.fromOffset(90, 14)
    rarity.Text = data.Rarity
    rarity.Font = Enum.Font.Gotham
    rarity.TextSize = 7
    rarity.TextXAlignment = Enum.TextXAlignment.Left
    rarity.TextColor3 = RarityColors[data.Rarity] or Color3.new(1,1,1)
    rarity.Parent = row

    local value = Instance.new("TextLabel")
    value.BackgroundTransparency = 1
    value.Position = UDim2.new(1, -105, 5 / 38, 0)
    value.Size = UDim2.fromOffset(95, 28)
    value.Text = "$" .. formatNumber(data.Value)
    value.Font = Enum.Font.GothamBold
    value.TextSize = 9
    value.TextXAlignment = Enum.TextXAlignment.Right
    value.TextColor3 = Color3.fromRGB(65, 255, 145)
    value.Parent = row

    local selectButton = Instance.new("TextButton")
    selectButton.BackgroundTransparency = 1
    selectButton.Size = UDim2.fromScale(1, 1)
    selectButton.Text = ""
    selectButton.Parent = row
    selectButton.MouseButton1Click:Connect(function()
        SelectedEgg = data
        FeaturedName.Text = data.Name
        FeaturedRarity.Text = data.Rarity
        FeaturedRarity.TextColor3 = RarityColors[data.Rarity] or Color3.new(1,1,1)
        FeaturedValue.Text = "$" .. formatNumber(data.Value)
    end)
end

local function updateEggList()
    local eggs = getEggs()
    for _, child in ipairs(List:GetChildren()) do if child:IsA("Frame") then child:Destroy() end end
    
    local limit = Settings.ListMinimized and 5 or 20
    for index = 1, math.min(#eggs, limit) do 
        createEggRow(eggs[index], index) 
    end
    List.CanvasSize = UDim2.new(0, 0, 0, math.max(0, #eggs * 43))
    updateFeatured()
end

local function createESP(object, data)
    if ESPObjects[object] then return end
    local adornee = getAdornee(object)
    if not adornee then return end

    local billboard = Instance.new("BillboardGui")
    billboard.Name = "KratosESP"
    billboard.Adornee = adornee
    billboard.AlwaysOnTop = true
    billboard.Size = UDim2.fromOffset(170, 42)
    billboard.StudsOffset = Vector3.new(0, 3, 0)
    billboard.Parent = getGuiParent()

    local frame = Instance.new("Frame")
    frame.Size = UDim2.fromScale(1, 1)
    frame.BackgroundColor3 = Color3.fromRGB(6, 15, 9)
    frame.BackgroundTransparency = 0.15
    frame.Parent = billboard
    Instance.new("UICorner", frame).CornerRadius = UDim.new(0, 7)

    local label = Instance.new("TextLabel")
    label.BackgroundTransparency = 1
    label.Size = UDim2.fromScale(1, 1)
    label.Text = data.Name .. "\n" .. data.Rarity .. " • $" .. formatNumber(data.Value)
    label.Font = Enum.Font.GothamBold
    label.TextSize = 8
    label.TextColor3 = Color3.fromRGB(235, 245, 235)
    label.Parent = frame

    ESPObjects[object] = billboard
end

local function updateESP()
    if not Settings.ESP then
        for object, esp in pairs(ESPObjects) do pcall(function() esp:Destroy() end) ESPObjects[object] = nil end
        return
    end
    local eggs = getEggs()
    for _, data in ipairs(eggs) do
        if data.Object and data.Object.Parent then createESP(data.Object, data) end
    end
    for object in pairs(ESPObjects) do
        if not object or not object.Parent then
            pcall(function() ESPObjects[object]:Destroy() end)
            ESPObjects[object] = nil
        end
    end
end

Featured.InputBegan:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseButton1 then
        Settings.ESP = not Settings.ESP
        updateESP()
    end
end)

LocalPlayer.Idled:Connect(function()
    if Settings.AntiAFK then
        pcall(function()
            VirtualUser:CaptureController()
            VirtualUser:ClickButton2(Vector2.new(0, 0))
        end)
    end
end)

task.spawn(function()
    while task.wait(1) do
        updateEggList()
        updateESP()
    end
end)

updateAutoButton()
updateEggList()

pcall(function()
    game:GetService("StarterGui"):SetCore("SendNotification", {
        Title = "Kratos Hub",
        Text = "Bypass Ativado! Hub carregado.",
        Duration = 4
    })
end)
