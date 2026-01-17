-- LocalScript (colocar em StarterPlayerScripts)
local Players = game:GetService("Players")
local UserInputService = game:GetService("UserInputService")
local LocalPlayer = Players.LocalPlayer
local toggle = true

-- Gera um nome único de GUI para a personagem (usa UserId do player quando existir)
local function uiNameForCharacter(character)
	local p = Players:GetPlayerFromCharacter(character)
	if p then
		return "RP_" .. tostring(p.UserId)
	else
		-- NPCs/Models: usa o próprio nome (ok para maioria dos casos)
		return "RP_" .. character.Name
	end
end

-- cria o ScreenGui para essa personagem (somente 1 GUI por personagem no PlayerGui)
local function mkBBGForCharacter(character)
	if not LocalPlayer or not LocalPlayer:FindFirstChild("PlayerGui") then return end
	local guiName = uiNameForCharacter(character)
	local pg = LocalPlayer.PlayerGui

	if pg:FindFirstChild(guiName) then
		return pg[guiName]
	end

	local g = Instance.new("ScreenGui")
	g.Name = guiName
	g.ResetOnSpawn = false
	g.Parent = pg

	local bg = Instance.new("Frame")
	bg.Name = "BG"
	bg.Size = UDim2.new(0, 90, 0, 35)
	-- posição no cantinho esquerdo (pode ajustar se quiser)
	bg.Position = UDim2.new(0, 10, 0, 10)
	bg.AnchorPoint = Vector2.new(0, 0)
	bg.BackgroundColor3 = Color3.fromRGB(30,30,30)
	bg.BackgroundTransparency = 0.25
	bg.BorderSizePixel = 0
	bg.Parent = g

	local corner = Instance.new("UICorner")
	corner.CornerRadius = UDim.new(0,6)
	corner.Parent = bg

	local title = Instance.new("TextLabel")
	title.Name = "Title"
	title.Size = UDim2.new(1,0,0.5,0)
	title.Position = UDim2.new(0,0,0,0)
	title.BackgroundTransparency = 1
	title.Text = "RUNNER"
	title.Font = Enum.Font.GothamBold
	title.TextColor3 = Color3.new(1,1,1)
	title.TextScaled = true
	title.Parent = bg

	local percent = Instance.new("TextLabel")
	percent.Name = "PercentLabel"
	percent.Size = UDim2.new(1,0,0.5,0)
	percent.Position = UDim2.new(0,0,0.5,0)
	percent.BackgroundTransparency = 1
	percent.TextColor3 = Color3.new(1,1,1)
	percent.TextScaled = true
	percent.Font = Enum.Font.Cartoon
	percent.Text = "0%"
	percent.Parent = bg

	return g
end

local function rmBBGForCharacter(character)
	if not LocalPlayer or not LocalPlayer:FindFirstChild("PlayerGui") then return end
	local guiName = uiNameForCharacter(character)
	local pg = LocalPlayer.PlayerGui
	local g = pg:FindFirstChild(guiName)
	if g then g:Destroy() end
end

-- Toggle com V (esconde/mostra todas as GUIs criadas)
UserInputService.InputBegan:Connect(function(input, gameProcessed)
	if not gameProcessed and input.KeyCode == Enum.KeyCode.V then
		toggle = not toggle
		if LocalPlayer and LocalPlayer:FindFirstChild("PlayerGui") then
			for _, child in ipairs(LocalPlayer.PlayerGui:GetChildren()) do
				if type(child.Name) == "string" and child.Name:match("^RP_") then
					child.Enabled = toggle
				end
			end
		end
	end
end)

-- Função principal: mantém a mesma lógica do script base por personagem
local function setupBP(character)
	local hum = character:FindFirstChildOfClass("Humanoid")
	if not hum then return end

	-- cria label se BeastPowers já existir
	local function createLabel()
		local bp = character:FindFirstChild("BeastPowers")
		if bp then
			mkBBGForCharacter(character)
		end
	end

	local function removeLabel()
		rmBBGForCharacter(character)
	end

	local bp = character:FindFirstChild("BeastPowers")
	if bp then createLabel() end

	character.ChildAdded:Connect(function(child)
		if child and child.Name == "BeastPowers" then
			bp = child
			createLabel()
		end
	end)

	character.ChildRemoved:Connect(function(child)
		if child and child.Name == "BeastPowers" then
			bp = nil
			removeLabel()
		end
	end)

	-- loop de atualização: NÃO recria a GUI aqui, apenas atualiza o TextLabel existente
	task.spawn(function()
		while character.Parent do
			task.wait(0.1)
			bp = character:FindFirstChild("BeastPowers")
			if not bp then
				-- se não houver, remove a GUI (comportamento igual ao script base)
				removeLabel()
				continue
			end

			local v = bp:FindFirstChild("PowerProgressPercent")
			local pg = LocalPlayer:FindFirstChild("PlayerGui")
			local guiName = uiNameForCharacter(character)
			local g = pg and pg:FindFirstChild(guiName)
			local l = g and g:FindFirstChild("PercentLabel", true)

			-- Se GUI não existir, cria (apenas uma vez)
			if not g then
				mkBBGForCharacter(character)
				g = pg and pg:FindFirstChild(guiName)
				l = g and g:FindFirstChild("PercentLabel", true)
			end

			if l and g then
				g.Enabled = toggle
				if v and type(v.Value) == "number" then
					-- Normaliza valores 0..1 -> 0..100, se já estiver 0..100 usa direto
					local val = v.Value
					if val <= 1 then val = val * 100 end
					l.Text = tostring(math.floor(val)) .. "%"
				else
					l.Text = "0%"
				end
			end
		end
	end)
end

-- Conecta para todos os players como no script base
Players.PlayerAdded:Connect(function(player)
	player.CharacterAdded:Connect(setupBP)
end)

for _, player in ipairs(Players:GetPlayers()) do
	if player.Character then
		setupBP(player.Character)
	end
	player.CharacterAdded:Connect(setupBP)
end
