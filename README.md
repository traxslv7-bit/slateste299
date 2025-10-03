--[[ Simulação de ESP para Teste de Devs
     NÃO coleta dados de players reais.
     Apenas mostra como um ESP se comportaria.
--]]

local Players = {
    {Name = "Jogador1", Team = "Azul", X = 50, Y = 100},
    {Name = "Jogador2", Team = "Vermelho", X = 150, Y = 200},
    {Name = "Jogador3", Team = "Azul", X = 250, Y = 300}
}

local LocalPlayerTeam = "Azul"

-- Função de "render" para simular o ESP
local function RenderESP()
    for _, player in ipairs(Players) do
        local color = (player.Team == LocalPlayerTeam) and "Verde (Aliado)" or "Vermelho (Inimigo)"
        print("["..color.."] "..player.Name.." - Posição: ("..player.X..","..player.Y..")")
    end
end

-- Interface simples (console)
print("=== ESP SIMULADO - TESTE DE TEAM CHECK ===")
RenderESP()
