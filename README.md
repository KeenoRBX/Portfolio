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
