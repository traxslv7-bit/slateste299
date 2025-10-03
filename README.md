--// AIMBOT DEV TEST SCRIPT (uso local/offline para estudo)
--// NÃO use em servidores públicos, apenas para testes no Roblox Studio/privado.

-- Variáveis principais
local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local UserInputService = game:GetService("UserInputService")
local LocalPlayer = Players.LocalPlayer
local Camera = workspace.CurrentCamera

-- Configurações do Aimbot
local Settings = {
    Enabled = false,       -- Liga/Desliga
    TeamCheck = true,      -- Ignorar aliados
    Smoothness = 0.12,     -- Quanto menor, mais rápido a mira acompanha
    AimPart = "Head",      -- Parte alvo
    MaxDistance = 200,     -- Distância máxima
    FOV = 100,             -- Campo de visão
}

-- Criar GUI (painel simples)
local ScreenGui = Instance.new("ScreenGui")
ScreenGui.Name = "AimbotPanel"
ScreenGui.Parent = game:GetService("CoreGui")

local Frame = Instance.new("Frame", ScreenGui)
Frame.Size = UDim2.new(0, 180, 0, 140)
Frame.Position = UDim2.new(0, 20, 0, 200)
Frame.BackgroundColor3 = Color3.fromRGB(25,25,25)
Frame.BorderSizePixel = 0

local Title = Instance.new("TextLabel", Frame)
Title.Size = UDim2.new(1,0,0,30)
Title.BackgroundTransparency = 1
Title.Text = "Aimbot Dev Panel"
Title.TextColor3 = Color3.fromRGB(255,255,255)
Title.Font = Enum.Font.SourceSansBold
Title.TextSize = 18

local ToggleBtn = Instance.new("TextButton", Frame)
ToggleBtn.Size = UDim2.new(0.9,0,0,28)
ToggleBtn.Position = UDim2.new(0.05,0,0,40)
ToggleBtn.Text = "Aimbot: OFF"
ToggleBtn.BackgroundColor3 = Color3.fromRGB(50,50,50)
ToggleBtn.TextColor3 = Color3.fromRGB(255,255,255)
ToggleBtn.Font = Enum.Font.SourceSans
ToggleBtn.TextSize = 16

local TeamBtn = Instance.new("TextButton", Frame)
TeamBtn.Size = UDim2.new(0.9,0,0,28)
TeamBtn.Position = UDim2.new(0.05,0,0,75)
TeamBtn.Text = "TeamCheck: ON"
TeamBtn.BackgroundColor3 = Color3.fromRGB(50,50,50)
TeamBtn.TextColor3 = Color3.fromRGB(255,255,255)
TeamBtn.Font = Enum.Font.SourceSans
TeamBtn.TextSize = 16

-- Botões
ToggleBtn.MouseButton1Click:Connect(function()
    Settings.Enabled = not Settings.Enabled
    ToggleBtn.Text = "Aimbot: " .. (Settings.Enabled and "ON" or "OFF")
end)

TeamBtn.MouseButton1Click:Connect(function()
    Settings.TeamCheck = not Settings.TeamCheck
    TeamBtn.Text = "TeamCheck: " .. (Settings.TeamCheck and "ON" or "OFF")
end)

-- Funções auxiliares
local function GetClosestPlayer()
    local closest = nil
    local shortest = Settings.FOV

    for _,plr in pairs(Players:GetPlayers()) do
        if plr ~= LocalPlayer and plr.Character and plr.Character:FindFirstChild(Settings.AimPart) then
            local head = plr.Character[Settings.AimPart]
            local pos, onScreen = Camera:WorldToViewportPoint(head.Position)

            if onScreen then
                local dist = (Vector2.new(pos.X,pos.Y) - UserInputService:GetMouseLocation()).Magnitude
                local mag = (head.Position - Camera.CFrame.Position).Magnitude

                if dist < shortest and mag <= Settings.MaxDistance then
                    if Settings.TeamCheck and plr.Team == LocalPlayer.Team then
                        -- ignora aliado
                    else
                        shortest = dist
                        closest = head
                    end
                end
            end
        end
    end

    return closest
end

-- Loop principal (aim)
RunService.RenderStepped:Connect(function()
    if Settings.Enabled then
        local target = GetClosestPlayer()
        if target then
            local targetPos, _ = Camera:WorldToViewportPoint(target.Position)
            local mousePos = UserInputService:GetMouseLocation()
            local moveX = (targetPos.X - mousePos.X) * Settings.Smoothness
            local moveY = (targetPos.Y - mousePos.Y) * Settings.Smoothness
            mousemoverel(moveX, moveY) -- move o mouse no PC (função suportada em executores)
        end
    end
end)
