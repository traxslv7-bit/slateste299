-- Painel Dev (Aimbot Demo + ESP com Team Check)
-- Coloque como LocalScript em StarterPlayer > StarterPlayerScripts
-- Uso: apenas em jogos/servidores onde você tem AUTORIZAÇÃO (ambiente de desenvolvimento)
-- Este script implementa: painel arrastável, botão flutuante, toggles funcionais, slider de sensibilidade,
-- ESP visual (BillboardGui) com checagem de time e "Aim Assist" visual (mira que se move suavemente).
-- NÃO realiza ações de servidor (disparos automatizados / teleports) — apenas UI e efeitos locais.

local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local UserInputService = game:GetService("UserInputService")

local localPlayer = Players.LocalPlayer
local camera = workspace.CurrentCamera

-- === CONFIG VISUAL ===
local PANEL_W, PANEL_H = 340, 280
local BG = Color3.fromRGB(8,8,8)        -- fundo preto
local FG = Color3.fromRGB(240,240,240)  -- cor branca para texto
local ACCENT = Color3.fromRGB(255,255,255)

-- UTIL
local function new(class, props)
    local obj = Instance.new(class)
    if props then
        for k,v in pairs(props) do obj[k] = v end
    end
    return obj
end

-- Gera ScreenGui
local screenGui = new("ScreenGui", {Name = "DevTool_Panel", ResetOnSpawn = false, Parent = localPlayer:WaitForChild("PlayerGui")})

-- === FLOAT BUTTON (arrastável) ===
local float = new("Frame", {
    Parent = screenGui,
    Name = "FloatBtn",
    Size = UDim2.new(0,64,0,64),
    Position = UDim2.new(0.02,0,0.02,0),
    BackgroundColor3 = BG,
    BorderSizePixel = 0,
    ZIndex = 10,
})
new("UICorner",{Parent=float, CornerRadius = UDim.new(0,10)})
local icon = new("TextLabel", {
    Parent = float,
    Size = UDim2.new(1,0,1,0),
    BackgroundTransparency = 1,
    Text = "DEV",
    Font = Enum.Font.GothamBold,
    TextColor3 = FG,
    TextScaled = true,
})
-- Dragging
do
    local dragging, dragStart, startPos
    float.InputBegan:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1 then
            dragging = true
            dragStart = input.Position
            startPos = float.Position
            input.Changed:Connect(function()
                if input.UserInputState == Enum.UserInputState.End then dragging = false end
            end)
        end
    end)
    RunService.RenderStepped:Connect(function()
        if dragging then
            local mouseLoc = UserInputService:GetMouseLocation()
            local delta = mouseLoc - dragStart
            float.Position = UDim2.new(startPos.X.Scale, startPos.X.Offset + delta.X, startPos.Y.Scale, startPos.Y.Offset + delta.Y)
        end
    end)
end

-- === MAIN PANEL ===
local panel = new("Frame", {
    Parent = screenGui,
    Name = "MainPanel",
    Size = UDim2.new(0, PANEL_W, 0, PANEL_H),
    Position = UDim2.new(0, 20 + 0.02 * 1920, 0, 100), -- fallback pos; user can move float btn
    BackgroundColor3 = BG,
    BorderSizePixel = 0,
    Visible = true,
    ZIndex = 9,
})
new("UICorner",{Parent=panel, CornerRadius = UDim.new(0,12)})
local title = new("TextLabel", {
    Parent = panel,
    Size = UDim2.new(1,-20,0,48),
    Position = UDim2.new(0,10,0,6),
    BackgroundTransparency = 1,
    Text = "Painel Dev • Demo (Autorizado)",
    Font = Enum.Font.GothamBold,
    TextColor3 = ACCENT,
    TextSize = 18,
    TextXAlignment = Enum.TextXAlignment.Left,
})
local sep = new("Frame", {Parent = panel, Size = UDim2.new(1,-20,0,2), Position = UDim2.new(0,10,0,52), BackgroundColor3 = Color3.fromRGB(30,30,30)})

-- Container para controles
local container = new("Frame", {Parent = panel, Size = UDim2.new(1,-20,1,-86), Position = UDim2.new(0,10,0,62), BackgroundTransparency = 1})

-- Toggle helper (retorna função para obter estado)
local function makeToggle(parent, y, text)
    local row = new("Frame", {Parent=parent, Size=UDim2.new(1,0,0,44), Position=UDim2.new(0,0,0,(y-1)*48), BackgroundTransparency=1})
    local lbl = new("TextLabel", {Parent=row, Size=UDim2.new(0.7,0,1,0), BackgroundTransparency=1, Text=text, Font=Enum.Font.Gotham, TextColor3=FG, TextSize=16, TextXAlignment=Enum.TextXAlignment.Left})
    local btn = new("TextButton", {Parent=row, Size=UDim2.new(0.28,0,0.7,0), Position=UDim2.new(0.72,0,0.15,0), BackgroundColor3=Color3.fromRGB(45,45,45), Text="OFF", Font=Enum.Font.GothamBold, TextColor3=FG, TextSize=14})
    new("UICorner",{Parent=btn, CornerRadius=UDim.new(0,8)})
    local state = false
    btn.MouseButton1Click:Connect(function()
        state = not state
        btn.Text = state and "ON" or "OFF"
        btn.BackgroundColor3 = state and Color3.fromRGB(80,80,80) or Color3.fromRGB(45,45,45)
    end)
    return function() return state end, function(v) state = v; btn.Text = state and "ON" or "OFF"; btn.BackgroundColor3 = state and Color3.fromRGB(80,80,80) or Color3.fromRGB(45,45,45) end
end

local getAimState, setAimState = makeToggle(container, 1, "Aim Assist (Demo)")
local getESPState, setESPState = makeToggle(container, 2, "ESP - Team Check (Demo)")

-- Sensibilidade slider
local sensLabel = new("TextLabel", {Parent = container, Size=UDim2.new(1,0,0,18), Position=UDim2.new(0,0,0,100), BackgroundTransparency=1, Text="Sensibilidade: 0.60", Font=Enum.Font.Gotham, TextColor3=FG, TextSize=14, TextXAlignment=Enum.TextXAlignment.Left})
local slider = new("Frame", {Parent = container, Size=UDim2.new(1,0,0,18), Position=UDim2.new(0,0,0,122), BackgroundColor3=Color3.fromRGB(28,28,28)})
new("UICorner",{Parent=slider, CornerRadius=UDim.new(0,6)})
local fill = new("Frame", {Parent=slider, Size=UDim2.new(0.6,0,1,0), BackgroundColor3=Color3.fromRGB(80,80,80)})
local handle = new("ImageButton", {Parent=slider, Size=UDim2.new(0,0,1,0), Position=UDim2.new(0.6,-8,0,0), BackgroundTransparency=1, Image="rbxasset://textures/units/whiteDot.png"})
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

-- Footer
local footer = new("TextLabel", {Parent=panel, Size=UDim2.new(1,-20,0,20), Position=UDim2.new(0,10,1,-28), BackgroundTransparency=1, Text="Modo Demo • Somente para seu ambiente (dev)", Font=Enum.Font.Gotham, TextColor3=Color3.fromRGB(160,160,160), TextSize=12, TextXAlignment=Enum.TextXAlignment.Left})

-- Toggle panel visível via float button click
float.InputBegan:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseButton1 then
        panel.Visible = not panel.Visible
    end
end)

-- === ESP IMPLEMENTAÇÃO (BillboardGui) ===
local espMap = {}
local function createESP(char)
    if not char or espMap[char] then return end
    local head = char:FindFirstChild("Head")
    if not head then return end
    local bill = new("BillboardGui", {Parent = head, Adornee = head, Size = UDim2.new(0,140,0,32), AlwaysOnTop = true})
    bill.StudsOffset = Vector3.new(0,1.6,0)
    local bg = new("Frame", {Parent = bill, Size = UDim2.new(1,0,1,0), BackgroundColor3 = Color3.fromRGB(0,0,0), BackgroundTransparency = 0.25})
    new("UICorner", {Parent = bg, CornerRadius = UDim.new(0,6)})
    local txt = new("TextLabel", {Parent = bg, Size = UDim2.new(1,-6,1,-6), Position = UDim2.new(0,3,0,3), BackgroundTransparency=1, Text=char.Name, Font=Enum.Font.GothamBold, TextColor3=FG, TextSize=14, TextXAlignment=Enum.TextXAlignment.Center})
    espMap[char] = bill
end
local function removeESP(char)
    local g = espMap[char]
    if g then
        if g.Parent then g:Destroy() end
        espMap[char] = nil
    end
end

-- Utility: isEnemy (considera times; se não houver times, considera true para demo)
local function isEnemy(player)
    if not player or not player:IsA("Player") then return false end
    if player == localPlayer then return false end
    if player.Team and localPlayer.Team then
        return player.Team ~= localPlayer.Team
    end
    return true
end

-- === AIM ASSIST (VISUAL) ===
local aimUI = new("ImageLabel", {Parent = screenGui, Size = UDim2.new(0,28,0,28), Position = UDim2.new(0.5,-14,0.5,-14), BackgroundTransparency = 1, Image = "rbxassetid://3570695787", Visible = false})
new("UICorner",{Parent=aimUI, CornerRadius = UDim.new(1,14)})
local currentTarget = nil

-- Frame loop: atualiza ESP e Aim
RunService.RenderStepped:Connect(function(dt)
    -- ESP
    if getESPState() then
        for _, pl in pairs(Players:GetPlayers()) do
            if pl.Character and pl.Character:FindFirstChild("Head") and isEnemy(pl) then
                createESP(pl.Character)
            else
                if pl.Character then removeESP(pl.Character) end
            end
        end
        -- cleanup
        for char, gui in pairs(espMap) do
            if not char or not char.Parent then removeESP(char) end
        end
    else
        for char, _ in pairs(espMap) do removeESP(char) end
    end

    -- AIM ASSIST (visual only)
    if getAimState() then
        aimUI.Visible = true
        local best, bestDist = nil, math.huge
        for _, pl in pairs(Players:GetPlayers()) do
            if pl.Character and pl.Character:FindFirstChild("Head") and isEnemy(pl) then
                local head = pl.Character.Head
                local pos3, onScreen = camera:WorldToViewportPoint(head.Position)
                if onScreen then
                    local dx = pos3.X - (camera.ViewportSize.X/2)
                    local dy = pos3.Y - (camera.ViewportSize.Y/2)
                    local dist = math.sqrt(dx*dx + dy*dy)
                    if dist < bestDist then
                        bestDist = dist
                        best = {player = pl, screenPos = Vector2.new(pos3.X, pos3.Y)}
                    end
                end
            end
        end
        if best then
            currentTarget = best
            -- interpola posição da mira segundo sensibilidade (sens)
            local aimCenter = Vector2.new(aimUI.AbsolutePosition.X + aimUI.AbsoluteSize.X/2, aimUI.AbsolutePosition.Y + aimUI.AbsoluteSize.Y/2)
            local newPos = aimCenter:Lerp(best.screenPos, math.clamp(sens * 8 * dt, 0, 1))
            aimUI.Position = UDim2.new(0, newPos.X - aimUI.AbsoluteSize.X/2, 0, newPos.Y - aimUI.AbsoluteSize.Y/2)
        else
            -- retorna ao centro suavemente
            local center = Vector2.new(camera.ViewportSize.X/2, camera.ViewportSize.Y/2)
            local aimCenter = Vector2.new(aimUI.AbsolutePosition.X + aimUI.AbsoluteSize.X/2, aimUI.AbsolutePosition.Y + aimUI.AbsoluteSize.Y/2)
            local newPos = aimCenter:Lerp(center, math.clamp(sens * 4 * dt, 0, 1))
            aimUI.Position = UDim2.new(0, newPos.X - aimUI.AbsoluteSize.X/2, 0, newPos.Y - aimUI.AbsoluteSize.Y/2)
            currentTarget = nil
        end
    else
        aimUI.Visible = false
        currentTarget = nil
    end
end)

-- Clique do jogador: mostrar highlight visual no alvo (demo)
UserInputService.InputBegan:Connect(function(input, gameProcessed)
    if gameProcessed then return end
    if input.UserInputType == Enum.UserInputType.MouseButton1 and currentTarget and getAimState() then
        local p = currentTarget.player
        if p and p.Character and p.Character:FindFirstChild("Head") then
            local head = p.Character.Head
            local sel = Instance.new("SelectionBox")
            sel.Adornee = head
            sel.Color3 = Color3.fromRGB(255,120,120)
            sel.LineThickness = 0.03
            sel.Parent = head
            game:GetService("Debris"):AddItem(sel, 0.6)
        end
    end
end)

-- Cleanup on player leave / character removal
Players.PlayerRemoving:Connect(function(pl)
    if pl.Character then removeESP(pl.Character) end
end)
Players.PlayerAdded:Connect(function(pl)
    pl.CharacterAdded:Connect(function(c) if not getESPState() then removeESP(c) end end)
end)

-- INITIAL STATES: desligado
setAimState(false)
setESPState(false)

-- Mensagem de segurança / instruções rápidas (imprime no output)
print("[DevTool_Panel] Script carregado. Uso seguro: apenas em servidores privados / ambiente dev. Este script NÃO automatiza disparos nem modifica o servidor.")

-- FIM
