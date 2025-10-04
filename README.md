--[[
Painel Dev/Admin (LocalScript)
Colocar em: StarterPlayer > StarterPlayerScripts
Uso: Ferramenta de desenvolvimento/admin para seu jogo (servidores privados).
Funcionalidades:
 - Botão flutuante arrastável (minimiza/abre painel)
 - Painel com botões visíveis: Aim Assist (VISUAL), ESP (Billboard) com Team Check
 - Slider de sensibilidade para Aim Assist
 - Painel de erros/logs (append)
 - Tudo CLIENT-ONLY: efeitos visuais e debug. Não automatiza ataques nem envia comandos ao servidor.
--]]

local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local UserInputService = game:GetService("UserInputService")
local Debris = game:GetService("Debris")

local localPlayer = Players.LocalPlayer
local camera = workspace.CurrentCamera

-- VISUAL CONFIG
local PANEL_W, PANEL_H = 360, 300
local BG = Color3.fromRGB(8,8,8)
local FG = Color3.fromRGB(245,245,245)
local ACC = Color3.fromRGB(230,230,230)
local ACCENT_ON = Color3.fromRGB(100,100,100)
local ACCENT_OFF = Color3.fromRGB(40,40,40)

-- helper
local function new(class, props)
    local obj = Instance.new(class)
    if props then for k,v in pairs(props) do obj[k] = v end end
    return obj
end

-- create GUI root
local playerGui = localPlayer:WaitForChild("PlayerGui")
local screenGui = new("ScreenGui", {Name = "DevAdminPanelGUI", Parent = playerGui, ResetOnSpawn = false})

-- FLOAT BUTTON (arrastável)
local floatBtn = new("Frame", {
    Parent = screenGui,
    Name = "FloatBtn",
    Size = UDim2.new(0,64,0,64),
    Position = UDim2.new(0.02,0,0.02,0),
    BackgroundColor3 = BG,
    BorderSizePixel = 0,
    ZIndex = 50,
})
new("UICorner",{Parent=floatBtn, CornerRadius = UDim.new(0,12)})
local floatLabel = new("TextLabel", {
    Parent = floatBtn,
    Size = UDim2.new(1,0,1,0),
    BackgroundTransparency = 1,
    Text = "DEV",
    Font = Enum.Font.GothamBold,
    TextColor3 = FG,
    TextScaled = true,
})
-- drag variables
do
    local dragging = false
    local dragStart, startPos
    floatBtn.InputBegan:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1 then
            dragging = true
            dragStart = input.Position
            startPos = floatBtn.Position
            input.Changed:Connect(function()
                if input.UserInputState == Enum.UserInputState.End then dragging = false end
            end)
        end
    end)
    RunService.RenderStepped:Connect(function()
        if dragging then
            local mouse = UserInputService:GetMouseLocation()
            local delta = mouse - dragStart
            floatBtn.Position = UDim2.new(startPos.X.Scale, startPos.X.Offset + delta.X, startPos.Y.Scale, startPos.Y.Offset + delta.Y)
        end
    end)
end

-- MAIN PANEL
local panel = new("Frame", {
    Parent = screenGui,
    Name = "MainPanel",
    Size = UDim2.new(0, PANEL_W, 0, PANEL_H),
    Position = UDim2.new(0, 20, 0, 100),
    BackgroundColor3 = BG,
    BorderSizePixel = 0,
    ZIndex = 40,
})
new("UICorner",{Parent = panel, CornerRadius = UDim.new(0,14)})
-- Title
local title = new("TextLabel", {
    Parent = panel,
    Size = UDim2.new(1,-20,0,48),
    Position = UDim2.new(0,10,0,8),
    BackgroundTransparency = 1,
    Text = "Painel Dev • Ferramenta (Autorizado)",
    Font = Enum.Font.GothamBold,
    TextColor3 = ACC,
    TextSize = 18,
    TextXAlignment = Enum.TextXAlignment.Left,
})
-- separator
new("Frame",{Parent = panel, Size = UDim2.new(1,-20,0,2), Position = UDim2.new(0,10,0,56), BackgroundColor3 = Color3.fromRGB(30,30,30)})

-- container for controls
local container = new("Frame", {Parent = panel, Size = UDim2.new(1,-20,1,-92), Position = UDim2.new(0,10,0,66), BackgroundTransparency = 1})

-- Toggle factory
local function makeToggle(parent, y, text)
    local row = new("Frame", {Parent = parent, Size = UDim2.new(1,0,0,44), Position = UDim2.new(0,0,0,(y-1)*48), BackgroundTransparency = 1})
    local lbl = new("TextLabel", {Parent = row, Size = UDim2.new(0.68,0,1,0), BackgroundTransparency = 1, Text=text, Font=Enum.Font.Gotham, TextColor3=FG, TextSize=16, TextXAlignment=Enum.TextXAlignment.Left})
    local btn = new("TextButton", {Parent = row, Size = UDim2.new(0.3,0,0.7,0), Position = UDim2.new(0.70,0,0.15,0), BackgroundColor3=ACCENT_OFF, Text="OFF", Font=Enum.Font.GothamBold, TextColor3=FG, TextSize=14})
    new("UICorner",{Parent=btn, CornerRadius = UDim.new(0,8)})
    local state = false
    btn.MouseButton1Click:Connect(function()
        state = not state
        btn.Text = state and "ON" or "OFF"
        btn.BackgroundColor3 = state and ACCENT_ON or ACCENT_OFF
    end)
    return function() return state end, function(v) state = v; btn.Text = state and "ON" or "OFF"; btn.BackgroundColor3 = state and ACCENT_ON or ACCENT_OFF end
end

local getAimState, setAimState = makeToggle(container, 1, "Aim Assist (VISUAL)")
local getESPState, setESPState = makeToggle(container, 2, "ESP - Team Check")

-- Slider (sensibilidade)
local sensLabel = new("TextLabel", {Parent = container, Size = UDim2.new(1,0,0,18), Position = UDim2.new(0,0,0,98), BackgroundTransparency = 1, Text="Sensibilidade: 0.60", Font=Enum.Font.Gotham, TextColor3=FG, TextSize=14, TextXAlignment=Enum.TextXAlignment.Left})
local slider = new("Frame", {Parent = container, Size = UDim2.new(1,0,0,18), Position = UDim2.new(0,0,0,120), BackgroundColor3=Color3.fromRGB(28,28,28)})
new("UICorner",{Parent=slider, CornerRadius = UDim.new(0,6)})
local fill = new("Frame", {Parent = slider, Size=UDim2.new(0.6,0,1,0), BackgroundColor3=Color3.fromRGB(80,80,80)})
local handle = new("ImageButton", {Parent = slider, Size = UDim2.new(0,0,1,0), Position = UDim2.new(0.6,-8,0,0), BackgroundTransparency = 1, Image = "rbxasset://textures/units/whiteDot.png"})
local sens = 0.6
handle.MouseButton1Down:Connect(function()
    local dragging = true
    local conn
    conn = UserInputService.InputChanged:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseMovement and dragging then
            local x = input.Position.X - slider.AbsolutePosition.X
            local ratio = math.clamp(x / slider.AbsoluteSize.X, 0, 1)
            sens = ratio
            fill.Size = UDim2.new(ratio,0,1,0)
            handle.Position = UDim2.new(ratio,-8,0,0)
            sensLabel.Text = string.format("Sensibilidade: %.2f", sens)
        end
    end)
    local up
    up = UserInputService.InputEnded:Connect(function(i)
        if i.UserInputType == Enum.UserInputType.MouseButton1 then
            dragging = false
            conn:Disconnect()
            up:Disconnect()
        end
    end)
end)

-- Error/log panel (scrollable)
local logFrame = new("Frame", {Parent = panel, Size = UDim2.new(1,-20,0,60), Position = UDim2.new(0,10,1,-76), BackgroundColor3 = Color3.fromRGB(15,15,15)})
new("UICorner",{Parent=logFrame, CornerRadius = UDim.new(0,8)})
local logLabel = new("TextLabel", {Parent = logFrame, Size = UDim2.new(1,-8,1,-8), Position = UDim2.new(0,4,0,4), BackgroundTransparency = 1, Text="Logs:", Font=Enum.Font.GothamBold, TextColor3=Color3.fromRGB(200,200,200), TextSize=14, TextXAlignment = Enum.TextXAlignment.Left})
local logList = new("ScrollingFrame", {Parent = logFrame, Size = UDim2.new(1,-12,1,-28), Position = UDim2.new(0,6,0,22), BackgroundTransparency = 1, CanvasSize = UDim2.new(0,0,0,0)})
new("UIListLayout",{Parent=logList, Padding = UDim.new(0,4)})
logList.VerticalScrollBarInset = Enum.ScrollBarInset.ScrollBar

local function addLog(text)
    local t = new("TextLabel", {Parent = logList, Size = UDim2.new(1,0,0,18), BackgroundTransparency = 1, Text = text, Font = Enum.Font.Gotham, TextColor3 = Color3.fromRGB(210,210,210), TextSize = 14, TextXAlignment = Enum.TextXAlignment.Left})
    -- update canvas size
    local layout = logList:FindFirstChildOfClass("UIListLayout")
    logList.CanvasSize = UDim2.new(0,0,0, layout.AbsoluteContentSize.Y + 6)
end

-- minimize toggle via floatBtn click
floatBtn.InputBegan:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseButton1 then
        panel.Visible = not panel.Visible
    end
end)

-- ESP implementation (BillboardGui)
local espMap = {}
local function createESPForCharacter(char)
    if not char or espMap[char] then return end
    local head = char:FindFirstChild("Head")
    if not head then return end
    local bill = new("BillboardGui", {Parent = head, Adornee = head, Size = UDim2.new(0,160,0,36), AlwaysOnTop = true})
    bill.StudsOffset = Vector3.new(0,1.6,0)
    local bg = new("Frame", {Parent = bill, Size = UDim2.new(1,0,1,0), BackgroundColor3 = Color3.fromRGB(0,0,0), BackgroundTransparency = 0.25})
    new("UICorner",{Parent = bg, CornerRadius = UDim.new(0,6)})
    local nameLabel = new("TextLabel", {Parent = bg, Size = UDim2.new(1,-6,1,-6), Position = UDim2.new(0,3,0,3), BackgroundTransparency = 1, Text = char.Name, Font = Enum.Font.GothamBold, TextColor3 = FG, TextSize = 14, TextXAlignment = Enum.TextXAlignment.Center})
    -- optional: distance label
    local distLabel = new("TextLabel", {Parent = bg, Size = UDim2.new(1,-6,0,14), Position = UDim2.new(0,3,1,-18), BackgroundTransparency = 1, Text = "", Font = Enum.Font.Gotham, TextColor3 = Color3.fromRGB(200,200,200), TextSize = 12, TextXAlignment = Enum.TextXAlignment.Center})
    espMap[char] = {gui = bill, distLabel = distLabel}
end

local function removeESPForCharacter(char)
    local data = espMap[char]
    if data and data.gui and data.gui.Parent then
        data.gui:Destroy()
    end
    espMap[char] = nil
end

-- decide se é inimigo (team check) — se não houver times definidos, considera true (útil para demo)
local function isEnemy(pl)
    if not pl or pl == localPlayer then return false end
    if pl.Team and localPlayer.Team then
        return pl.Team ~= localPlayer.Team
    end
    return true
end

-- Aim Assist (visual) - move uma mira para alvo mais próximo da tela. NÃO dispara.
local aimUI = new("ImageLabel", {Parent = screenGui, Size = UDim2.new(0,28,0,28), Position = UDim2.new(0.5,-14,0.5,-14), BackgroundTransparency = 1, Image = "rbxassetid://3570695787", Visible = false, ZIndex = 60})
new("UICorner",{Parent=aimUI, CornerRadius = UDim.new(1,14)})
local currentTarget = nil

-- frame loop
RunService.RenderStepped:Connect(function(dt)
    -- ESP logic
    if getESPState() then
        for _, pl in pairs(Players:GetPlayers()) do
            if pl.Character and pl.Character:FindFirstChild("Head") and isEnemy(pl) then
                createESPForCharacter(pl.Character)
                -- update distance
                local head = pl.Character.Head
                local data = espMap[pl.Character]
                if data and data.distLabel then
                    local dist = (localPlayer.Character and localPlayer.Character:FindFirstChild("HumanoidRootPart") and (localPlayer.Character.HumanoidRootPart.Position - head.Position).Magnitude) or 0
                    data.distLabel.Text = string.format("%.0f studs", dist)
                end
            else
                if pl.Character then removeESPForCharacter(pl.Character) end
            end
        end
        -- cleanup orphaned
        for char, _ in pairs(espMap) do
            if not char or not char.Parent then removeESPForCharacter(char) end
        end
    else
        -- remove all
        for char,_ in pairs(espMap) do removeESPForCharacter(char) end
    end

    -- AIM (visual)
    if getAimState() then
        aimUI.Visible = true
        local best, bestDist = nil, math.huge
        for _, pl in pairs(Players:GetPlayers()) do
            if pl.Character and pl.Character:FindFirstChild("Head") and isEnemy(pl) then
                local head = pl.Character.Head
                local screenPos, onScreen = camera:WorldToViewportPoint(head.Position)
                if onScreen then
                    local dx = screenPos.X - (camera.ViewportSize.X/2)
                    local dy = screenPos.Y - (camera.ViewportSize.Y/2)
                    local dist = math.sqrt(dx*dx + dy*dy)
                    if dist < bestDist then
                        bestDist = dist
                        best = {player = pl, pos = Vector2.new(screenPos.X, screenPos.Y)}
                    end
                end
            end
        end
        if best then
            currentTarget = best
            -- move aimUI toward target with sensitivity
            local center = Vector2.new(aimUI.AbsolutePosition.X + aimUI.AbsoluteSize.X/2, aimUI.AbsolutePosition.Y + aimUI.AbsoluteSize.Y/2)
            local tweenTo = best.pos
            local lerp = math.clamp(sens * 8 * dt, 0, 1)
            local newPos = center:Lerp(tweenTo, lerp)
            aimUI.Position = UDim2.new(0, newPos.X - aimUI.AbsoluteSize.X/2, 0, newPos.Y - aimUI.AbsoluteSize.Y/2)
        else
            -- return to center
            local center = Vector2.new(camera.ViewportSize.X/2, camera.ViewportSize.Y/2)
            local cur = Vector2.new(aimUI.AbsolutePosition.X + aimUI.AbsoluteSize.X/2, aimUI.AbsolutePosition.Y + aimUI.AbsoluteSize.Y/2)
            local newPos = cur:Lerp(center, math.clamp(sens * 4 * dt, 0, 1))
            aimUI.Position = UDim2.new(0, newPos.X - aimUI.AbsoluteSize.X/2, 0, newPos.Y - aimUI.AbsoluteSize.Y/2)
            currentTarget = nil
        end
    else
        aimUI.Visible = false
        currentTarget = nil
    end
end)

-- click feedback (visual highlight) — simulação apenas
UserInputService.InputBegan:Connect(function(input, gameProcessed)
    if gameProcessed then return end
    if input.UserInputType == Enum.UserInputType.MouseButton1 and currentTarget and getAimState() then
        local p = currentTarget.player
        if p and p.Character and p.Character:FindFirstChild("Head") then
            local sel = Instance.new("SelectionBox")
            sel.Adornee = p.Character.Head
            sel.Color3 = Color3.fromRGB(255,120,120)
            sel.LineThickness = 0.03
            sel.Parent = p.Character.Head
            Debris:AddItem(sel, 0.6)
            addLog("Highlight aplicado em: "..p.Name)
        end
    end
end)

-- Player cleanup
Players.PlayerRemoving:Connect(function(pl)
    if pl.Character then removeESPForCharacter(pl.Character) end
end)
Players.PlayerAdded:Connect(function(pl)
    pl.CharacterAdded:Connect(function(c)
        if not getESPState() then removeESPForCharacter(c) end
    end)
end)

-- init OFF
setAimState(false)
setESPState(false)
addLog("Script carregado com sucesso. Painel pronto.")

-- Optional: command keyboard shortcuts
UserInputService.InputBegan:Connect(function(input, gameProcessed)
    if gameProcessed then return end
    if input.KeyCode == Enum.KeyCode.F1 then
        setAimState(not getAimState())
        addLog("Aim Assist: "..(getAimState() and "ON" or "OFF"))
    elseif input.KeyCode == Enum.KeyCode.F2 then
        setESPState(not getESPState())
        addLog("ESP: "..(getESPState() and "ON" or "OFF"))
    elseif input.KeyCode == Enum.KeyCode.F3 then
        panel.Visible = not panel.Visible
        addLog("Painel: "..(panel.Visible and "Visível" or "Minimizado"))
    end
end)

-- safety print
print("[DevAdminPanel] Iniciado. Use apenas em servidores privados/ambiente de desenvolvimento. Script não automatiza ações de servidor.")

-- FIM
