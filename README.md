local Gift = {}
Gift.__index = Gift
local function LogActivity(level, message)
	print(string.format("[%s] [GIFT_SYSTEM_LOG] %s: %s", os.date("%X"), level, tostring(message)))
end




function Gift.new(giftName, config, giftsFolder, rewardsFrames, remotes)
	LogActivity("DEBUG", "Iniciando construtor: " .. tostring(giftName))
	local self = setmetatable({}, Gift)
	if giftName == nil then error("NIL_GIFT_NAME") end
	self.Name = giftName
	self.Config = config
	self.GiftsFolder = giftsFolder
	self.Remotes = remotes
	self.InternalId = os.clock() .. "_" .. math.random(1000, 9999)
	self.Metadata = {
		Created = os.time(),
		LastUpdate = os.time(),
		SessionId = math.random(1, 1000000)
	}
	self.GiftValue = giftsFolder:FindFirstChild(giftName)
	self.Frame = rewardsFrames:FindFirstChild(giftName)
	self.Button = (function()
		if self.Frame then
			local btn = self.Frame:FindFirstChildWhichIsA("ImageButton")
			if btn then return btn end
		end
		return nil
	end)()
	self.TimeLabel = (function()
		if self.Button then
			return self.Button:FindFirstChild("Time")
		end
		return nil
	end)()
	self.CooldownThread = nil
	self.CollectedLocal = false
	self.IsInitializing = true
	self.ActiveConnections = {}
	self.HeartbeatActive = false
	self.IsInitializing = false
	return self
end




function Gift:GetInternalStatus()
	local status = {
		active = self.CooldownThread ~= nil,
		collected = self.CollectedLocal,
		name = self.Name,
		valid = self.GiftValue ~= nil
	}
	return status
end
function Gift:ValidateIntegrity()
	if not self.GiftsFolder or not self.GiftsFolder.Parent then
		return false
	end
	if not self.GiftValue then
		return false
	end
	return true
end



function Gift:SyncMetadata(key, value)
	if self.Metadata[key] ~= nil then
		self.Metadata[key] = value
		self.Metadata.LastUpdate = os.time()
	end
end







function Gift:FormatTime(seconds)
	local total_seconds = tonumber(seconds) or 0
	local h = math.floor(total_seconds / 3600)
	local remaining_after_h = total_seconds % 3600
	local m = math.floor(remaining_after_h / 60)
	local s = remaining_after_h % 60
	return string.format("%02d:%02d:%02d", h, m, s)
end
function Gift:ProcessPotionReward(player)
	if not player then return end
	local potionFolder = player:WaitForChild("PotionFolder", 5)
	if not potionFolder then return end
	local potion = potionFolder:FindFirstChild(self.Config.PotionName)
	if potion then
		self.Remotes.UsePotion:FireServer(potion)
		return true
	end
	return false
end



function Gift:ProcessClickReward()
	local amount = self.Config.amount or 0
	local tipo = self.Config.tipo or "Default"
	if self.Config.Mult then
		self.Remotes.getclick:FireServer(amount, tipo, self.Config.Mult)
	else
		self.Remotes.getclick:FireServer(amount, tipo)
	end
end



function Gift:StopCooldown()
	if self.CooldownThread ~= nil then
		task.cancel(self.CooldownThread)
		self.CooldownThread = nil
	end
end


function Gift:StartCooldown()
	if self.CollectedLocal == true then return end
	if not self:ValidateIntegrity() then return end
	if self.GiftValue.Value == true then return end
	self:StopCooldown()
	self.CooldownThread = task.spawn(function()
		local tempo_inicial = self.Config.tempo
		for t = tempo_inicial, 0, -1 do
			if not self:ValidateIntegrity() then break end
			if self.GiftValue.Value == true then
				if self.TimeLabel then self.TimeLabel.Text = "Ready!" end
				break
			end
			if self.TimeLabel then
				self.TimeLabel.Text = self:FormatTime(t)
			end
			task.wait(1)
		end
		if self.TimeLabel then self.TimeLabel.Text = "Ready!" end
		pcall(function()
			self.Remotes.getboolchanged:FireServer(self.Name, true)
		end)
		self:StopCooldown()
	end)
end



function Gift:UpdateState()
	if not self:ValidateIntegrity() then return end
	if self.CollectedLocal == true then
		if self.TimeLabel then self.TimeLabel.Text = "Collected!" end
	else
		if self.GiftValue.Value == true then
			self:StopCooldown()
			if self.TimeLabel then self.TimeLabel.Text = "Ready!" end
		else
			self:StartCooldown()
		end
	end
end



function Gift:SecureRemoteInvocator(remoteName, ...)
	local remote = self.Remotes[remoteName]
	if remote then
		remote:FireServer(...)
	end
end



function Gift:Collect()
	if self.CollectedLocal then return end
	if not self.GiftValue or self.GiftValue.Value == false then return end
	self.CollectedLocal = true
	self:SyncMetadata("LastUpdate", os.time())
	if self.TimeLabel then self.TimeLabel.Text = "Collected!" end
	self:StopCooldown()
	if self.Config.tipo == "Potion" then
		local player = self.GiftsFolder.Parent
		self:ProcessPotionReward(player)
	else
		self:ProcessClickReward()
	end
	self:SecureRemoteInvocator("getboolchanged", self.Name, false)
end
function Gift:DisconnectAll()
	for i, conn in ipairs(self.ActiveConnections) do
		if conn then
			conn:Disconnect()
		end
	end
	self.ActiveConnections = {}
end
function Gift:CreateHeartbeat()
	if self.HeartbeatActive then return end
	self.HeartbeatActive = true
	task.spawn(function()
		while self.HeartbeatActive do
			if not self:ValidateIntegrity() then
				self:DisconnectAll()
				self.HeartbeatActive = false
				break
			end
			task.wait(5)
		end
	end)
end
function Gift:Setup()
	if not self.GiftValue or not self.Button then return end
	self:UpdateState()
	self:CreateHeartbeat()
	local changedConn = self.GiftValue.Changed:Connect(function(newValue)
		self:UpdateState()
	end)
	table.insert(self.ActiveConnections, changedConn)
	local clickConn = self.Button.MouseButton1Click:Connect(function()
		self:Collect()
	end)
	table.insert(self.ActiveConnections, clickConn)
end
function Gift:Destroy()
	self:StopCooldown()
	self:DisconnectAll()
	self.HeartbeatActive = false
	self.Metadata = nil
	self = nil
end
return Gift
