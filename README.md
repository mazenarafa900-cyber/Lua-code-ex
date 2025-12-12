local Players = game:GetService("Players")
local DataStoreService = game:GetService("DataStoreService")
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local RunService = game:GetService("RunService")

local MoneyStore = DataStoreService:GetDataStore("playermoney")
local InventoryStore = DataStoreService:GetDataStore("playerinventory")
local event = ReplicatedStorage:WaitForChild("core")
local TOOL_NAME = "Sword"
local COOLDOWN_TIME = 5
local MAX_MONEY_PER_EVENT = 500
local AUTO_SAVE_INTERVAL = 60

local multipliers = {}
local cooldowns = {}
local dailyClaimed = {}

-- Load player money
local function loadPlayerData(player)
	local success, data = pcall(function()
		return MoneyStore:GetAsync(player.UserId)
	end)
	if success then
		return data or 0
	else
		return 0
	end
end

-- Save player money
local function savePlayerData(player, value)
	pcall(function()
		MoneyStore:SetAsync(player.UserId, value)
	end)
end

-- Load inventory
local function loadInventory(player)
	local success, data = pcall(function()
		return InventoryStore:GetAsync(player.UserId)
	end)
	if success then
		return data or {}
	else
		return {}
	end
end

-- Save inventory
local function saveInventory(player, inventory)
	pcall(function()
		InventoryStore:SetAsync(player.UserId, inventory)
	end)
end

-- Check cooldown
local function canUse(player)
	if not cooldowns[player.UserId] then
		return true
	end
	if os.time() >= cooldowns[player.UserId] then
		return true
	end
	return false
end

local function setCooldown(player)
	cooldowns[player.UserId] = os.time() + COOLDOWN_TIME
end

-- Give tool
local function givePlayerTool(player, toolName)
	local backpack = player:FindFirstChild("Backpack")
	if not backpack then return end
	local tool = ReplicatedStorage:FindFirstChild(toolName)
	if tool then
		tool:Clone().Parent = backpack
		local inventory = loadInventory(player)
		table.insert(inventory, toolName)
		saveInventory(player, inventory)
	end
end

local function applyMultiplier(player, amount)
	return amount * (multipliers[player.UserId] or 1)
end

local function addMoney(player, amount)
	amount = math.min(amount, MAX_MONEY_PER_EVENT)
	player.leaderstats.Money.Value = player.leaderstats.Money.Value + applyMultiplier(player, amount)
	savePlayerData(player, player.leaderstats.Money.Value)
end

local function removeMoney(player, amount)
	amount = math.min(amount, player.leaderstats.Money.Value)
	player.leaderstats.Money.Value = player.leaderstats.Money.Value - amount
	savePlayerData(player, player.leaderstats.Money.Value)
end

local function resetMultiplier(player)
	multipliers[player.UserId] = 1
end

local function updateLeaderboard(player)
end


local function claimDaily(player)
	local last = dailyClaimed[player.UserId] or 0
	if os.time() - last >= 86400 then
		addMoney(player, 100)
		dailyClaimed[player.UserId] = os.time()
	end
end


Players.PlayerAdded:Connect(function(player)
	local leaderstats = Instance.new("Folder")
	leaderstats.Name = "leaderstats"
	leaderstats.Parent = player

	local Money = Instance.new("IntValue")
	Money.Name = "Money"
	Money.Parent = leaderstats
	Money.Value = loadPlayerData(player)

	multipliers[player.UserId] = 1
	cooldowns[player.UserId] = 0
	dailyClaimed[player.UserId] = 0

	player:WaitForChild("leaderstats")
	local stats = player.leaderstats

	for _, stat in ipairs(stats:GetChildren()) do
		if stat:IsA("NumberValue") then
			stat.Changed:Connect(function(newVal)
				local mult = multipliers[player.UserId] or 1
				if mult > 1 then
					local expected = math.floor(newVal / mult)
					if expected * mult ~= newVal then
						stat.Value = expected * mult
					end
				end
			end)
		end
	end

	stats.ChildAdded:Connect(function(child)
		if child:IsA("NumberValue") then
			child.Changed:Connect(function(newVal)
				local mult = multipliers[player.UserId] or 1
				if mult > 1 then
					local expected = math.floor(newVal / mult)
					if expected * mult ~= newVal then
						child.Value = expected * mult
					end
				end
			end)
		end
	end)

	local inventory = loadInventory(player)
	for _, toolName in ipairs(inventory) do
		givePlayerTool(player, toolName)
	end
end)


Players.PlayerRemoving:Connect(function(player)
	savePlayerData(player, player.leaderstats.Money.Value)
	saveInventory(player, loadInventory(player))
	multipliers[player.UserId] = nil
	cooldowns[player.UserId] = nil
	dailyClaimed[player.UserId] = nil
end)

-- Event handler
event.OnServerEvent:Connect(function(player, amount)
	if not canUse(player) then return end
	setCooldown(player)

	amount = tonumber(amount) or 50
	if amount > MAX_MONEY_PER_EVENT then return end

	addMoney(player, amount)
	givePlayerTool(player, TOOL_NAME)
end)


function giveDoubleMoney(player)
	multipliers[player.UserId] = 2
end

-- Auto-save
task.spawn(function()
	while true do
		task.wait(AUTO_SAVE_INTERVAL)
		for _, player in ipairs(Players:GetPlayers()) do
			if player:FindFirstChild("leaderstats") then
				savePlayerData(player, player.leaderstats.Money.Value)
				saveInventory(player, loadInventory(player))
			end
		end
	end
end)

-- Reset multipliers periodically
task.spawn(function()
	while true do
		task.wait(1)
		for userId, mult in pairs(multipliers) do
			if mult > 1 then
				multipliers[userId] = 1
			end
		end
	end
end)

RunService.Heartbeat:Connect(function()
	for _, player in ipairs(Players:GetPlayers()) do
		if not player:FindFirstChild("leaderstats") then
			local leaderstats = Instance.new("Folder")
			leaderstats.Name = "leaderstats"
			leaderstats.Parent = player
		end
	end
end)
