-- TargetManager (ServerScriptService)
-- Gera alvos, valida hits vindos do cliente e controla pontuação

local ReplicatedStorage = game:GetService("ReplicatedStorage")
local Players = game:GetService("Players")
local Debris = game:GetService("Debris")
local Workspace = game:GetService("Workspace")

-- RemoteEvent
local event = ReplicatedStorage:FindFirstChild("TrainingEvent")
if not event then
    event = Instance.new("RemoteEvent")
    event.Name = "TrainingEvent"
    event.Parent = ReplicatedStorage
end

-- Config
local ARENA_FOLDER_NAME = "TrainingArena"
local TARGET_COUNT = 8
local TARGET_RADIUS = 2.5       -- raio (studs) pra considerar um acerto
local TARGET_LIFETIME = 6      -- segundos visível antes de sumir / respawn
local RESPAWN_DELAY = 1.2

-- Localiza / cria folder de arena
local arenaFolder = Workspace:FindFirstChild(ARENA_FOLDER_NAME)
if not arenaFolder then
    arenaFolder = Instance.new("Folder")
    arenaFolder.Name = ARENA_FOLDER_NAME
    arenaFolder.Parent = Workspace
end

-- Função util: cria um target físico (part) em posição aleatória dentro de uma área
local function createTarget(position)
    local part = Instance.new("Part")
    part.Size = Vector3.new(1.4, 1.4, 1.4)
    part.Shape = Enum.PartType.Ball
    part.Material = Enum.Material.Neon
    part.Anchored = true
    part.CanCollide = false
    part.Position = position
    part.Name = "TrainingTarget"
    part.Parent = arenaFolder

    -- Tag visual (Billboard)
    local bill = Instance.new("BillboardGui", part)
    bill.Size = UDim2.new(0,100,0,30)
    bill.StudsOffset = Vector3.new(0, 1.7, 0)
    bill.AlwaysOnTop = true
    local frame = Instance.new("Frame", bill)
    frame.Size = UDim2.new(1,0,1,0)
    frame.BackgroundTransparency = 0.2
    frame.BackgroundColor3 = Color3.fromRGB(0,0,0)
    local txt = Instance.new("TextLabel", frame)
    txt.Size = UDim2.new(1,-6,1,-6)
    txt.Position = UDim2.new(0,3,0,3)
    txt.BackgroundTransparency = 1
    txt.Text = "ALVO"
    txt.Font = Enum.Font.GothamBold
    txt.TextColor3 = Color3.fromRGB(255,255,255)
    txt.TextScaled = true

    return part
end

-- Gera posições dentro de uma área simples (retângulo) — ajuste conforme seu mapa
local function randomPositions(center, rangeX, rangeZ, count)
    local positions = {}
    for i = 1, count do
        local x = center.X + math.random(-rangeX*10, rangeX*10) / 10
        local z = center.Z + math.random(-rangeZ*10, rangeZ*10) / 10
        local y = center.Y
        -- garante pelo menos um pequeno offset de altura
        table.insert(positions, Vector3.new(x, y, z))
    end
    return positions
end

-- Gera targets iniciais
local function spawnTargets()
    -- centraliza na posição do SpawnLocation se existir, senão usa posição (0,5,0)
    local center = Vector3.new(0, 5, 0)
    local spawnLocation = Workspace:FindFirstChildWhichIsA("SpawnLocation")
    if spawnLocation then center = spawnLocation.Position + Vector3.new(0, 3, 0) end

    local posList = randomPositions(center, 20, 12, TARGET_COUNT)
    for i, pos in ipairs(posList) do
        local t = createTarget(pos)
        -- movimento simples: tween vertical +/- e loop
        local amplitude = 1 + math.random() * 1.6
        local speed = 1 + math.random() * 1.8
        spawn(function()
            local direction = 1
            while t.Parent and t:IsDescendantOf(game) do
                local newPos = t.Position + Vector3.new(0, 0.8 * math.sin(tick()*speed) * amplitude, 0)
                t.Position = newPos
                wait(0.03)
            end
        end)
    end
end

-- Remove e respawn target (quando acertado)
local function handleTargetHit(targetPart)
    if not (targetPart and targetPart:IsDescendantOf(arenaFolder)) then return end
    -- efeito visual
    local p = Instance.new("Part")
    p.Size = Vector3.new(1.5, 1.5, 1.5)
    p.CFrame = targetPart.CFrame
    p.Anchored = true
    p.CanCollide = false
    p.Material = Enum.Material.Neon
    p.BrickColor = BrickColor.new("Bright red")
    p.Parent = arenaFolder
    Debris:AddItem(p, 0.6)

    targetPart:Destroy()
    -- respawn simples depois de alguns segundos
    delay(RESPAWN_DELAY, function()
        -- spawn em torno da arena (reusing spawnTargets but single)
        local center = Vector3.new(0,5,0)
        local spawnLocation = Workspace:FindFirstChildWhichIsA("SpawnLocation")
        if spawnLocation then center = spawnLocation.Position + Vector3.new(0, 3, 0) end
        local pos = center + Vector3.new(math.random(-18,18), 0, math.random(-10,10))
        local t = createTarget(pos)
        -- movimento simples novamente
        local amplitude = 1 + math.random() * 1.6
        local speed = 1 + math.random() * 1.8
        spawn(function()
            while t.Parent and t:IsDescendantOf(game) do
                local newPos = t.Position + Vector3.new(0, 0.8 * math.sin(tick()*speed) * amplitude, 0)
                t.Position = newPos
                wait(0.03)
            end
        end)
    end)
end

-- Leaderstats simples
local function onPlayerAdded(pl)
    local leaderstats = Instance.new("Folder")
    leaderstats.Name = "leaderstats"
    leaderstats.Parent = pl

    local score = Instance.new("IntValue")
    score.Name = "Score"
    score.Value = 0
    score.Parent = leaderstats

    local shots = Instance.new("IntValue")
    shots.Name = "Shots"
    shots.Value = 0
    shots.Parent = leaderstats

    local hits = Instance.new("IntValue")
    hits.Name = "Hits"
    hits.Value = 0
    hits.Parent = leaderstats
end
Players.PlayerAdded:Connect(onPlayerAdded)

-- Evento: cliente disparou com posição do mouse
-- payload: {position = Vector3, timestamp = number}
event.OnServerEvent:Connect(function(player, payload)
    if not payload or typeof(payload.position) ~= "Vector3" then return end
    local pos = payload.position
    -- increment shots
    if player and player:FindFirstChild("Shots") == nil and player:FindFirstChild("leaderstats") then
        -- some setups may differ; attempt set below
    end
    if player and player:FindFirstChild("leaderstats") then
        local shots = player.leaderstats:FindFirstChild("Shots")
        if shots then shots.Value = shots.Value + 1 end
    end

    -- valida colisão com alvos no raio TARGET_RADIUS
    for _, part in ipairs(arenaFolder:GetChildren()) do
        if part:IsA("BasePart") and part.Name == "TrainingTarget" then
            local dist = (part.Position - pos).Magnitude
            if dist <= TARGET_RADIUS then
                -- player acerta target
                -- increment hits / score
                if player and player:FindFirstChild("leaderstats") then
                    local hits = player.leaderstats:FindFirstChild("Hits")
                    local score = player.leaderstats:FindFirstChild("Score")
                    if hits then hits.Value = hits.Value + 1 end
                    if score then score.Value = score.Value + 10 end
                end
                -- efeito e respawn
                handleTargetHit(part)
                break
            end
        end
    end
end)

-- inicializa targets se não houver
spawn(function()
    -- cleanup antigos
    for _, c in ipairs(arenaFolder:GetChildren()) do
        if c:IsA("BasePart") and c.Name == "TrainingTarget" then c:Destroy() end
    end
    wait(0.2)
    spawnTargets()
end)

print("[TargetManager] Sistema de treino inicializado.")
