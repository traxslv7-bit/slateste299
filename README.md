-- TrainingClient (LocalScript)
-- Mostra UI de crosshair, faz raycast no clique e envia posição para o servidor via RemoteEvent
-- Também exibe HUD com score/accuracy lidas do leaderstats.

local Players = game:GetService("Players")
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local UserInputService = game:GetService("UserInputService")
local RunService = game:GetService("RunService")

local localPlayer = Players.LocalPlayer
local camera = workspace.CurrentCamera
local event = ReplicatedStorage:WaitForChild("TrainingEvent")

-- UI
local playerGui = localPlayer:WaitForChild("PlayerGui")
local screenGui = Instance.new("ScreenGui", playerGui)
screenGui.Name = "TrainingClientGUI"
screenGui.ResetOnSpawn = false

-- Crosshair
local cross = Instance.new("Frame", screenGui)
cross.Name = "Crosshair"
cross.Size = UDim2.new(0,2,0,2)
cross.BackgroundColor3 = Color3.new(1,1,1)
cross.AnchorPoint = Vector2.new(0.5,0.5)
cross.Position = UDim2.new(0.5,0.5,0.5,0)
cross.ZIndex = 50

local crossV = Instance.new("Frame", screenGui)
crossV.Size = UDim2.new(0,12,0,2)
crossV.Position = UDim2.new(0.5,-6,0.5, -1)
crossV.BackgroundColor3 = Color3.new(1,1,1)
crossV.AnchorPoint = Vector2.new(0,0)
crossV.ZIndex = 50

local crossH = Instance.new("Frame", screenGui)
crossH.Size = UDim2.new(0,2,0,12)
crossH.Position = UDim2.new(0.5,-1,0.5,-6)
crossH.BackgroundColor3 = Color3.new(1,1,1)
crossH.AnchorPoint = Vector2.new(0,0)
crossH.ZIndex = 50

-- HUD (score / hits / shots / accuracy)
local hud = Instance.new("Frame", screenGui)
hud.Name = "HUD"
hud.Size = UDim2.new(0,200,0,80)
hud.Position = UDim2.new(0.01,0,0.01,0)
hud.BackgroundTransparency = 0.15
hud.BackgroundColor3 = Color3.fromRGB(10,10,10)
hud.BorderSizePixel = 0
hud.ZIndex = 60
local uiCorner = Instance.new("UICorner", hud)
uiCorner.CornerRadius = UDim.new(0,8)

local title = Instance.new("TextLabel", hud)
title.Size = UDim2.new(1, -12, 0, 22)
title.Position = UDim2.new(0,6,0,6)
title.BackgroundTransparency = 1
title.Text = "Treino de Mira"
title.Font = Enum.Font.GothamBold
title.TextColor3 = Color3.fromRGB(230,230,230)
title.TextSize = 16
title.TextXAlignment = Enum.TextXAlignment.Left

local scoreLabel = Instance.new("TextLabel", hud)
scoreLabel.Size = UDim2.new(1, -12, 0, 18)
scoreLabel.Position = UDim2.new(0,6,0,32)
scoreLabel.BackgroundTransparency = 1
scoreLabel.Text = "Score: 0"
scoreLabel.Font = Enum.Font.Gotham
scoreLabel.TextColor3 = Color3.fromRGB(220,220,220)
scoreLabel.TextSize = 14
scoreLabel.TextXAlignment = Enum.TextXAlignment.Left

local statsLabel = Instance.new("TextLabel", hud)
statsLabel.Size = UDim2.new(1, -12, 0, 18)
statsLabel.Position = UDim2.new(0,6,0,50)
statsLabel.BackgroundTransparency = 1
statsLabel.Text = "Hits: 0 | Shots: 0 | Accuracy: 0%"
statsLabel.Font = Enum.Font.Gotham
statsLabel.TextColor3 = Color3.fromRGB(200,200,200)
statsLabel.TextSize = 13
statsLabel.TextXAlignment = Enum.TextXAlignment.Left

-- envia clique para o servidor (pos = mouse.Hit.Position)
local function onClick()
    local mouse = localPlayer:GetMouse()
    if not mouse then return end
    local targetPos = mouse.Hit and mouse.Hit.Position
    if targetPos then
        -- envia a posição para o servidor validar
        event:FireServer({position = targetPos, timestamp = tick()})
    end
end

UserInputService.InputBegan:Connect(function(input, gameProcessed)
    if gameProcessed then return end
    if input.UserInputType == Enum.UserInputType.MouseButton1 then
        onClick()
    end
end)

-- Atualiza HUD pegando leaderstats do jogador (server cria leaderstats)
local function updateHUD()
    local ls = localPlayer:FindFirstChild("leaderstats")
    if ls then
        local score = ls:FindFirstChild("Score")
        local shots = ls:FindFirstChild("Shots")
        local hits = ls:FindFirstChild("Hits")
        if score then scoreLabel.Text = "Score: "..tostring(score.Value) end
        if shots and hits then
            local shotsV, hitsV = shots.Value, hits.Value
            local acc = 0
            if shotsV > 0 then acc = math.floor((hitsV / shotsV) * 100) end
            statsLabel.Text = string.format("Hits: %d | Shots: %d | Accuracy: %d%%", hitsV, shotsV, acc)
        end
    end
end

-- observa mudanças
local function onLeaderstatsAdded()
    local ls = localPlayer:FindFirstChild("leaderstats")
    if ls then
        for _, v in pairs(ls:GetChildren()) do
            v.Changed:Connect(updateHUD)
        end
    end
    updateHUD()
end

if localPlayer:FindFirstChild("leaderstats") then
    onLeaderstatsAdded()
end
local addedConn
addedConn = localPlayer.ChildAdded:Connect(function(child)
    if child.Name == "leaderstats" then
        onLeaderstatsAdded()
        addedConn:Disconnect()
    end
end)

-- feedback visual ao clicar (pequeno recoil UI)
local recoilAmt = 4
local function clickRecoil()
    local origPos = cross.Position
    local tweenTime = 0.06
    cross.Position = UDim2.new(0.5, 0, 0.5, -recoilAmt)
    wait(tweenTime)
    cross.Position = origPos
end

-- optional: som de clique local
local clickSound = Instance.new("Sound", workspace)
clickSound.SoundId = "rbxassetid://12222216" -- substitua se quiser outro som (ou remova)
clickSound.Volume = 0.6
clickSound.PlayOnRemove = false

UserInputService.InputBegan:Connect(function(input, gameProcessed)
    if gameProcessed then return end
    if input.UserInputType == Enum.UserInputType.MouseButton1 then
        -- feedbacks
        coroutine.wrap(function()
            pcall(clickRecoil)
        end)()
        pcall(function() clickSound:Play() end)
    end
end)

-- loop para atualizar UI central (crosshair) suavemente se quiser (aqui apenas mantém fixo)
RunService.RenderStepped:Connect(function()
    -- opcional: pulsar crosshair para efeito
    -- cross.BackgroundTransparency = 0.0 + (math.abs(math.sin(tick()*6)) * 0.2)
end)

print("[TrainingClient] Cliente inicializado.")
