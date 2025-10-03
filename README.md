-- Simulador Anti-Hit + ESP (apenas para testes locais no Roblox Studio)
-- Coloque este LocalScript em StarterGui e rode em Play Solo.
-- Não interage com Remotes/Servidores: tudo é local e seguro para testes.

local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local UserInputService = game:GetService("UserInputService")

local localPlayer = Players.LocalPlayer

-- CONFIG
local NUM_SIM_PLAYERS = 6
local SIM_AREA_SIZE = 60 -- área onde os "players" serão criados
local ESP_DISTANCE_LIMIT = 300 -- distância máxima para mostrar ESP

-- ---------- Cria GUI ----------
local screenGui = Instance.new("ScreenGui")
screenGui.Name = "AntiHitSimulatorGUI"
screenGui.ResetOnSpawn = false
screenGui.Parent = localPlayer:WaitForChild("PlayerGui")

-- Painel simples
local panel = Instance.new("Frame")
panel.Name = "Panel"
panel.Size = UDim2.new(0, 260, 0, 120)
panel.Position = UDim2.new(0, 12, 0, 12)
panel.BackgroundTransparency = 0.15
panel.BackgroundColor3 = Color3.fromRGB(18,18,18)
panel.BorderSizePixel = 0
panel.Parent = screenGui

local title = Instance.new("TextLabel")
title.Name = "Title"
title.Size = UDim2.new(1, -12, 0, 28)
title.Position = UDim2.new(0, 6, 0, 6)
title.BackgroundTransparency = 1
title.Text = "Simulator — AntiHit Test"
title.TextColor3 = Color3.fromRGB(255,255,255)
title.Font = Enum.Font.SourceSansBold
title.TextSize = 18
title.TextXAlignment = Enum.TextXAlignment.Left
title.Parent = panel

-- Toggle button
local toggleBtn = Instance.new("TextButton")
toggleBtn.Name = "AntiHitToggle"
toggleBtn.Size = UDim2.new(0, 120, 0, 36)
toggleBtn.Position = UDim2.new(0, 8, 0, 40)
toggleBtn.Text = "AntiHit: OFF"
toggleBtn.Font = Enum.Font.SourceSansBold
toggleBtn.TextSize = 16
toggleBtn.BackgroundColor3 = Color3.fromRGB(40,40,40)
toggleBtn.TextColor3 = Color3.fromRGB(255,255,255)
toggleBtn.BorderSizePixel = 0
toggleBtn.Parent = panel

-- Info label
local infoLabel = Instance.new("TextLabel")
infoLabel.Name = "Info"
infoLabel.Size = UDim2.new(1, -16, 0, 36)
infoLabel.Position = UDim2.new(0, 8, 0, 84)
infoLabel.BackgroundTransparency = 1
infoLabel.Text = "Clique em um alvo para 'atacar'."
infoLabel.TextColor3 = Color3.fromRGB(200,200,200)
infoLabel.Font = Enum.Font.SourceSans
infoLabel.TextSize = 14
infoLabel.TextXAlignment = Enum.TextXAlignment.Left
infoLabel.Parent = panel

-- Toggle state
local antiHitEnabled = false
local function updateToggleUI()
    if antiHitEnabled then
        toggleBtn.Text = "AntiHit: ON"
        toggleBtn.BackgroundColor3 = Color3.fromRGB(30, 120, 40)
    else
        toggleBtn.Text = "AntiHit: OFF"
        toggleBtn.BackgroundColor3 = Color3.fromRGB(40,40,40)
    end
end

toggleBtn.MouseButton1Click:Connect(function()
    antiHitEnabled = not antiHitEnabled
    updateToggleUI()
end)

updateToggleUI()

-- ---------- Simula "players" na cena ----------
local workspace = game:GetService("Workspace")
local simFolder = Instance.new("Folder", workspace)
simFolder.Name = "SimulatedPlayers"

-- Limpa caso exista
for _,c in pairs(simFolder:GetChildren()) do c:Destroy() end

local simulated = {} -- tabela de "players" simulados

local function createSimulatedPlayer(id, position)
    local part = Instance.new("Part")
    part.Name = "SimPlayer_"..id
    part.Size = Vector3.new(2, 5, 2)
    part.Position = position
    part.Anchored = true
    part.Parent = simFolder
    part.CanCollide = false

    -- Humanoid-like info (simulado)
    local tag = Instance.new("BillboardGui", part)
    tag.Name = "ESP"
    tag.Size = UDim2.new(0,120,0,40)
    tag.AlwaysOnTop = true
    tag.StudsOffset = Vector3.new(0, 3.3, 0)

    local label = Instance.new("TextLabel", tag)
    label.Size = UDim2.new(1,0,1,0)
    label.BackgroundTransparency = 1
    label.Text = "PlayerSim_"..id
    label.Font = Enum.Font.SourceSansBold
    label.TextSize = 14
    label.TextColor3 = Color3.fromRGB(255,255,255)

    table.insert(simulated, {
        id = id,
        part = part,
        label = label,
        health = 100
    })
end

-- gerar alguns "players" aleatórios
math.randomseed(tick())
for i=1,NUM_SIM_PLAYERS do
    local x = (math.random() - 0.5) * SIM_AREA_SIZE
    local z = (math.random() - 0.5) * SIM_AREA_SIZE
    local pos = workspace.Baseplate.Position + Vector3.new(x, 3, z)
    createSimulatedPlayer(i, pos)
end

-- ---------- ESP: desenha ou oculta dependendo da distância ----------
local camera = workspace.CurrentCamera
RunService.RenderStepped:Connect(function()
    for _,sim in pairs(simulated) do
        local dist = (sim.part.Position - camera.CFrame.Position).Magnitude
        sim.label.Visible = dist <= ESP_DISTANCE_LIMIT
        -- muda cor do texto com base no health
        local healthPct = sim.health / 100
        local r = math.clamp(1 - healthPct, 0, 1)
        local g = math.clamp(healthPct, 0, 1)
        sim.label.TextColor3 = Color3.new(g, 1-r, 0.2) -- só para visual
        sim.label.Text = string.format("PlayerSim_%d  |  HP: %d", sim.id, math.floor(sim.health))
    end
end)

-- ---------- Simula "ataque" ao clicar com o mouse ----------
local mouse = localPlayer:GetMouse()

local function showFloatingText(position, text)
    local billboard = Instance.new("BillboardGui", workspace.Terrain)
    billboard.Size = UDim2.new(0,120,0,30)
    billboard.StudsOffset = Vector3.new(0, 3, 0)
    billboard.Adornee = nil
    billboard.AlwaysOnTop = true
    billboard.MaxDistance = 200

    local label = Instance.new("TextLabel", billboard)
    label.Size = UDim2.new(1,0,1,0)
    label.BackgroundTransparency = 0.5
    label.BackgroundColor3 = Color3.new(0,0,0)
    label.TextColor3 = Color3.new(1,1,1)
    label.Text = text
    label.Font = Enum.Font.SourceSansBold
    label.TextSize = 14

    -- posiciona próximo ao ponto passado convertendo em CFrame
    billboard.WorldPosition = position

    delay(1.2, function()
        billboard:Destroy()
    end)
end

-- ao clicar, verifica se clicou em alguma parte simulada
mouse.Button1Down:Connect(function()
    local target = mouse.Target
    if not target then return end

    -- se clicar em um SimPlayer part
    for _,sim in pairs(simulated) do
        if target == sim.part then
            -- Lógica de "hit" simulada:
            if antiHitEnabled then
                -- AntiHit cancela o dano — registramos evento de "blocked"
                showFloatingText(sim.part.Position, "HIT BLOCKED (AntiHit ON)")
                -- Podemos marcar para fins de detecção: incrementar contador local
                sim._blocked = (sim._blocked or 0) + 1
            else
                -- aplica dano normal
                local dano = math.random(8, 25)
                sim.health = math.max(0, sim.health - dano)
                showFloatingText(sim.part.Position, "- "..dano.." HP")
                if sim.health <= 0 then
                    showFloatingText(sim.part.Position, "SIM PLAYER DOWN")
                    -- efeito simples: tornar invisível
                    sim.part.Transparency = 0.8
                    sim.label.Text = sim.label.Text .. " [DOWN]"
                end
            end
            break
        end
    end
end)

-- ---------- Mostra um pequeno log quando AntiHit bloqueia muitos hits ----------
local function monitorBlocked()
    while true do
        wait(4)
        for _,sim in pairs(simulated) do
            if sim._blocked and sim._blocked >= 3 then
                -- aqui você poderia enviar um alerta local, log, etc. (apenas visual)
                infoLabel.Text = string.format("Detectado %d bloqueios contra PlayerSim_%d", sim._blocked, sim.id)
            end
        end
    end
end

spawn(monitorBlocked)

-- ---------- Mensagem inicial ----------
infoLabel.Text = "Clique nos alvos para test. AntiHit bloqueia o dano localmente."

-- FIM
