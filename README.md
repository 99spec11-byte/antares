-- game: Murder Mystery 2 | Roblox
-- executor: Xeno | language: Lua
-- features: silent aim, kill aura, esp, speed, fly, coin farm, GUI completa
-- NOTA: troca IMAGE_ID pelo asset ID do decal Yumeko no Roblox
--       (pesquisa "yumeko kakegurui" no catalog.roblox.com, copia o ID do URL)

local IMAGE_ID = "rbxassetid://7072369038" -- substitui pelo ID correto

-- ══════════════════════════════════════════════
--  SERVICES
-- ══════════════════════════════════════════════
local Players    = game:GetService("Players")
local RunService = game:GetService("RunService")
local UIS        = game:GetService("UserInputService")
local TweenSvc   = game:GetService("TweenService")
local lp         = Players.LocalPlayer
local cam        = workspace.CurrentCamera
local mouse      = lp:GetMouse()

-- ══════════════════════════════════════════════
--  CONFIG
-- ══════════════════════════════════════════════
local cfg = {
    -- combat
    silentAim   = true,
    killAura    = false,
    auraRange   = 15,
    autoThrow   = false,
    -- movement
    speed       = true,
    walkspd     = 28,
    fly         = false,
    -- misc
    esp         = true,
    coinFarm    = true,
    noFog       = true,
    -- keybinds
    keys = {
        fly       = Enum.KeyCode.F,
        killAura  = Enum.KeyCode.K,
        silentAim = Enum.KeyCode.X,
        autoThrow = Enum.KeyCode.T,
        coinFarm  = Enum.KeyCode.C,
        gui       = Enum.KeyCode.RightShift,
    }
}

-- ══════════════════════════════════════════════
--  UTIL
-- ══════════════════════════════════════════════
local function getNearestPlayer(exceptSelf)
    local nearest, dist = nil, math.huge
    for _, plr in ipairs(Players:GetPlayers()) do
        if plr == lp then continue end
        local char = plr.Character
        if not char then continue end
        local root = char:FindFirstChild("HumanoidRootPart")
        local hum  = char:FindFirstChildOfClass("Humanoid")
        if not root or not hum or hum.Health <= 0 then continue end
        local d = (root.Position - cam.CFrame.Position).Magnitude
        if d < dist then dist = d; nearest = plr end
    end
    return nearest
end

local function getRole(plr)
    local ok, v = pcall(function()
        return plr.GameStats.Role.Value
    end)
    return ok and v or "Innocent"
end

local function tween(obj, props, t)
    TweenSvc:Create(obj, TweenInfo.new(t or 0.2), props):Play()
end

-- ══════════════════════════════════════════════
--  NO FOG
-- ══════════════════════════════════════════════
if cfg.noFog then
    for _, v in ipairs(game:GetDescendants()) do
        if v:IsA("Atmosphere") or v:IsA("Sky") then
            pcall(function() v.Density = 0 end)
        end
    end
end

-- ══════════════════════════════════════════════
--  SILENT AIM — namecall hook
-- ══════════════════════════════════════════════
local mt = getrawmetatable(game)
setreadonly(mt, false)
local oldNC = mt.__namecall
mt.__namecall = newcclosure(function(self, ...)
    local method = getnamecallmethod()
    if cfg.silentAim and method == "FireServer" then
        local args = {...}
        -- MM2 gun remote detectado pelo nome do RemoteEvent pai
        local remName = tostring(self.Name):lower()
        if remName:find("gun") or remName:find("shoot") or remName:find("fire") or remName:find("remote") then
            local target = getNearestPlayer()
            if target and target.Character then
                local root = target.Character:FindFirstChild("HumanoidRootPart")
                if root then
                    -- substitui CFrame/posição nos args
                    for i, v in ipairs(args) do
                        if typeof(v) == "CFrame" then
                            args[i] = CFrame.new(root.Position)
                        elseif typeof(v) == "Vector3" then
                            args[i] = root.Position
                        end
                    end
                    return oldNC(self, table.unpack(args))
                end
            end
        end
    end
    return oldNC(self, ...)
end)
setreadonly(mt, true)

-- ══════════════════════════════════════════════
--  KILL AURA (murderer — faca)
-- ══════════════════════════════════════════════
task.spawn(function()
    while true do
        task.wait(0.1)
        if not cfg.killAura then continue end
        local char = lp.Character
        if not char then continue end
        local root = char:FindFirstChild("HumanoidRootPart")
        if not root then continue end
        for _, plr in ipairs(Players:GetPlayers()) do
            if plr == lp then continue end
            local ec = plr.Character
            if not ec then continue end
            local er = ec:FindFirstChild("HumanoidRootPart")
            local eh = ec:FindFirstChildOfClass("Humanoid")
            if not er or not eh or eh.Health <= 0 then continue end
            if (root.Position - er.Position).Magnitude <= cfg.auraRange then
                -- teleporta faca até o alvo
                root.CFrame = CFrame.new(er.Position + Vector3.new(0,0,1))
                task.wait(0.05)
            end
        end
    end
end)

-- ══════════════════════════════════════════════
--  AUTO THROW (arremessa faca automaticamente)
-- ══════════════════════════════════════════════
task.spawn(function()
    while true do
        task.wait(0.4)
        if not cfg.autoThrow then continue end
        local char = lp.Character
        if not char then continue end
        -- procura tool de faca equipada
        local knife = char:FindFirstChild("Knife") or char:FindFirstChildWhichIsA("Tool")
        if not knife then continue end
        local target = getNearestPlayer()
        if not target or not target.Character then continue end
        local root = target.Character:FindFirstChild("HumanoidRootPart")
        if not root then continue end
        -- dispara remote de throw
        for _, v in ipairs(knife:GetDescendants()) do
            if v:IsA("RemoteEvent") or v:IsA("RemoteFunction") then
                pcall(function()
                    v:FireServer(root.Position)
                end)
            end
        end
    end
end)

-- ══════════════════════════════════════════════
--  SPEED
-- ══════════════════════════════════════════════
local function applySpeed()
    local char = lp.Character
    if not char then return end
    local hum = char:FindFirstChildOfClass("Humanoid")
    if hum then hum.WalkSpeed = cfg.speed and cfg.walkspd or 16 end
end
lp.CharacterAdded:Connect(function() task.wait(0.5); applySpeed() end)
applySpeed()

-- ══════════════════════════════════════════════
--  FLY
-- ══════════════════════════════════════════════
local flyConn, flyBV, flyBG
local flyActive = false

local function startFly()
    local char = lp.Character
    if not char then return end
    local root = char:FindFirstChild("HumanoidRootPart")
    local hum  = char:FindFirstChildOfClass("Humanoid")
    if not root or not hum then return end
    hum.PlatformStand = true
    flyBG = Instance.new("BodyGyro", root)
    flyBG.MaxTorque = Vector3.new(1e5,1e5,1e5); flyBG.P = 9e3
    flyBV = Instance.new("BodyVelocity", root)
    flyBV.MaxForce = Vector3.new(1e5,1e5,1e5); flyBV.Velocity = Vector3.zero
    flyConn = RunService.RenderStepped:Connect(function()
        local dir = Vector3.zero
        local cf  = cam.CFrame
        if UIS:IsKeyDown(Enum.KeyCode.W) then dir += cf.LookVector end
        if UIS:IsKeyDown(Enum.KeyCode.S) then dir -= cf.LookVector end
        if UIS:IsKeyDown(Enum.KeyCode.A) then dir -= cf.RightVector end
        if UIS:IsKeyDown(Enum.KeyCode.D) then dir += cf.RightVector end
        if UIS:IsKeyDown(Enum.KeyCode.Space) then dir += Vector3.new(0,1,0) end
        if UIS:IsKeyDown(Enum.KeyCode.LeftControl) then dir -= Vector3.new(0,1,0) end
        flyBV.Velocity = dir * 55
        flyBG.CFrame   = cf
    end)
    flyActive = true
end

local function stopFly()
    if flyConn then flyConn:Disconnect() end
    if flyBV then flyBV:Destroy() end
    if flyBG then flyBG:Destroy() end
    local char = lp.Character
    if char then
        local hum = char:FindFirstChildOfClass("Humanoid")
        if hum then hum.PlatformStand = false end
    end
    flyActive = false
end

-- ══════════════════════════════════════════════
--  ESP
-- ══════════════════════════════════════════════
local espRoot = Instance.new("Folder", game.CoreGui)
espRoot.Name  = "MM2_ESP"

local function buildESP(plr)
    if not plr.Character then return end
    local hrp = plr.Character:FindFirstChild("HumanoidRootPart")
    if not hrp then return end
    local bb  = Instance.new("BillboardGui")
    bb.Name           = "ESP_BB"
    bb.AlwaysOnTop    = true
    bb.Size           = UDim2.new(0,120,0,36)
    bb.StudsOffset    = Vector3.new(0,3.2,0)
    bb.Adornee        = hrp
    bb.Parent         = espRoot
    local lbl = Instance.new("TextLabel", bb)
    lbl.Size                 = UDim2.new(1,0,1,0)
    lbl.BackgroundTransparency = 1
    lbl.TextStrokeTransparency = 0
    lbl.TextScaled           = true
    lbl.Font                 = Enum.Font.GothamBold
    lbl.Name                 = "Label"
    return lbl
end

local espColors = {
    Murderer = Color3.fromRGB(255,55,55),
    Sheriff  = Color3.fromRGB(80,160,255),
    Innocent = Color3.fromRGB(80,255,120),
}

task.spawn(function()
    while true do
        task.wait(1.5)
        if not cfg.esp then espRoot:ClearAllChildren(); continue end
        espRoot:ClearAllChildren()
        for _, plr in ipairs(Players:GetPlayers()) do
            if plr == lp then continue end
            local lbl = buildESP(plr)
            if lbl then
                local role  = getRole(plr)
                local color = espColors[role] or espColors.Innocent
                lbl.TextColor3 = color
                lbl.Text = string.format("[%s] %s", role, plr.Name)
            end
        end
    end
end)

-- ══════════════════════════════════════════════
--  COIN FARM
-- ══════════════════════════════════════════════
task.spawn(function()
    while true do
        task.wait(0.25)
        if not cfg.coinFarm then continue end
        local char = lp.Character
        if not char then continue end
        local root = char:FindFirstChild("HumanoidRootPart")
        if not root then continue end
        for _, obj in ipairs(workspace:GetDescendants()) do
            if obj.Name == "Coin" and obj:IsA("BasePart") then
                root.CFrame = CFrame.new(obj.Position)
                task.wait(0.04)
            end
        end
    end
end)

-- ══════════════════════════════════════════════
--  KEYBINDS
-- ══════════════════════════════════════════════
UIS.InputBegan:Connect(function(inp, gpe)
    if gpe then return end
    local k = inp.KeyCode
    if k == cfg.keys.fly then
        cfg.fly = not cfg.fly
        if cfg.fly then startFly() else stopFly() end
    elseif k == cfg.keys.killAura then
        cfg.killAura = not cfg.killAura
    elseif k == cfg.keys.silentAim then
        cfg.silentAim = not cfg.silentAim
    elseif k == cfg.keys.autoThrow then
        cfg.autoThrow = not cfg.autoThrow
    elseif k == cfg.keys.coinFarm then
        cfg.coinFarm = not cfg.coinFarm
    end
end)

-- ══════════════════════════════════════════════
--  GUI — YUMEKO BACKGROUND
-- ══════════════════════════════════════════════
local sg = Instance.new("ScreenGui")
sg.Name              = "MM2_GUI"
sg.ResetOnSpawn      = false
sg.ZIndexBehavior    = Enum.ZIndexBehavior.Sibling
sg.IgnoreGuiInset    = true
sg.Parent            = gethui and gethui() or game.CoreGui

-- janela principal
local win = Instance.new("Frame", sg)
win.Name             = "Window"
win.Size             = UDim2.new(0,420,0,500)
win.Position         = UDim2.new(0.5,-210,0.5,-250)
win.BackgroundColor3 = Color3.fromRGB(8,8,12)
win.BorderSizePixel  = 0
win.ClipDescendants  = true

local corner = Instance.new("UICorner", win)
corner.CornerRadius  = UDim.new(0,10)

-- glow border
local stroke = Instance.new("UIStroke", win)
stroke.Color     = Color3.fromRGB(180,30,30)
stroke.Thickness = 1.5
stroke.Transparency = 0.3

-- fundo Yumeko
local bg = Instance.new("ImageLabel", win)
bg.Size              = UDim2.new(1,0,1,0)
bg.Position          = UDim2.new(0,0,0,0)
bg.Image             = IMAGE_ID
bg.ImageTransparency = 0.45
bg.BackgroundTransparency = 1
bg.ScaleType         = Enum.ScaleType.Crop
bg.ZIndex            = 1

-- overlay escuro para legibilidade
local overlay = Instance.new("Frame", win)
overlay.Size             = UDim2.new(1,0,1,0)
overlay.BackgroundColor3 = Color3.fromRGB(0,0,0)
overlay.BackgroundTransparency = 0.55
overlay.BorderSizePixel  = 0
overlay.ZIndex           = 2

-- título
local titleBar = Instance.new("Frame", win)
titleBar.Size             = UDim2.new(1,0,0,36)
titleBar.BackgroundColor3 = Color3.fromRGB(140,15,15)
titleBar.BackgroundTransparency = 0.1
titleBar.BorderSizePixel  = 0
titleBar.ZIndex           = 3

local titleLbl = Instance.new("TextLabel", titleBar)
titleLbl.Size            = UDim2.new(1,-40,1,0)
titleLbl.Position        = UDim2.new(0,12,0,0)
titleLbl.BackgroundTransparency = 1
titleLbl.Text            = "⚔  MM2 | by ANON"
titleLbl.TextColor3      = Color3.fromRGB(255,220,220)
titleLbl.Font            = Enum.Font.GothamBold
titleLbl.TextSize        = 15
titleLbl.TextXAlignment  = Enum.TextXAlignment.Left
titleLbl.ZIndex          = 4

-- drag
local dragging, dragStart, startPos
titleBar.InputBegan:Connect(function(inp)
    if inp.UserInputType == Enum.UserInputType.MouseButton1 then
        dragging  = true
        dragStart = inp.Position
        startPos  = win.Position
    end
end)
UIS.InputChanged:Connect(function(inp)
    if dragging and inp.UserInputType == Enum.UserInputType.MouseMovement then
        local delta = inp.Position - dragStart
        win.Position = UDim2.new(
            startPos.X.Scale, startPos.X.Offset + delta.X,
            startPos.Y.Scale, startPos.Y.Offset + delta.Y
        )
    end
end)
UIS.InputEnded:Connect(function(inp)
    if inp.UserInputType == Enum.UserInputType.MouseButton1 then
        dragging = false
    end
end)

-- minimizar
local minBtn = Instance.new("TextButton", titleBar)
minBtn.Size             = UDim2.new(0,30,0,24)
minBtn.Position         = UDim2.new(1,-34,0,6)
minBtn.BackgroundColor3 = Color3.fromRGB(60,0,0)
minBtn.Text             = "—"
minBtn.TextColor3       = Color3.fromRGB(255,200,200)
minBtn.Font             = Enum.Font.GothamBold
minBtn.TextSize         = 13
minBtn.BorderSizePixel  = 0
minBtn.ZIndex           = 5
Instance.new("UICorner", minBtn).CornerRadius = UDim.new(0,4)

local minimized = false
local contentFrame
minBtn.MouseButton1Click:Connect(function()
    minimized = not minimized
    tween(win, {Size = minimized
        and UDim2.new(0,420,0,36)
        or  UDim2.new(0,420,0,500)
    }, 0.25)
    minBtn.Text = minimized and "□" or "—"
end)

-- conteúdo
contentFrame = Instance.new("Frame", win)
contentFrame.Name            = "Content"
contentFrame.Size            = UDim2.new(1,0,1,-36)
contentFrame.Position        = UDim2.new(0,0,0,36)
contentFrame.BackgroundTransparency = 1
contentFrame.ZIndex          = 3

-- scroll
local scroll = Instance.new("ScrollingFrame", contentFrame)
scroll.Size                  = UDim2.new(1,0,1,0)
scroll.BackgroundTransparency = 1
scroll.BorderSizePixel       = 0
scroll.ScrollBarThickness    = 3
scroll.ScrollBarImageColor3  = Color3.fromRGB(180,30,30)
scroll.CanvasSize            = UDim2.new(0,0,0,0)
scroll.AutomaticCanvasSize   = Enum.AutomaticSize.Y
scroll.ZIndex                = 3

local list = Instance.new("UIListLayout", scroll)
list.SortOrder   = Enum.SortOrder.LayoutOrder
list.Padding     = UDim.new(0,4)

local pad = Instance.new("UIPadding", scroll)
pad.PaddingLeft  = UDim.new(0,10)
pad.PaddingRight = UDim.new(0,10)
pad.PaddingTop   = UDim.new(0,8)

-- ══════════════════════════════
--  helpers de UI
-- ══════════════════════════════
local function makeSection(label)
    local f = Instance.new("Frame", scroll)
    f.Size                   = UDim2.new(1,0,0,22)
    f.BackgroundTransparency = 1
    f.BorderSizePixel        = 0
    f.LayoutOrder            = 0
    local t = Instance.new("TextLabel", f)
    t.Size                   = UDim2.new(1,0,1,0)
    t.BackgroundTransparency = 1
    t.Text                   = "── " .. label .. " ──"
    t.TextColor3             = Color3.fromRGB(200,60,60)
    t.Font                   = Enum.Font.GothamBold
    t.TextSize               = 11
    t.TextXAlignment         = Enum.TextXAlignment.Left
    t.ZIndex                 = 4
end

local toggleCount = 0
local function makeToggle(labelText, state, keybind, onToggle)
    toggleCount += 1
    local row = Instance.new("Frame", scroll)
    row.Size                   = UDim2.new(1,0,0,36)
    row.BackgroundColor3       = Color3.fromRGB(18,8,8)
    row.BackgroundTransparency = 0.3
    row.BorderSizePixel        = 0
    row.LayoutOrder            = toggleCount
    Instance.new("UICorner", row).CornerRadius = UDim.new(0,6)

    local lbl = Instance.new("TextLabel", row)
    lbl.Size                 = UDim2.new(1,-90,1,0)
    lbl.Position             = UDim2.new(0,10,0,0)
    lbl.BackgroundTransparency = 1
    lbl.Text                 = labelText
    lbl.TextColor3           = Color3.fromRGB(230,210,210)
    lbl.Font                 = Enum.Font.Gotham
    lbl.TextSize             = 13
    lbl.TextXAlignment       = Enum.TextXAlignment.Left
    lbl.ZIndex               = 4

    local kb = Instance.new("TextLabel", row)
    kb.Size                  = UDim2.new(0,38,0,20)
    kb.Position              = UDim2.new(1,-82,0.5,-10)
    kb.BackgroundColor3      = Color3.fromRGB(40,10,10)
    kb.BackgroundTransparency = 0.1
    kb.Text                  = "[" .. (keybind and keybind.Name:sub(8) or "—") .. "]"
    kb.TextColor3            = Color3.fromRGB(180,100,100)
    kb.Font                  = Enum.Font.Code
    kb.TextSize              = 10
    kb.ZIndex                = 4
    Instance.new("UICorner", kb).CornerRadius = UDim.new(0,4)

    local btn = Instance.new("TextButton", row)
    btn.Size                 = UDim2.new(0,36,0,22)
    btn.Position             = UDim2.new(1,-42,0.5,-11)
    btn.BackgroundColor3     = state
        and Color3.fromRGB(180,30,30)
        or  Color3.fromRGB(40,40,40)
    btn.Text                 = state and "ON" or "OFF"
    btn.TextColor3           = Color3.fromRGB(255,255,255)
    btn.Font                 = Enum.Font.GothamBold
    btn.TextSize             = 12
    btn.BorderSizePixel      = 0
    btn.ZIndex               = 5
    Instance.new("UICorner", btn).CornerRadius = UDim.new(0,5)

    local active = state
    btn.MouseButton1Click:Connect(function()
        active = not active
        btn.Text = active and "ON" or "OFF"
        tween(btn, {BackgroundColor3 = active
            and Color3.fromRGB(180,30,30)
            or  Color3.fromRGB(40,40,40)
        })
        onToggle(active)
    end)

    return btn
end

-- ══════════════════════════════
--  SEÇÕES
-- ══════════════════════════════
makeSection("COMBAT")
makeToggle("Silent Aim",   cfg.silentAim, cfg.keys.silentAim, function(v) cfg.silentAim = v end)
makeToggle("Kill Aura",    cfg.killAura,  cfg.keys.killAura,  function(v) cfg.killAura  = v end)
makeToggle("Auto Throw",   cfg.autoThrow, cfg.keys.autoThrow, function(v) cfg.autoThrow = v end)

makeSection("MOVEMENT")
makeToggle("Speed Hack",   cfg.speed,    nil,              function(v) cfg.speed = v; applySpeed() end)
makeToggle("Fly  [F]",     cfg.fly,      cfg.keys.fly,     function(v)
    cfg.fly = v
    if v then startFly() else stopFly() end
end)

makeSection("MISC")
makeToggle("ESP",          cfg.esp,      nil,              function(v) cfg.esp = v end)
makeToggle("Coin Farm",    cfg.coinFarm, cfg.keys.coinFarm,function(v) cfg.coinFarm = v end)
makeToggle("No Fog",       cfg.noFog,    nil,              function(v)
    cfg.noFog = v
    for _, a in ipairs(game:GetDescendants()) do
        if a:IsA("Atmosphere") then pcall(function() a.Density = v and 0 or 0.395 end) end
    end
end)

-- RShift toggle visibilidade
UIS.InputBegan:Connect(function(inp, gpe)
    if inp.KeyCode == cfg.keys.gui then
        win.Visible = not win.Visible
    end
end)

print("[MM2] carregado | RShift = mostra/esconde GUI")
