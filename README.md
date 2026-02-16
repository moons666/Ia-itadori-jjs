-- IA PLAYER Jujutsu Style
-- Coloque este script em ServerScriptService

local Players = game:GetService("Players")
local PathfindingService = game:GetService("PathfindingService")
local RunService = game:GetService("RunService")
local ReplicatedStorage = game:GetService("ReplicatedStorage")

-- CONFIG
local DETECTION_RANGE = 60
local ATTACK_RANGE = 6
local ATTACK_COOLDOWN = 1.2

-- Remotes (ajuste os nomes se forem diferentes no seu jogo)
local Remotes = ReplicatedStorage:WaitForChild("Remotes")
local BasicAttack = Remotes:WaitForChild("BasicAttack")
local Skill1 = Remotes:WaitForChild("Skill1")
local Skill2 = Remotes:WaitForChild("Skill2")

-- Cooldowns
local lastAttack = 0
local skills = {
	Skill1 = {cooldown = 6, lastUse = 0},
	Skill2 = {cooldown = 12, lastUse = 0}
}

-- Acha player mais próximo
local function getNearestPlayer(botChar)
	local nearest = nil
	local shortest = DETECTION_RANGE

	for _, player in pairs(Players:GetPlayers()) do
		if player.Character
			and player.Character ~= botChar
			and player.Character:FindFirstChild("HumanoidRootPart")
			and player.Character:FindFirstChild("Humanoid")
			and player.Character.Humanoid.Health > 0 then

			local dist = (player.Character.HumanoidRootPart.Position
				- botChar.HumanoidRootPart.Position).Magnitude

			if dist < shortest then
				shortest = dist
				nearest = player.Character
			end
		end
	end

	return nearest, shortest
end

-- Movimento até o alvo
local function moveToTarget(botChar, targetPos)
	local humanoid = botChar:FindFirstChild("Humanoid")
	if not humanoid then return end

	local path = PathfindingService:CreatePath()
	path:ComputeAsync(botChar.HumanoidRootPart.Position, targetPos)

	for _, waypoint in ipairs(path:GetWaypoints()) do
		if humanoid.Health <= 0 then break end
		humanoid:MoveTo(waypoint.Position)
		humanoid.MoveToFinished:Wait()
	end
end

-- Ataque básico
local function attack()
	if tick() - lastAttack < ATTACK_COOLDOWN then return end
	lastAttack = tick()
	BasicAttack:FireServer()
end

-- Skills
local function useSkill(name)
	local skill = skills[name]
	if tick() - skill.lastUse >= skill.cooldown then
		skill.lastUse = tick()
		Remotes[name]:FireServer()
	end
end

-- Inicializa IA no personagem
local function startAI(character)
	local humanoid = character:WaitForChild("Humanoid")
	local root = character:WaitForChild("HumanoidRootPart")

	task.spawn(function()
		while humanoid.Health > 0 do
			task.wait(0.2)

			local target, dist = getNearestPlayer(character)

			if target then
				if dist > ATTACK_RANGE then
					moveToTarget(character, target.HumanoidRootPart.Position)
				else
					-- Decide ação
					local chance = math.random(1,100)

					if chance <= 50 then
						attack()
					elseif chance <= 80 then
						useSkill("Skill1")
					else
						useSkill("Skill2")
					end
				end
			end
		end
	end)
end

-- Detecta players bots
Players.PlayerAdded:Connect(function(player)
	player.CharacterAdded:Connect(function(character)
		if player:GetAttribute("Bot") == true then
			startAI(character)
		end
	end)
end)

-- Caso o bot já esteja no jogo
for _, player in pairs(Players:GetPlayers()) do
	if player:GetAttribute("Bot") == true and player.Character then
		startAI(player.Character)
	end
end
