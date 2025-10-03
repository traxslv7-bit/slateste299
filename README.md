-- SafeDevInvulnerable.lua
-- Script SERVER-SIDE para habilitar invulnerabilidade temporária a jogadores autorizados (apenas para DEV / TESTES)
-- ATENÇÃO: Somente use em servidores de teste/controlados. NÃO distribuia ou ative em servidores públicos.

local Players = game:GetService("Players")
local HttpService = game:GetService("HttpService") -- usado só para gerar IDs de log; opção
local RunService = game:GetService("RunService")

-- CONFIGURAÇÃO: IDs dos desenvolvedores/admins que podem receber invulnerabilidade
local ALLOWED_ADMIN_USERIDS = {
    -- substitua pelos seus UserIds
    12345678, -- Matheus (exemplo)
    -- adicionar outros...
}

-- Tempo padrão de invulnerabilidade em segundos (0 = indefinido até desativação manual)
local DEFAULT_INVULN_DURATION = 0

-- Logs em server console e coleção local
local invulnLogs = {}

-- Estado por jogador
local invulnState = {} -- [player] = {enabled = bool, expiresAt = number or nil}

-- Função util: verifica se userid é admin
local function isAdmin(userId)
    for _, id in ipairs(ALLOWED_ADMIN_USERIDS) do
        if id == userId then return true end
    end
    return false
end

-- Ativa invulnerabilidade para um Player (server-side)
-- duration em segundos; 0 = indefinido (até desativação manual)
local function enableInvulnerable(player, duration)
    if not player or not player.Parent then return false, "Player inválido" end
    if not isAdmin(player.UserId) then return false, "Não autorizado" end

    local now = os.time()
    local expires = nil
    if duration and duration > 0 then expires = now + duration end

    invulnState[player] = { enabled = true, expiresAt = expires, activatedAt = now }

    -- Mecanismo server-side: interceptar dano colocando um listener no Humanoid
    local character = player.Character
    if character then
        local humanoid = character:FindFirstChildOfClass("Humanoid")
        if humanoid then
            -- defina uma tag no humanoid para identificar invulnerabilidade (opcional)
            humanoid:SetAttribute("DevInvulnerable", true)
            -- Também garantir HP alto e bloquear TakeDamage events (server-side)
            humanoid.MaxHealth = math.max(humanoid.MaxHealth, 999999)
            humanoid.Health = humanoid.MaxHealth
        end
    end

    -- Log
    local logEntry = string.format("%s | ENABLE | Player=%s (%d) | duration=%s", os.date("%Y-%m-%d %H:%M:%S", now), player.Name, player.UserId, tostring(duration))
    table.insert(invulnLogs, logEntry)
    print(logEntry)

    return true, "Invulnerabilidade ativada"
end

-- Desativa invulnerabilidade para um Player (server-side)
local function disableInvulnerable(player)
    if not player then return false, "Player inválido" end
    if not invulnState[player] or not invulnState[player].enabled then return false, "Já desativado" end

    invulnState[player].enabled = false
    invulnState[player].expiresAt = nil

    local character = player.Character
    if character then
        local humanoid = character:FindFirstChildOfClass("Humanoid")
        if humanoid then
            humanoid:SetAttribute("DevInvulnerable", false)
            -- Opcional: restaurar HP para um valor padrão seguro
            humanoid.MaxHealth = 100
            humanoid.Health = humanoid.MaxHealth
        end
    end

    local now = os.time()
    local logEntry = string.format("%s | DISABLE | Player=%s (%d)", os.date("%Y-%m-%d %H:%M:%S", now), player.Name, player.UserId)
    table.insert(invulnLogs, logEntry)
    print(logEntry)
    return true, "Invulnerabilidade removida"
end

-- Intercepta evento de dano (exemplo defensivo): redefine qualquer dano recebido se invulnerável
-- OBS: Não confie apenas nisso para produção; use um sistema de dano centralizado server-side.
local function onCharacterAdded(player, character)
    local humanoid = character:WaitForChild("Humanoid", 5)
    if not humanoid then return end

    -- sempre que o Humanoid tomar dano, checar atributo
    -- NOTE: Roblox não expõe evento "TakingDamage" diretamente; usamos Changed("Health") como workaround.
    local lastHealth = humanoid.Health
    humanoid:GetPropertyChangedSignal("Health"):Connect(function()
        if invulnState[player] and invulnState[player].enabled then
            -- restaurar imediatamente
            if humanoid.Health < humanoid.MaxHealth then
                humanoid.Health = humanoid.MaxHealth
            end
        end
        lastHealth = humanoid.Health
    end)
end

-- Monitora players entrando/saindo
Players.PlayerAdded:Connect(function(player)
    -- inicial estado
    invulnState[player] = invulnState[player] or { enabled = false, expiresAt = nil }

    player.CharacterAdded:Connect(function(character)
        -- aplica atributos se já estiver invulnerável
        if invulnState[player] and invulnState[player].enabled then
            local humanoid = character:FindFirstChildOfClass("Humanoid")
            if humanoid then
                humanoid:SetAttribute("DevInvulnerable", true)
                humanoid.MaxHealth = math.max(humanoid.MaxHealth, 999999)
                humanoid.Health = humanoid.MaxHealth
            end
        end
        -- ligar listener defensivo
        onCharacterAdded(player, character)
    end)
end)

Players.PlayerRemoving:Connect(function(player)
    invulnState[player] = nil
end)

-- Tick para expirar invulnerabilidades configuradas
RunService.Heartbeat:Connect(function(dt)
    local now = os.time()
    for player, state in pairs(invulnState) do
        if state.enabled and state.expiresAt and state.expiresAt <= now then
            disableInvulnerable(player)
        end
    end
end)

-- Expor uma API segura via BindableFunction/RemoteFunction somente server-side (opcional)
-- NÃO expor um RemoteEvent público que qualquer cliente possa chamar para virar invulnerável.
-- Em vez disso, use Console/CommandBar ou RemoteFunction com validação server-side estrita (ex.: checar userId).
-- Aqui vai um exemplo simples de BindableFunction (server-only admin tool):
local ServerStorage = game:GetService("ServerStorage")
local bindableFolder = ServerStorage:FindFirstChild("DevAdminBindables") or Instance.new("Folder", ServerStorage)
bindableFolder.Name = "DevAdminBindables"

local enableFunc = Instance.new("BindableFunction")
enableFunc.Name = "EnableDevInvuln"
enableFunc.Parent = bindableFolder

enableFunc.OnInvoke = function(userId, duration)
    -- só permitir se userId for admin (passado por server-side tool)
    if not isAdmin(userId) then return false, "Não autorizado" end
    -- buscar Player
    for _, p in pairs(Players:GetPlayers()) do
        if p.UserId == userId then
            return enableInvulnerable(p, duration or DEFAULT_INVULN_DURATION)
        end
    end
    return false, "Player não encontrado"
end

local disableFunc = Instance.new("BindableFunction")
disableFunc.Name = "DisableDevInvuln"
disableFunc.Parent = bindableFolder
disableFunc.OnInvoke = function(userId)
    for _, p in pairs(Players:GetPlayers()) do
        if p.UserId == userId then
            return disableInvulnerable(p)
        end
    end
    return false, "Player não encontrado"
end

-- UTILITÁRIOS: comandos via Output/Server Console (para testar manualmente)
print("DevInvulnerable script carregado. Use bindables em ServerStorage.DevAdminBindables para controlar invuln.")

-- Exemplo rápido de uso em console (Server Script):
-- local bf = game:GetService("ServerStorage").DevAdminBindables.EnableDevInvuln
-- bf:Invoke(12345678, 0)  -- ativa invuln indefinido para userid 12345678

-- FIM
