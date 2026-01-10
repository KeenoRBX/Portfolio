-- Main Module
local Gift = {}
Gift.__index = Gift


function Gift.new(giftName, config, giftsFolder, rewardsFrames, remotes)
	local self = setmetatable({}, Gift)

	self.Name = giftName
	self.Config = config
	self.GiftsFolder = giftsFolder
	self.Remotes = remotes

	self.GiftValue = giftsFolder:FindFirstChild(giftName)
	self.Frame = rewardsFrames:FindFirstChild(giftName)
	self.Button = self.Frame and self.Frame:FindFirstChildWhichIsA("ImageButton")
	self.TimeLabel = self.Button and self.Button:FindFirstChild("Time")

	self.CooldownThread = nil
	self.CollectedLocal = false

	return self
end


function Gift:FormatTime(seconds)
	local h = math.floor(seconds / 3600)
	local m = math.floor((seconds % 3600) / 60)
	local s = seconds % 60
	return string.format("%02d:%02d:%02d", h, m, s)
end


function Gift:StopCooldown()
	if self.CooldownThread then
		task.cancel(self.CooldownThread)
		self.CooldownThread = nil
	end
end

function Gift:StartCooldown()
	if self.CollectedLocal then return end
	if not self.GiftValue or self.GiftValue.Value then return end

	self:StopCooldown()

	self.CooldownThread = task.spawn(function()
		for t = self.Config.tempo, 0, -1 do
			if self.GiftValue.Value then
				self.TimeLabel.Text = "Ready!"
				self:StopCooldown()
				return
			end
			self.TimeLabel.Text = self:FormatTime(t)
			task.wait(1)
		end

		self.TimeLabel.Text = "Ready!"
		self.Remotes.getboolchanged:FireServer(self.Name, true)
		self:StopCooldown()
	end)
end


function Gift:UpdateState()
	if not self.GiftValue then return end

	if self.CollectedLocal then
		self.TimeLabel.Text = "Collected!"
		return
	end

	if self.GiftValue.Value then
		self:StopCooldown()
		self.TimeLabel.Text = "Ready!"
	else
		self:StartCooldown()
	end
end


function Gift:Collect()
	if self.CollectedLocal then return end
	if not self.GiftValue.Value then return end

	self.CollectedLocal = true
	self.TimeLabel.Text = "Collected!"
	self:StopCooldown()

	
	if self.Config.tipo == "Potion" then
		local player = self.GiftsFolder.Parent
		local potionFolder = player:WaitForChild("PotionFolder")
		local potion = potionFolder:FindFirstChild(self.Config.PotionName)

		if not potion then
			
			return
		end

		self.Remotes.UsePotion:FireServer(potion)
	else
		
		if self.Config.Mult then
			self.Remotes.getclick:FireServer(
				self.Config.amount,
				self.Config.tipo,
				self.Config.Mult
			)
		else
			self.Remotes.getclick:FireServer(
				self.Config.amount,
				self.Config.tipo
			)
		end
	end

	self.Remotes.getboolchanged:FireServer(self.Name, false)
end


function Gift:Setup()
	if not self.GiftValue or not self.Button then return end

	self:UpdateState()

	self.GiftValue.Changed:Connect(function()
		self:UpdateState()
	end)

	self.Button.MouseButton1Click:Connect(function()
		self:Collect()
	end)
end

return Gift

	---------++++++-------------++++



















	-------------------------------------------------------------------------------------------------
	Server Code
	-- =========================
-- SERVICES
-- =========================
local Players = game:GetService("Players")
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local MarketplaceService = game:GetService("MarketplaceService")

-- =========================
-- MODULES
-- =========================
local ValueGifts = require(
	ReplicatedStorage.GiftSystem.IncreaseValue.ValueGifts
)

local chooseFunction = require(
	ReplicatedStorage.GiftSystem.IncreaseValue.ValueGifts.chooseFunction
)

local PotionSystem = require(
	ReplicatedStorage.modules.PotionSystem.PotionSystem
)

-- =========================
-- REMOTES
-- =========================
local Remotes = ReplicatedStorage:WaitForChild("Remotes")
local UsePotion = Remotes:WaitForChild("UsePotion")
local GetClick = Remotes:WaitForChild("getclick")
local GetBoolChanged = Remotes:WaitForChild("getboolchanged")
local PurchaseAll = Remotes:WaitForChild("purchaseallgifts")

-- =========================
-- USE POTION
-- =========================
UsePotion.OnServerEvent:Connect(function(player, potionValue)
	if not potionValue
		or not potionValue:IsDescendantOf(player:WaitForChild("PotionFolder")) then
		warn("you cant use potion")
		return
	end

	if potionValue.Value <= 0 then return end

	potionValue.Value -= 1
	PotionSystem.PotionHandler(player, potionValue.Name)
end)

-- =========================
-- BOOL GIFT STATE
-- =========================
GetBoolChanged.OnServerEvent:Connect(function(player, giftName, value)
	local folder = player:FindFirstChild("GiftsFolder")
	local gift = folder and folder:FindFirstChild(giftName)
	if gift then
		gift.Value = value
	end
end)

-- =========================
-- CLICK / REWARD
-- =========================
GetClick.OnServerEvent:Connect(function(player, amount, kind, multName)
	local multValue = 1

	if multName then
		local multFolder = player:FindFirstChild("Multipliers")
		local mult = multFolder and multFolder:FindFirstChild(multName)
		if mult then
			multValue = mult.Value
		end
	end

	-- sempre passa o VALOR NUMÉRICO
	chooseFunction(player, amount, kind, multValue)
end)

-- =========================
-- DEV PRODUCT (PURCHASE ALL)
-- =========================
MarketplaceService.PromptProductPurchaseFinished:Connect(function(userId, productId, wasPurchased)
	if not wasPurchased then return end

	local player = Players:GetPlayerByUserId(userId)
	if not player then return end

	if productId ~= 3478654682 then return end

	local giftsFolder = player:FindFirstChild("GiftsFolder")
	if not giftsFolder then return end

	for _, gift in ipairs(giftsFolder:GetChildren()) do
		if gift:IsA("BoolValue") then
			gift.Value = true
			PurchaseAll:FireClient(player, gift.Name)
		end
	end
end)

-- =========================
-- PLAYER ADDED
-- =========================
Players.PlayerAdded:Connect(function(player)

	-- Gifts
	local giftsFolder = Instance.new("Folder")
	giftsFolder.Name = "GiftsFolder"
	giftsFolder.Parent = player

	for giftName in pairs(ValueGifts) do
		local b = Instance.new("BoolValue")
		b.Name = giftName
		b.Value = false
		b.Parent = giftsFolder
	end

	-- Multipliers
	local multFolder = Instance.new("Folder")
	multFolder.Name = "Multipliers"
	multFolder.Parent = player

	local cashMult = Instance.new("NumberValue")
	cashMult.Name = "CashMult"
	cashMult.Value = 1
	cashMult.Parent = multFolder

	local winsMult = Instance.new("NumberValue")
	winsMult.Name = "WinsMult"
	winsMult.Value = 1
	winsMult.Parent = multFolder

	-- Potions
	local potionFolder = Instance.new("Folder")
	potionFolder.Name = "PotionFolder"
	potionFolder.Parent = player

	local cashPotion = Instance.new("IntValue")
	cashPotion.Name = "2x Cash"
	cashPotion.Value = 1
	cashPotion.Parent = potionFolder

	-- Leaderstats (exemplo)
	local leaderstats = Instance.new("Folder")
	leaderstats.Name = "leaderstats"
	leaderstats.Parent = player

	local cash = Instance.new("IntValue")
	cash.Name = "Cash"
	cash.Value = 0
	cash.Parent = leaderstats

	local wins = Instance.new("IntValue")
	wins.Name = "Wins"
	wins.Value = 0
	wins.Parent = leaderstats
end)







