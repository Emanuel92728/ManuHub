local Players = game:GetService("Players")
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local Workspace = game:GetService("Workspace")

-- Cria pasta Assets se não existir (onde você pode colocar modelos de casas/veículos)
if not Workspace:FindFirstChild("Assets") then
	local assets = Instance.new("Folder")
	assets.Name = "Assets"
	assets.Parent = Workspace
end

-- Cria RemoteEvents em ReplicatedStorage/RemoteEvents
local remotesFolder = ReplicatedStorage:FindFirstChild("RemoteEvents")
if not remotesFolder then
	remotesFolder = Instance.new("Folder")
	remotesFolder.Name = "RemoteEvents"
	remotesFolder.Parent = ReplicatedStorage
end

local function getOrCreateRemote(name)
	local r = remotesFolder:FindFirstChild(name)
	if not r then
		r = Instance.new("RemoteEvent")
		r.Name = name
		r.Parent = remotesFolder
	end
	return r
end

local RequestSpawnHouse = getOrCreateRemote("RequestSpawnHouse")
local RequestEnterVehicle = getOrCreateRemote("RequestEnterVehicle")
local RequestShowMessage = getOrCreateRemote("RequestShowMessage") -- usado para mostrar mensagens no cliente

-- ======= Dados & Managers (server-side) =======

local PlayerManager = {}
PlayerManager._data = {} -- dados por userId

function PlayerManager:InitPlayer(player)
	local userId = player.UserId
	self._data[userId] = {
		Money = 1000,
		HousesOwned = {}
	}
	-- leaderstats (visível no cliente)
	local leaderstats = Instance.new("Folder")
	leaderstats.Name = "leaderstats"
	leaderstats.Parent = player

	local money = Instance.new("IntValue")
	money.Name = "Money"
	money.Value = self._data[userId].Money
	money.Parent = leaderstats
end

function PlayerManager:GetData(player)
	return self._data[player.UserId]
end

function PlayerManager:AddMoney(player, amount)
	local d = self._data[player.UserId]
	if not d then return end
	d.Money = d.Money + amount
	if player:FindFirstChild("leaderstats") and player.leaderstats:FindFirstChild("Money") then
		player.leaderstats.Money.Value = d.Money
	end
end

function PlayerManager:CanAfford(player, amount)
	local d = self._data[player.UserId]
	if not d then return false end
	return d.Money >= amount
end

function PlayerManager:SaveAndCleanup(player)
	self._data[player.UserId] = nil
end

-- Simples tabela de casas (IDs, nome, preço, nome do modelo)
local Houses = {
	{Id = 1, Name = "Casa Pequena", Price = 500, ModelName = "exampleHouseModel_small"},
	{Id = 2, Name = "Casa Média",  Price = 1500, ModelName = "exampleHouseModel_medium"},
	{Id = 3, Name = "Casa Grande", Price = 3000, ModelName = "exampleHouseModel_large"},
}

local HouseManager = {}

function HouseManager:GetHouseById(id)
	for _,h in ipairs(Houses) do
		if h.Id == id then return h end
	end
	return nil
end

-- Spawn house server-side com validação
function HouseManager:SpawnHouseForPlayer(player, id)
	local houseInfo = self:GetHouseById(id)
	if not houseInfo then
		RequestShowMessage:FireClient(player, "Casa inválida.")
		return
	end

	if not PlayerManager:CanAfford(player, houseInfo.Price) then
		RequestShowMessage:FireClient(player, "Dinheiro insuficiente para comprar " .. houseInfo.Name)
		return
	end

	-- deduz dinheiro server-side
	PlayerManager:AddMoney(player, -houseInfo.Price)

	-- procura o modelo em Workspace.Assets (você deve adicionar modelos lá)
	local asset = Workspace:FindFirstChild("Assets") and Workspace.Assets:FindFirstChild(houseInfo.ModelName)
	if not asset then
		RequestShowMessage:FireClient(player, "Modelo da casa não encontrado em Workspace.Assets (adicione um modelo chamado: "..houseInfo.ModelName..")")
		return
	end

	local clone = asset:Clone()
	clone.Name = player.Name .. "_House_" .. id
	clone.Parent = Workspace

	-- posicionamento simples: espalha pelo eixo X usando UserId para evitar sobreposição
	local offsetX = 60 * (player.UserId % 10)
	local basePos = Vector3.new(0, 0, 0) + Vector3.new(offsetX, 0, 0)
	if clone.PrimaryPart then
		clone:SetPrimaryPartCFrame(CFrame.new(basePos + Vector3.new(0, clone:GetExtentsSize().Y/2, 0)))
	else
		-- tenta definir PrimaryPart se houver
		local part = clone:FindFirstChildWhichIsA("BasePart", true)
		if part then
			clone.PrimaryPart = part
			clone:SetPrimaryPartCFrame(CFrame.new(basePos + Vector3.new(0, clone:GetExtentsSize().Y/2, 0)))
		end
	end

	-- registra propriedade
	local d = PlayerManager:GetData(player)
	table.insert(d.HousesOwned, clone.Name)

	RequestShowMessage:FireClient(player, "Parabéns! Você comprou: "..houseInfo.Name)
end

-- VehicleManager simples
local VehicleManager = {}

function VehicleManager:TryEnterVehicle(player, vehicle)
	if not vehicle or not vehicle:IsA("Instance") then return end
	local seat = vehicle:FindFirstChildWhichIsA("VehicleSeat", true)
	if not seat then
		RequestShowMessage:FireClient(player, "Veículo inválido.")
		return
	end
	if seat.Occupant then
		RequestShowMessage:FireClient(player, "Assento já ocupado.")
		return
	end

	-- transporta jogador para o assento
	if player.Character and player.Character:FindFirstChild("HumanoidRootPart") then
		player.Character.HumanoidRootPart.CFrame = seat.CFrame + Vector3.new(0, 3, 0)
		-- humanoid deve sentar automaticamente ao entrar no VehicleSeat
	end
end

-- ======= Bind RemoteEvents (server authoritative) =======

RequestSpawnHouse.OnServerEvent:Connect(function(player, houseId)
	if typeof(houseId) ~= "number" then
		RequestShowMessage:FireClient(player, "ID de casa inválido.")
		return
	end
	HouseManager:SpawnHouseForPlayer(player, houseId)
end)

RequestEnterVehicle.OnServerEvent:Connect(function(player, vehicleInstance)
	-- segurança: só permita se for uma instância válida do Workspace (modelo)
	if typeof(vehicleInstance) ~= "Instance" then return end
	VehicleManager:TryEnterVehicle(player, vehicleInstance)
end)

-- ======= Cliente: código injetado =======
-- Será colocado num LocalScript dentro de PlayerGui para cada jogador
local clientCode = [[
-- LocalScript gerado automaticamente pelo servidor (Brookhaven-Lite client)
local Players = game:GetService("Players")
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local UserInputService = game:GetService("UserInputService")
local player = Players.LocalPlayer
local mouse = player:GetMouse()

local remotes = ReplicatedStorage:WaitForChild("RemoteEvents")
local RequestSpawnHouse = remotes:WaitForChild("RequestSpawnHouse")
local RequestEnterVehicle = remotes:WaitForChild("RequestEnterVehicle")
local RequestShowMessage = remotes:WaitForChild("RequestShowMessage")

-- Função para criar um HUD simples (MoneyLabel + botões de compra)
local function createHUD()
	local screenGui = Instance.new("ScreenGui")
	screenGui.Name = "RP_HUD"
	screenGui.ResetOnSpawn = false
	screenGui.Parent = player:WaitForChild("PlayerGui")

	-- Money label (atualizado automaticamente pelo leaderstats mostrado na tela)
	local moneyLabel = Instance.new("TextLabel")
	moneyLabel.Name = "MoneyLabel"
	moneyLabel.Size = UDim2.new(0,200,0,40)
	moneyLabel.Position = UDim2.new(0,10,0,10)
	moneyLabel.BackgroundTransparency = 0.3
	moneyLabel.TextScaled = true
	moneyLabel.Text = "Dinheiro: Carregando..."
	moneyLabel.Parent = screenGui

	-- Atualiza a label lendo leaderstats (se existir)
	local function updateMoney()
		local ls = player:FindFirstChild("leaderstats")
		if ls and ls:FindFirstChild("Money") then
			moneyLabel.Text = "Dinheiro: $" .. tostring(ls.Money.Value)
			ls.Money:GetPropertyChangedSignal("Value"):Connect(function()
				moneyLabel.Text = "Dinheiro: $" .. tostring(ls.Money.Value)
			end)
		end
	end
	spawn(updateMoney)

	-- House menu simples (botões)
	local houseFrame = Instance.new("Frame")
	houseFrame.Name = "HouseMenu"
	houseFrame.Size = UDim2.new(0,220,0,160)
	houseFrame.Position = UDim2.new(0,10,0,60)
	houseFrame.BackgroundTransparency = 0.5
	houseFrame.Parent = screenGui

	local title = Instance.new("TextLabel")
	title.Size = UDim2.new(1,0,0,30)
	title.Position = UDim2.new(0,0,0,0)
	title.Text = "Comprar Casas"
	title.Parent = houseFrame
	title.TextScaled = true

	-- botões para cada casa (IDs hardcoded para combinar com server)
	local houses = {
		{Id=1, Name="Casa Pequena", Price=500},
		{Id=2, Name="Casa Média", Price=1500},
		{Id=3, Name="Casa Grande", Price=3000},
	}
	for i,h in ipairs(houses) do
		local btn = Instance.new("TextButton")
		btn.Size = UDim2.new(1,-10,0,36)
		btn.Position = UDim2.new(0,5,0,30 + (i-1)*40)
		btn.Text = h.Name.." - $"..h.Price
		btn.Parent = houseFrame
		btn.TextScaled = true
		btn.MouseButton1Click:Connect(function()
			RequestSpawnHouse:FireServer(h.Id)
		end)
	end
end

-- Função que mostra mensagem simples (do servidor)
RequestShowMessage.OnClientEvent:Connect(function(text)
	-- Cria um pequeno Message GUI temporário
	local screenGui = player:FindFirstChild("PlayerGui") and player.PlayerGui:FindFirstChild("RP_HUD")
	if not screenGui then
		-- cria se não existir
		createHUD()
		screenGui = player.PlayerGui:FindFirstChild("RP_HUD")
	end
	local msg = Instance.new("TextLabel")
	msg.Size = UDim2.new(0,300,0,40)
	msg.Position = UDim2.new(0.5,-150,0.2,0)
	msg.BackgroundTransparency = 0.3
	msg.TextScaled = true
	msg.Text = text
	msg.Parent = player.PlayerGui
	delay(3, function() msg:Destroy() end)
end)

-- Atalho: pressione H para pedir casa com id=1 (exemplo)
UserInputService.InputBegan:Connect(function(input, gp)
	if gp then return end
	if input.KeyCode == Enum.KeyCode.H then
		RequestSpawnHouse:FireServer(1)
	end
end)

-- Clique com o mouse em veículo para entrar
mouse.Button1Down:Connect(function()
	local target = mouse.Target
	if not target then return end
	local model = target:FindFirstAncestorOfClass("Model")
	if not model then return end
	-- envia o modelo para o servidor para tentar entrar
	RequestEnterVehicle:FireServer(model)
end)

-- Inicializa HUD
createHUD()
]]

-- ======= PlayerAdded: inicializa dados e injeta LocalScript cliente =======
Players.PlayerAdded:Connect(function(player)
	-- inicializa dados server-side
	PlayerManager:InitPlayer(player)

	-- cria LocalScript com o código cliente e parent em PlayerGui
	-- (LocalScripts colocados em PlayerGui rodam no cliente)
	local success, err = pcall(function()
		-- espera PlayerGui existir
		local playerGui = player:WaitForChild("PlayerGui", 10)
		if not playerGui then return end

		-- cria um LocalScript e define Source para o clientCode
		local ls = Instance.new("LocalScript")
		ls.Name = "BrookhavenLite_Client"
		ls.Source = clientCode
		ls.Parent = playerGui
	end)

	if not success then
		warn("Erro ao injetar LocalScript no jogador:", err)
	end
end)

Players.PlayerRemoving:Connect(function(player)
	PlayerManager:SaveAndCleanup(player)
end)

