-- main.lua
-- Simulador de "Aimbot" para TESTES / DETECÇÃO - Love2D
-- NÃO interage com jogos reais. Apenas simula comportamento.

math.randomseed(os.time())

local players = {}
local localTeam = "Azul"
local crosshair = { x = 400, y = 300 } -- posição inicial da mira (center)
local aimActive = false
local aimKey = "space" -- tecla que ativa o "aim"
local teamCheck = true
local aimFOV = 200 -- raio de detecção em pixels
local smoothing = 0.12 -- 0 = snap, 1 = muito lento
local predictionFactor = 0.25 -- quanto usar da velocidade do alvo para prever
local showHUD = true
local closestTarget = nil

local function createMockPlayers(n)
    local teams = {"Azul", "Vermelho"}
    for i = 1, n do
        local team = teams[math.random(#teams)]
        local p = {
            id = i,
            name = "Player"..i,
            team = team,
            x = math.random(50, 750),
            y = math.random(50, 550),
            vx = math.random(-80, 80), -- pixels/sec
            vy = math.random(-80, 80),
            radius = 12,
            health = math.random(40, 100)
        }
        table.insert(players, p)
    end
end

local function distance(x1,y1,x2,y2)
    local dx = x1-x2; local dy = y1-y2
    return math.sqrt(dx*dx + dy*dy)
end

local function isValidTarget(p)
    if p.health <= 0 then return false end
    if teamCheck and p.team == localTeam then return false end
    return true
end

local function findTarget()
    local best = nil
    local bestDist = math.huge
    for _,p in ipairs(players) do
        if isValidTarget(p) then
            local d = distance(crosshair.x, crosshair.y, p.x, p.y)
            if d <= aimFOV and d < bestDist then
                bestDist = d
                best = p
            end
        end
    end
    return best, bestDist
end

local function predictPosition(p, dt)
    -- predição simples baseada na velocidade do alvo
    local px = p.x + p.vx * predictionFactor * dt
    local py = p.y + p.vy * predictionFactor * dt
    return px, py
end

local function aimAt(target, dt)
    if not target then return end
    local px, py = predictPosition(target, dt)
    local dx = px - crosshair.x
    local dy = py - crosshair.y
    -- aplicar smoothing: mover a mira uma fração do vetor
    crosshair.x = crosshair.x + dx * smoothing
    crosshair.y = crosshair.y + dy * smoothing
end

function love.load()
    love.window.setMode(800, 600)
    love.window.setTitle("Simulador de Aimbot para Testes (Sandbox)")
    createMockPlayers(8)
    font = love.graphics.newFont(14)
    love.graphics.setFont(font)
end

function love.update(dt)
    -- atualizar posições dos "players" mock
    for _,p in ipairs(players) do
        p.x = p.x + p.vx * dt
        p.y = p.y + p.vy * dt
        -- bounce nas bordas
        if p.x < 20 then p.x = 20; p.vx = -p.vx end
        if p.x > 780 then p.x = 780; p.vx = -p.vx end
        if p.y < 20 then p.y = 20; p.vy = -p.vy end
        if p.y > 580 then p.y = 580; p.vy = -p.vy end
    end

    -- encontrar alvo mais próximo dentro do FOV
    closestTarget = findTarget()

    -- se aim ativo e existe alvo -> mover a mira
    if aimActive and closestTarget then
        aimAt(closestTarget, dt)
    end

    -- opcional: mover crosshair com mouse quando aim inativo
    if not aimActive then
        local mx, my = love.mouse.getPosition()
        crosshair.x = mx
        crosshair.y = my
    end
end

function love.draw()
    -- background
    love.graphics.clear(0.08, 0.08, 0.08)

    -- desenhar jogadores
    for _,p in ipairs(players) do
        if p.team == localTeam then
            love.graphics.setColor(0.1, 0.5, 1, 1) -- azul (aliado)
        else
            love.graphics.setColor(1, 0.1, 0.1, 1) -- vermelho (inimigo)
        end
        love.graphics.circle("fill", p.x, p.y, p.radius)
        love.graphics.setColor(1,1,1)
        love.graphics.print(p.name.." ("..p.team..")", p.x - 24, p.y - 26)
        love.graphics.print("HP:"..p.health, p.x - 24, p.y + 14)
    end

    -- desenhar FOV
    love.graphics.setColor(1,1,1,0.06)
    love.graphics.circle("fill", crosshair.x, crosshair.y, aimFOV)

    -- desenhar crosshair
    love.graphics.setColor(1,1,1)
    love.graphics.setLineWidth(2)
    love.graphics.line(crosshair.x - 10, crosshair.y, crosshair.x + 10, crosshair.y)
    love.graphics.line(crosshair.x, crosshair.y - 10, crosshair.x, crosshair.y + 10)
    love.graphics.circle("line", crosshair.x, crosshair.y, 8)

    -- se existe alvo, desenhar linha até ele e predição
    if closestTarget then
        love.graphics.setColor(1,1,0,0.9)
        love.graphics.line(crosshair.x, crosshair.y, closestTarget.x, closestTarget.y)
        local px, py = predictPosition(closestTarget, 1) -- pred com dt=1 para visual
        love.graphics.setColor(1,1,0,0.5)
        love.graphics.circle("line", px, py, 10)
        love.graphics.print("Alvo: "..closestTarget.name.." ("..closestTarget.team..")", 10, 10)
    else
        love.graphics.setColor(1,1,1)
        love.graphics.print("Alvo: nenhum dentro do FOV", 10, 10)
    end

    -- HUD
    if showHUD then
        love.graphics.setColor(0,0,0,0.6)
        love.graphics.rectangle("fill", 10, 420, 300, 160, 6)
        love.graphics.setColor(1,1,1)
        love.graphics.print("CONTROLES:", 20, 430)
        love.graphics.print("Tecla ["..aimKey.."] : Toggle Aim (atual: "..(aimActive and "ON" or "OFF")..")", 20, 452)
        love.graphics.print("T : Toggle Team Check (atual: "..(teamCheck and "ON" or "OFF")..")", 20, 474)
        love.graphics.print("S : Aumentar smoothing (atual: "..string.format("%.3f", smoothing)..")", 20, 496)
        love.graphics.print("A : Diminuir smoothing", 20, 518)
        love.graphics.print("F : Aumentar FOV (atual: "..aimFOV..")   G : Diminuir FOV", 20, 540)
    end
end

function love.keypressed(key)
    if key == aimKey then
        aimActive = not aimActive
    elseif key == "t" then
        teamCheck = not teamCheck
    elseif key == "s" then
        smoothing = math.min(0.9, smoothing + 0.02)
    elseif key == "a" then
        smoothing = math.max(0, smoothing - 0.02)
    elseif key == "f" then
        aimFOV = math.min(600, aimFOV + 10)
    elseif key == "g" then
        aimFOV = math.max(20, aimFOV - 10)
    elseif key == "h" then
        showHUD = not showHUD
    elseif key == "r" then
        -- reiniciar mock players
        players = {}
        createMockPlayers(8)
    elseif key == "escape" then
        love.event.quit()
    end
end
