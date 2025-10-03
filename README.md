-- main.lua
-- Simulador de Aimbot COMPLETO para TESTES / DETECÇÃO - Love2D
-- ATENÇÃO: Não é um cheat. NÃO acessa jogos reais. Uso somente para simulação e gerar logs.

math.randomseed(os.time())

-- CONFIG
local WINDOW_W, WINDOW_H = 900, 650
local localTeam = "Azul"

-- mira / comportamento
local crosshair = { x = WINDOW_W/2, y = WINDOW_H/2 }
local aimActive = false
local aimKey = "space"
local teamCheck = true
local aimFOV = 220
local smoothing = 0.14
local predictionFactor = 0.28
local aimBone = "head" -- "head" or "body"
local aimMode = "closest" -- "closest" | "fov_closest" | "health" (selection strategy)

-- players (mocks)
local players = {}
local function createMockPlayers(n)
    players = {}
    local teams = {"Azul","Vermelho"}
    for i=1,n do
        local t = teams[math.random(#teams)]
        table.insert(players, {
            id = i,
            name = "Player"..i,
            team = t,
            x = math.random(40, WINDOW_W-40),
            y = math.random(40, WINDOW_H-40),
            vx = math.random(-90,90),
            vy = math.random(-90,90),
            radius = math.random(10,14),
            health = math.random(30,100),
            head_offset = -14 -- offset visual
        })
    end
end

-- logging (gera CSV)
local logEvents = {}
local function logShot(target, dist)
    local ts = os.time()
    table.insert(logEvents, string.format("%d,%s,%d,%d,%d,%.2f,%.3f,%.3f",
        ts, target.name, target.id, target.health, dist, smoothing, predictionFactor, aimFOV))
end

local function saveLogCSV()
    if #logEvents == 0 then return false, "No events to save" end
    local header = "timestamp,name,id,health,distance,smoothing,predictionFactor,aimFOV\n"
    local content = header .. table.concat(logEvents, "\n")
    local filename = "aim_sim_log_" .. os.time() .. ".csv"
    love.filesystem.write(filename, content)
    return true, filename
end

-- util
local function dist(x1,y1,x2,y2)
    local dx = x1-x2; local dy = y1-y2; return math.sqrt(dx*dx + dy*dy)
end

local function isValidTarget(p)
    if p.health <= 0 then return false end
    if teamCheck and p.team == localTeam then return false end
    return true
end

-- estrategia de escolha de alvo
local function scoreForTarget(p)
    local d = dist(crosshair.x, crosshair.y, p.x, p.y)
    local score = 0
    if aimMode == "closest" then
        score = -d
    elseif aimMode == "fov_closest" then
        if d <= aimFOV then score = 1000 - d else score = -1e6 end
    elseif aimMode == "health" then
        score = (100 - p.health) * 5 - d*0.1
    end
    return score, d
end

local function findTarget()
    local best = nil; local bestScore = -1e9; local bestDist = math.huge
    for _,p in ipairs(players) do
        if isValidTarget(p) then
            local s, d = scoreForTarget(p)
            if s > bestScore then bestScore = s; best = p; bestDist = d end
        end
    end
    return best, bestDist
end

local function predictPos(p, dt)
    local px = p.x + p.vx * predictionFactor * dt
    local py = p.y + p.vy * predictionFactor * dt
    -- simular "bone" (head offset)
    if aimBone == "head" then py = py + p.head_offset end
    return px, py
end

-- mover crosshair de forma suavizada
local function aimAt(target, dt)
    if not target then return end
    local px, py = predictPos(target, dt)
    local dx = px - crosshair.x; local dy = py - crosshair.y
    crosshair.x = crosshair.x + dx * smoothing
    crosshair.y = crosshair.y + dy * smoothing
end

-- "disparo" simulado (quando crosshair muito perto do alvo)
local firedCooldown = 0
local firedInterval = 0.5 -- segundos entre "disparos" simulados
local function tryFire(target, distToTarget)
    if not target then return end
    firedCooldown = math.max(0, firedCooldown - love.timer.getDelta())
    if firedCooldown <= 0 and distToTarget <= 12 + target.radius then
        firedCooldown = firedInterval
        -- simula acerto reduzindo HP (não afeta outros sistemas)
        target.health = math.max(0, target.health - math.random(8,35))
        logShot(target, distToTarget)
    end
end

-- LOVE2D callbacks
function love.load()
    love.window.setMode(WINDOW_W, WINDOW_H)
    love.window.setTitle("SIMULADOR AIMBOT - TESTES/DETECÇÃO (SAFE)")
    font = love.graphics.newFont(13)
    love.graphics.setFont(font)
    createMockPlayers(10)
end

function love.update(dt)
    -- mover mocks
    for _,p in ipairs(players) do
        p.x = p.x + p.vx * dt
        p.y = p.y + p.vy * dt
        if p.x < 20 then p.x=20; p.vx = -p.vx end
        if p.x > WINDOW_W-20 then p.x=WINDOW_W-20; p.vx = -p.vx end
        if p.y < 20 then p.y=20; p.vy = -p.vy end
        if p.y > WINDOW_H-20 then p.y=WINDOW_H-20; p.vy = -p.vy end
    end

    local target, d = findTarget()
    if aimActive and target then
        aimAt(target, dt)
        tryFire(target, d)
    else
        -- quando aim inativo, mira segue mouse
        local mx,my = love.mouse.getPosition()
        if mx and my then crosshair.x = mx; crosshair.y = my end
    end
end

function love.draw()
    love.graphics.clear(0.06,0.06,0.06)
    -- desenha jogadores
    for _,p in ipairs(players) do
        if p.team == localTeam then
            love.graphics.setColor(0.18,0.45,0.85,1)
        else
            love.graphics.setColor(0.85,0.15,0.15,1)
        end
        love.graphics.circle("fill", p.x, p.y, p.radius)
        -- draw head indicator
        love.graphics.setColor(1,1,1,0.12)
        love.graphics.circle("line", p.x, p.y + p.head_offset, p.radius*0.6)
        love.graphics.setColor(1,1,1,1)
        love.graphics.print(string.format("%s (HP:%d)", p.name, p.health), p.x - 28, p.y + p.radius + 4)
    end

    -- desenha FOV
    love.graphics.setColor(1,1,1,0.06)
    love.graphics.circle("fill", crosshair.x, crosshair.y, aimFOV)

    -- find and show current target
    local cur, d = findTarget()
    if cur then
        love.graphics.setColor(1,1,0,0.9)
        love.graphics.line(crosshair.x, crosshair.y, cur.x, cur.y)
        local px,py = predictPos(cur, 1)
        love.graphics.setColor(1,1,0,0.5)
        love.graphics.circle("line", px, py, 10)
        love.graphics.setColor(1,1,1)
        love.graphics.print("Alvo atual: "..cur.name.." ("..cur.team..")  Dist: "..string.format("%.1f", d), 10, 10)
    else
        love.graphics.setColor(1,1,1)
        love.graphics.print("Alvo atual: nenhum dentro do FOV / critérios", 10, 10)
    end

    -- crosshair
    love.graphics.setColor(1,1,1)
    love.graphics.setLineWidth(2)
    love.graphics.line(crosshair.x - 12, crosshair.y, crosshair.x + 12, crosshair.y)
    love.graphics.line(crosshair.x, crosshair.y - 12, crosshair.x, crosshair.y + 12)
    love.graphics.circle("line", crosshair.x, crosshair.y, 8)

    -- HUD
    love.graphics.setColor(0,0,0,0.6)
    love.graphics.rectangle("fill", 10, WINDOW_H - 180, 420, 170, 8)
    love.graphics.setColor(1,1,1)
    love.graphics.print("CONTROLES E STATUS:", 20, WINDOW_H - 170)
    love.graphics.print("SPACE : Toggle Aim (atual: "..(aimActive and "ON" or "OFF")..")", 20, WINDOW_H - 150)
    love.graphics.print("T : Toggle Team Check (atual: "..(teamCheck and "ON" or "OFF")..")", 20, WINDOW_H - 128)
    love.graphics.print("M : Alternar aimBone (head/body) (atual: "..aimBone..")", 20, WINDOW_H - 106)
    love.graphics.print("1/2 : Mudar aimMode (1:closest, 2:fov_closest, 3:health) (atual: "..aimMode..")", 20, WINDOW_H - 84)
    love.graphics.print("S/A : aumentar/diminuir smoothing (atual: "..string.format("%.3f",smoothing)..")", 20, WINDOW_H - 62)
    love.graphics.print("F/G : aumentar/diminuir FOV (atual: "..aimFOV..")", 20, WINDOW_H - 40)
    love.graphics.print("P : alterar predictionFactor (cycle)  |  R : reset mocks  |  L : salvar log CSV", 20, WINDOW_H - 18)
end

function love.keypressed(k)
    if k == aimKey then aimActive = not aimActive
    elseif k == "t" then teamCheck = not teamCheck
    elseif k == "m" then aimBone = (aimBone=="head") and "body" or "head"
    elseif k == "s" then smoothing = math.min(0.95, smoothing + 0.02)
    elseif k == "a" then smoothing = math.max(0, smoothing - 0.02)
    elseif k == "f" then aimFOV = math.min(800, aimFOV + 10)
    elseif k == "g" then aimFOV = math.max(20, aimFOV - 10)
    elseif k == "r" then createMockPlayers(10)
    elseif k == "1" then aimMode = "closest"
    elseif k == "2" then aimMode = "fov_closest"
    elseif k == "3" then aimMode = "health"
    elseif k == "p" then
        predictionFactor = predictionFactor + 0.05
        if predictionFactor > 1 then predictionFactor = 0.0 end
    elseif k == "l" then
        local ok, name = saveLogCSV()
        if ok then print("Log salvo: "..name) else print("Log não salvo: "..name) end
    elseif k == "escape" then love.event.quit()
    end
end
