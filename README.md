-- Painel de Debug / Demo UI para Roblox Studio
-- Uso: Cole em StarterPlayerScripts como LocalScript
-- Atenção: Este é um DEMO VISUAL para fins de desenvolvimento / aprendizado.
-- Não manipula entradas do servidor nem fornece cheats; destaca apenas visualmente alvos no cliente.

local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local UserInputService = game:GetService("UserInputService")
local TweenService = game:GetService("TweenService")

local localPlayer = Players.LocalPlayer
local camera = workspace.CurrentCamera

-- CONFIGURAÇÕES VISUAIS
local PANEL_WIDTH = 300
local PANEL_HEIGHT = 260
local BG_COLOR = Color3.fromRGB(10,10,10)        -- fundo preto
local ACCENT_COLOR = Color3.fromRGB(255,255,255) -- branco para texto / toggles

-- UTIL: cria instância com propriedades
local function new(class, props)
    local obj = Instance.new(class)
    for k,v in pairs(props or {}) do obj[k] = v end
    return obj
end

-- GUI principal (ScreenGui)
local screenGui = new("ScreenGui", {Name = "DevDemoPanel", ResetOnSpawn = false, Parent = localPlayer:WaitForChild("PlayerGui")})

-- Floating button (arrastável) para minimizar/abrir
local floatBtn = new("Frame", {
    Name = "FloatButton",
    AnchorPoint = Vector2.new(0,0),
    Position = UDim2.new(0.02, 0, 0.02, 0),
    Size = UDim2.new(0, 60, 0, 60),
    BackgroundColor3 = BG_COLOR,
    BorderSizePixel = 0,
    Parent = screenGui,
    ZIndex = 5,
    ClipsDescendants = false,
})
local floatLabel = new("TextLabel", {
    Parent = floatBtn,
    Size = UDim2.new(1,0,1,0),
    BackgroundTransparency = 1,
    Text = "", -- ícone minimal (pode trocar)
    Font = Enum.Font.SourceSansBold,
    TextColor3 = ACCENT_COLOR,
    TextScaled = true,
})
-- Make it draggable
do
    local dragging = false
    local dragStart = Vector2.new()
    local startPos = UDim2.new()
    floatBtn.InputBegan:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1 then
            dragging = true
            dragStart = input.Position
            startPos = floatBtn.Position
            input.Changed:Connect(function()
                if input.UserInputState == Enum.UserInputState.End then
                    dragging = false
                end
            end)
        end
    end)
    floatBtn.InputChanged:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseMovement then
            floatBtn:GetPropertyChangedSignal("AbsolutePosition"):Wait() -- no-op to keep responsive
        end
    end)
    RunService.RenderStepped:Connect(function()
        if dragging then
            local mousePos = UserInputService:GetMouseLocation()
            local delta = mousePos - dragStart
            floatBtn.Position = UDim2.new(startPos.X.Scale, startPos.X.Offset + delta.X, startPos.Y.Scale, startPos.Y.Offset + delta.Y)
        end
    end)
end

-- Painel principal
local panel = new("Frame", {
    Name = "MainPanel",
    AnchorPoint = Vector2.new(0,0),
    Position = UDim2.new(0.02, 0, 0.02, 70),
    Size = UDim2.new(0, PANEL_WIDTH, 0, PANEL_HEIGHT),
    BackgroundColor3 = BG_COLOR,
    BorderSizePixel = 0,
    Visible = true,
    Parent = screenGui,
    ZIndex = 4,
})
local uiCorner = new("UICorner", {Parent = panel, CornerRadius = UDim.new(0,10)})
local title = new("TextLabel", {
    Parent = panel,
    Size = UDim2.new(1, -20, 0, 50),
    Position = UDim2.new(0,10,0,10),
    BackgroundTransparency = 1,
    Text = "Painel Demo • Ferramenta Dev",
    Font = Enum.Font.GothamBold,
    TextColor3 = ACCENT_COLOR,
    TextSize = 18,
    TextXAlignment = Enum.TextXAlignment.Left,
})
-- Separator
local sep = new("Frame", {Parent = panel, Size = UDim2.new(1, -20, 0, 2), Position = UDim2.new(0,10,0,60), BackgroundColor3 = Color3.fromRGB(30,30,30)})

-- Container para opções
local container = new("Frame", {
    Parent = panel,
    Size = UDim2.new(1, -20, 1, -80),
    Position = UDim2.new(0,10,0,70),
    BackgroundTransparency = 1,
})
local function makeToggle(parent, labelText, y)
    local row = new("Frame", {Parent = parent, Size = UDim2.new(1,0,0,42), Position = UDim2.new(0,0,0,(y-1)*46), BackgroundTransparency = 1})
    local lbl = new("TextLabel", {Parent = row, Size = UDim2.new(0.7,0,1,0), Position = UDim2.new(0,0,0,0), BackgroundTransparency = 1,
        Text = labelText, Font = Enum.Font.Gotham, TextColor3 = ACCENT_COLOR, TextSize = 16, TextXAlignment = Enum.TextXAlignment.Left})
    local toggle = new("TextButton", {Parent = row, Size = UDim2.new(0.28,0,0.8,0), Position = UDim2.new(0.72,0,0.1,0),
        BackgroundColor3 = Color3.fromRGB(40,40,40), Text = "OFF", Font = Enum.Font.GothamBold, TextColor3 = ACCENT_COLOR, TextSize = 14})
    local corner = new("UICorner", {Parent = toggle, CornerRadius = UDim.new(0,6)})
    local state = false
    toggle.MouseButton1Click:Connect(function()
        state = not state
        toggle.Text = state and "ON" or "OFF"
        toggle.BackgroundColor3 = state and Color3.fromRGB(70,70,70) or Color3.fromRGB(40,40,40)
        if labelText == "Aim Assist (Demo)" then
            _G.DEMO_AIM = state
        elseif labelText == "ESP - Team Check (Demo)" then
            _G.DEMO_ESP = state
        end
    end)
    return row, function() return state end
end

-- Cria toggles
local _, aimGetter = makeToggle(container, "Aim Assist (Demo)", 1)
local _, espGetter = makeToggle(container, "ESP - Team Check (Demo)", 2)

-- Sensibilidade slider (visual only)
local sensLabel = new("TextLabel", {Parent = container, Size = UDim2.new(1,0,0,20), Position = UDim2.new(0,0,0,92), BackgroundTransparency = 1, Text = "Sensibilidade Demo: 0.6", Font = Enum.Font.Gotham, TextColor3 = ACCENT_COLOR, TextSize = 14, TextXAlignment = Enum.TextXAlignment.Left})
local sensValue = 0.6

local sensSlider = new("Frame", {Parent = container, Size = UDim2.new(1,0,0,16), Position = UDim2.new(0,0,0,116), BackgroundColor3 = Color3.fromRGB(30,30,30)})
new("UICorner", {Parent = sensSlider, CornerRadius = UDim.new(0,6)})
local fill = new("Frame", {Parent = sensSlider, Size = UDim2.new(sensValue,0,1,0), BackgroundColor3 = Color3.fromRGB(70,70,70)})
local handle = new("ImageButton", {Parent = sensSlider, Size = UDim2.new(0,0,1,0), Position = UDim2.new(sensValue, -8, 0, 0), BackgroundTransparency = 1, Image = "rbxasset://textures/units/whiteDot.png"})
handle.MouseButton1Down:Connect(function()
    local dragging = true
    local conn
    conn = UserInputService.InputChanged:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseMovement and dragging then
            local x = input.Position.X - sensSlider.AbsolutePosition.X
            local ratio = math.clamp(x / sensSlider.AbsoluteSize.X, 0, 1)
            sensValue = ratio
            fill.Size = UDim2.new(ratio,0,1,0)
            handle.Position = UDim2.new(ratio, -8, 0, 0)
            sensLabel.Text = string.format("Sensibilidade Demo: %.2f", sensValue)
        end
    end)
    local upConn
    upConn = UserInputService.InputEnded:Connect(function(inp)
        if inp.UserInputType == Enum.UserInputType.MouseButton1 then
            dragging = false
            upConn:Disconnect()
            conn:Disconnect()
        end
    end)
end)

-- Minimize / maximize via floating button click
floatBtn.InputBegan:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseButton1 then
        panel.Visible = not panel.Visible
    end
end)

-- CRIAÇÃO DE ESP: gera BillboardGui em personagens quando ESP ativado
local espPool = {} -- map character -> billboard
local function createESPForCharacter(char)
    if not char or not char:IsA("Model") then return end
    if espPool[char] then return end
    local head = char:FindFirstChild("Head")
    if not head then return end
    local bill = new("BillboardGui", {Parent = head, Size = UDim2.new(0,120,0,30), Adornee = head, AlwaysOnTop = true})
    bill.StudsOffset = Vector3.new(0, 1.5, 0)
    local bg = new("Frame", {Parent = bill, Size = UDim2.new(1,0,1,0), BackgroundColor3 = Color3.fromRGB(0,0,0), BackgroundTransparency = 0.35})
    new("UICorner", {Parent = bg, CornerRadius = UDim.new(0,6)})
    local txt = new("TextLabel", {Parent = bg, Size = UDim2.new(1, -6, 1, -6), Position = UDim2.new(0,3,0,3), BackgroundTransparency = 1, Text = char.Name, Font = Enum.Font.Gotham, TextColor3 = ACCENT_COLOR, TextSize = 14, TextXAlignment = Enum.TextXAlignment.Center})
    espPool[char] = bill
end
local function removeESPForCharacter(char)
    local bill = espPool[char]
    if bill and bill.Parent then bill:Destroy() end
    espPool[char] = nil
end

-- Atualiza ESP/Targeting cada frame
_G.DEMO_ESP = false
_G.DEMO_AIM = false

local function isEnemy(targetPlayer)
    if not targetPlayer or not targetPlayer.Team or not localPlayer.Team then
        -- Sem times definidos: considerar todos como "alvos" para demo
        return true
    end
    return targetPlayer.Team ~= localPlayer.Team
end

local aimCross = new("ImageLabel", {Parent = screenGui, Size = UDim2.new(0,28,0,28), Position = UDim2.new(0.5, -14, 0.5, -14), BackgroundTransparency = 1, Image = "rbxassetid://3570695787"})
new("UICorner", {Parent = aimCross, CornerRadius = UDim.new(1,14)})
local aimTarget = nil

RunService.RenderStepped:Connect(function(dt)
    -- ESP logic
    if _G.DEMO_ESP then
        for _, pl in pairs(Players:GetPlayers()) do
            if pl ~= localPlayer and pl.Character and pl.Character:FindFirstChild("Head") then
                if isEnemy(pl) then
                    createESPForCharacter(pl.Character)
                else
                    removeESPForCharacter(pl.Character)
                end
            end
        end
        -- cleanup players who left or changed
        for char, gui in pairs(espPool) do
            if not char or not char.Parent then
                removeESPForCharacter(char)
            end
        end
    else
        -- remover todos
        for char, _ in pairs(espPool) do
            removeESPForCharacter(char)
        end
    end

    -- Aim Assist Demo: encontra alvo mais próximo ao centro da tela (visualmente) e move a mira suavemente
    if _G.DEMO_AIM then
        local best, bestDist = nil, math.huge
        for _, pl in pairs(Players:GetPlayers()) do
            if pl ~= localPlayer and pl.Character and pl.Character:FindFirstChild("Head") and isEnemy(pl) then
                local head = pl.Character.Head
                local screenPos, onScreen = camera:WorldToViewportPoint(head.Position)
                if onScreen then
                    local dx = screenPos.X - (camera.ViewportSize.X/2)
                    local dy = screenPos.Y - (camera.ViewportSize.Y/2)
                    local dist = math.sqrt(dx*dx + dy*dy)
                    if dist < bestDist then
                        best = {player = pl, pos = Vector2.new(screenPos.X, screenPos.Y), dist = dist}
                        bestDist = dist
                    end
                end
            end
        end
        if best then
            aimTarget = best
            -- Sensibilidade influencia a velocidade de "arrastar" da mira visual (demo)
            local currentPos = Vector2.new(aimCross.AbsolutePosition.X + aimCross.AbsoluteSize.X/2, aimCross.AbsolutePosition.Y + aimCross.AbsoluteSize.Y/2)
            local targetPos = best.pos
            local interp = math.clamp(sensValue * 10 * dt, 0, 1) -- controlado por slider
            local newPos = currentPos:Lerp(targetPos, interp)
            aimCross.Position = UDim2.new(0, newPos.X - aimCross.AbsoluteSize.X/2, 0, newPos.Y - aimCross.AbsoluteSize.Y/2)
        else
            -- volta ao centro suavemente
            local center = Vector2.new(camera.ViewportSize.X/2, camera.ViewportSize.Y/2)
            local currentPos = Vector2.new(aimCross.AbsolutePosition.X + aimCross.AbsoluteSize.X/2, aimCross.AbsolutePosition.Y + aimCross.AbsoluteSize.Y/2)
            local newPos = currentPos:Lerp(center, math.clamp(sensValue * 4 * dt, 0,1))
            aimCross.Position = UDim2.new(0, newPos.X - aimCross.AbsoluteSize.X/2, 0, newPos.Y - aimCross.AbsoluteSize.Y/2)
            aimTarget = nil
        end
        aimCross.Visible = true
    else
        -- ocultar mira demo
        aimCross.Visible = false
    end
end)

-- Mensagem de ajuda / legendas
local footer = new("TextLabel", {Parent = panel, Size = UDim2.new(1, -20, 0, 20), Position = UDim2.new(0,10,1,-28), BackgroundTransparency = 1, Text = "Modo Demo • Use apenas em seu ambiente de desenvolvimento", Font = Enum.Font.Gotham, TextColor3 = Color3.fromRGB(180,180,180), TextSize = 12, TextXAlignment = Enum.TextXAlignment.Left})

-- Feedback visual ao clicar em um alvo (simulação apenas)
UserInputService.InputBegan:Connect(function(i,gp)
    if gp then return end
    if i.UserInputType == Enum.UserInputType.MouseButton1 and aimTarget and _G.DEMO_AIM then
        -- efeito visual de "foco" no alvo — apenas UI
        local p = aimTarget.player
        if p and p.Character and p.Character:FindFirstChild("Head") then
            local head = p.Character.Head
            local flash = Instance.new("SelectionBox")
            flash.Adornee = head
            flash.Color3 = Color3.fromRGB(255, 180, 180)
            flash.LineThickness = 0.02
            flash.Parent = head
            delay(0.5, function()
                if flash and flash.Parent then flash:Destroy() end
            end)
        end
    end
end)

-- Certifique que ao sair o GUI se limpe
localPlayer.AncestryChanged:Connect(function()
    if not localPlayer:IsDescendantOf(game) then
        screenGui:Destroy()
    end
end)

-- FIM DO SCRIPT
