-- main.lua
-- SIMULADOR + ANTI-HIT DETECTOR para TESTES/DETECÇÃO (Love2D)
-- ATENÇÃO: Não é cheat. NÃO interage com jogos reais.
-- Gera logs CSV com eventos suspeitos para análise.

math.randomseed(os.time())

-- CONFIG
local W, H = 900, 650
local FONT_SIZE = 13

-- thresholds (ajustáveis via HUD)
local MAX_HUMAN_SPEED = 300        -- px/s (velocidade máxima plausível)
local MAX_TELEPORT_DIST = 200     -- px em um frame = teleport
local MIN_REACTION_TIME = 0.06    -- s (reação humana mínima plausível)
local MAX_ANGULAR_SPEED = 720     -- deg/s (mudança angular instantânea suspeita)

-- simulação de players e tiros
local players = {}
local bullets = {}
local localPlayer = { x = W/2, y = H/2, lastShotTime = -5 }
local numPlayers = 10

-- logs
local suspiciousLogs = {}

-- util
local function dist(x1,y1,x2,y2) local dx=x1-x2; local dy=y1-y2; return math.sqrt(dx*dx+dy*dy) end
local function angleBetween(x1,y1,x2,y2) return math.deg(math.atan2(y2-y1, x2-x1)) end

-- cria mocks
local function createPlayers(n)
    players = {}
    local teams = {"Azul","Vermelho"}
    for i=1,n do
        local p = {
            id = i,
            name = "P"..i,
            x = math.random(40, W-40),
            y = math.random(40, H-40),
            vx = math.random(-120,120),
            vy = math.random(-120,120),
            radius = math.random(10,14),
            team = (math.random() < 0.5) and teams[1] or teams[2],
            lastX = 0, lastY = 0,
            lastUpdate = love.timer.getTime(),
            lastAngle = 0,
            lastSeenTime = love.timer.getTime(),
            lastDodgeTime = -10,
            hp = 100,
        }
        p.lastX, p.lastY = p.x, p.y
        p.lastAngle = angleBetween(localPlayer.x, localPlayer.y, p.x, p.y)
        table.insert(players, p)
    end
end

-- criar bala (simples)
local function fireBullet(fromX, fromY, toX, toY)
    local dx, dy = toX - fromX, toY - fromY
    local ang = math.atan2(dy, dx)
    local speed = 700
    table.insert(bullets, { x = fromX, y = fromY, vx = math.cos(ang)*speed, vy = math.sin(ang)*speed, spawn = love.timer.getTime() })
    localPlayer.lastShotTime = love.timer.getTime()
end

-- detector: analisa cada player e registra comportamento suspeito
local function analyzePlayer(p, dt)
    local now = love.timer.getTime()
    -- velocidade estimada (entre frames)
    local dx = p.x - p.lastX
    local dy = p.y - p.lastY
    local frameDt = now - p.lastUpdate
    local spd = 0
    if frameDt > 0 then spd = math.sqrt(dx*dx + dy*dy) / frameDt end

    if spd > MAX_HUMAN_SPEED then
        table.insert(suspiciousLogs, string.format("%f,ID_%d,HIGH_SPEED,%.2f,thr=%.1f", now, p.id, spd, MAX_HUMAN_SPEED))
    end

    -- teleport detection: movimento muito grande num frame
    local frameDist = math.sqrt(dx*dx + dy*dy)
    if frameDist > MAX_TELEPORT_DIST then
        table.insert(suspiciousLogs, string.format("%f,ID_%d,TELEPORT,dist=%.1f,thr=%.1f", now, p.id, frameDist, MAX_TELEPORT_DIST))
    end

    -- mudança angular (olhando a direção local->player)
    local ang = angleBetween(localPlayer.x, localPlayer.y, p.x, p.y)
    local deltaAng = math.abs((ang - p.lastAngle + 180) % 360 - 180) -- diferença mínima
    local angSpeed = 0
    if frameDt > 0 then angSpeed = deltaAng / frameDt end
    if angSpeed > MAX_ANGULAR_SPEED then
        table.insert(suspiciousLogs, string.format("%f,ID_%d,HIGH_ANGULAR_SPEED,deg/s=%.1f,thr=%.1f", now, p.id, angSpeed, MAX_ANGULAR_SPEED))
    end

    -- reaction/dodge detection: se uma bala passa perto e o player mudou movimento extremamente rápido
    for i=#bullets,1,-1 do
        local b = bullets[i]
        local bdist = dist(b.x, b.y, p.x, p.y)
        if bdist < p.radius + 6 then
            -- bala quase acerta: verificar tempo desde última manobra brusca
            local timeSinceLastDodge = now - p.lastDodgeTime
            if timeSinceLastDodge < MIN_REACTION_TIME then
                table.insert(suspiciousLogs, string.format("%f,ID_%d,INSTANT_DODGE,delta=%.3f,thr=%.3f", now, p.id, timeSinceLastDodge, MIN_REACTION_TIME))
            end
            -- simula que bala não causou dano (apenas para teste)
        end
    end

    -- update histórico
    p.lastX, p.lastY = p.x, p.y
    p.lastUpdate = now
    p.lastAngle = ang
end

-- função para forçar um "dodge" (usada na simulação pra testar detector)
local function forceDodge(p)
    -- muda velocidade bruscamente
    p.vx = p.vx + math.random(-600,600)
    p.vy = p.vy + math.random(-600,600)
    p.lastDodgeTime = love.timer.getTime()
end

-- salvar logs CSV
local function saveLogs()
    if #suspiciousLogs == 0 then return false, "Nenhum evento suspeito" end
    local header = "timestamp,event\n"
    local content = header .. table.concat(suspiciousLogs, "\n")
    local filename = "anti_hit_det_logs_" .. os.time() .. ".csv"
    love.filesystem.write(filename, content)
    return true, filename
end

-- LOVE2D callbacks
function love.load()
    love.window.setMode(W, H)
    love.window.setTitle("SIMULADOR ANTI-HIT (DETECÇÃO) - LOVE2D")
    font = love.graphics.newFont(FONT_SIZE)
    love.graphics.setFont(font)
    createPlayers(numPlayers)
end

function love.update(dt)
    local now = love.timer.getTime()

    -- atualizar players
    for _,p in ipairs(players) do
        -- movimento simples com bounce
        p.x = p.x + p.vx * dt
        p.y = p.y + p.vy * dt
        if p.x < 20 then p.x=20; p.vx = -p.vx end
        if p.x > W-20 then p.x=W-20; p.vx = -p.vx end
        if p.y < 20 then p.y=20; p.vy = -p.vy end
        if p.y > H-20 then p.y=H-20; p.vy = -p.vy end

        -- ocasionalmente gerar um dodge brusco aleatório (para testar detector)
        if math.random() < 0.002 then forceDodge(p) end

        analyzePlayer(p, dt)
    end

    -- atualizar balas
    for i=#bullets,1,-1 do
        local b = bullets[i]
        b.x = b.x + b.vx * dt
        b.y = b.y + b.vy * dt
        if b.x < -50 or b.x > W+50 or b.y < -50 or b.y > H+50 then
            table.remove(bullets, i)
        end
    end
end

function love.draw()
    love.graphics.clear(0.08,0.08,0.08)
    -- desenhar jogadores
    for _,p in ipairs(players) do
        if p.team == "Azul" then love.graphics.setColor(0.2,0.45,0.9) else love.graphics.setColor(0.9,0.2,0.2) end
        love.graphics.circle("fill", p.x, p.y, p.radius)
        love.graphics.setColor(1,1,1)
        love.graphics.print(string.format("%s (ID:%d) HP:%d", p.name, p.id, p.hp), p.x - 30, p.y + p.radius + 4)
    end

    -- desenhar balas
    love.graphics.setColor(1,0.9,0.3)
    for _,b in ipairs(bullets) do
        love.graphics.circle("fill", b.x, b.y, 4)
    end

    -- desenhar local player
    love.graphics.setColor(0.9,0.9,0.9)
    love.graphics.circle("line", localPlayer.x, localPlayer.y, 14)
    love.graphics.print("LOCAL (clicar para atirar)", localPlayer.x - 48, localPlayer.y + 18)

    -- debug + HUD
    love.graphics.setColor(0,0,0,0.6)
    love.graphics.rectangle("fill", 10, H - 200, 540, 190, 8)
    love.graphics.setColor(1,1,1)
    love.graphics.print("CONTROLES:", 18, H - 190)
    love.graphics.print("Clique esquerdo : atirar (simulado)", 18, H - 168)
    love.graphics.print("R : reiniciar players    D : forçar dodge aleatório em todos (test)", 18, H - 148)
    love.graphics.print("S : salvar logs CSV    + / - : ajustar MAX_HUMAN_SPEED (atual: "..tostring(MAX_HUMAN_SPEED)..")", 18, H - 126)
    love.graphics.print("T : ajustar MAX_TELEPORT_DIST (atual: "..tostring(MAX_TELEPORT_DIST)..")  U : ajustar MIN_REACTION_TIME (atual: "..string.format("%.3f", MIN_REACTION_TIME)..")", 18, H - 104)
    love.graphics.print("Logs detectados (ultimos 8):", 18, H - 82)

    -- mostrar últimos logs
    local start = math.max(1, #suspiciousLogs - 7)
    for i = start, #suspiciousLogs do
        love.graphics.print(suspiciousLogs[i], 18, H - 82 + 14 * (i - start + 1))
    end
end

function love.mousepressed(x,y,button)
    if button == 1 then
        -- atira em direção ao mouse
        fireBullet(localPlayer.x, localPlayer.y, x, y)
    end
end

function love.keypressed(k)
    if k == "r" then createPlayers(numPlayers); suspiciousLogs = {} ; bullets = {} 
    elseif k == "d" then for _,p in ipairs(players) do forceDodge(p) end
    elseif k == "s" then local ok, name = saveLogs(); if ok then print("Logs salvos: "..name) else print(name) end
    elseif k == "+" or k == "=" then MAX_HUMAN_SPEED = MAX_HUMAN_SPEED + 50
    elseif k == "-" then MAX_HUMAN_SPEED = math.max(50, MAX_HUMAN_SPEED - 50)
    elseif k == "t" then MAX_TELEPORT_DIST = MAX_TELEPORT_DIST + 20
    elseif k == "u" then MIN_REACTION_TIME = math.max(0.01, MIN_REACTION_TIME - 0.01)
    elseif k == "i" then MIN_REACTION_TIME = MIN_REACTION_TIME + 0.01
    elseif k == "escape" then love.event.quit() end
end
