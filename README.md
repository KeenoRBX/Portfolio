# Portfolio
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local TweenService = game:GetService("TweenService")
local Debris = game:GetService("Debris")

local fireskill = ReplicatedStorage:WaitForChild("Fire Skill")
local purple = ReplicatedStorage:WaitForChild("Purple")
local hollowpurple = ReplicatedStorage:WaitForChild("hollow purple")
local tweeninfo = TweenInfo.new(3, Enum.EasingStyle.Quart, Enum.EasingDirection.In)
local tweeninfo2 = TweenInfo.new(1, Enum.EasingStyle.Linear)
local ShakeCamera = ReplicatedStorage:WaitForChild("ShakeCamera")

server-
fireskill.OnServerEvent:Connect(function(plr)
	local char = plr.Character
	if not char then return end

	local hrp : Part = char:FindFirstChild("HumanoidRootPart")
	local hum = char:FindFirstChild("Humanoid")

	


	local blue = ReplicatedStorage.part:FindFirstChild("Blue"):Clone()
	local red = ReplicatedStorage.part:FindFirstChild("red"):Clone()

	blue.Parent = workspace
	red.Parent = workspace

	
	blue.CFrame = hrp.CFrame * CFrame.new(15, 2, -8)
	red.CFrame = hrp.CFrame * CFrame.new(-15, 2, -8)


	local centerCFrame = hrp.CFrame * CFrame.new(0, 2, -10)

	local tweenBlue = TweenService:Create(blue, tweeninfo, {CFrame = centerCFrame})
	local tweenRed = TweenService:Create(red, tweeninfo, {CFrame = centerCFrame})
	ShakeCamera:FireClient(plr,0.25, 2)
	tweenBlue:Play()
	tweenRed:Play()

	
	task.wait(3) 

	
	blue:Destroy()
	red:Destroy()

	local purpclone = purple:Clone()
	purpclone.Parent = workspace
	purpclone.CFrame = centerCFrame * CFrame.new(0,0,-3)
	
	ShakeCamera:FireClient(plr,1,1 )
	local attachment = purpclone:FindFirstChild("Attachment")
	
	if attachment then
		hollowpurple:FireClient(plr)
		for _, v in ipairs(attachment:GetChildren()) do
			if v:IsA("ParticleEmitter") then
				v.Enabled = true
				
				task.spawn(function()
			
					for i = 1,20 do
						v.Size = NumberSequence.new(i  * 1.5)
						task.wait(0.05)
					end
				end)
			end
		end
	end
	task.wait(0.5)
	
	local att = Instance.new("Attachment")
	att.Parent = purpclone  

	local linear = Instance.new("LinearVelocity")
	linear.MaxForce = math.huge
	linear.Attachment0 = att
	linear.VectorVelocity = purpclone.CFrame.LookVector * 250
	linear.RelativeTo = Enum.ActuatorRelativeTo.World
	linear.Parent = purpclone
	


	
	
end)
