-- SERVICES
local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local Camera = workspace.CurrentCamera
local ReplicatedStorage = game:GetService("ReplicatedStorage")

local player = Players.LocalPlayer

-- GUI
local gui = Instance.new("ScreenGui")
gui.Name = "MiniHubV3"
gui.ResetOnSpawn = false
gui.Parent = player:WaitForChild("PlayerGui")

-- PANEL
local panel = Instance.new("Frame", gui)
panel.Size = UDim2.new(0,320,0,260)
panel.Position = UDim2.new(0.35,0,0.3,0)
panel.BackgroundColor3 = Color3.fromRGB(15,15,20)
panel.BorderSizePixel = 0
panel.Active = true
panel.Draggable = true
Instance.new("UICorner", panel).CornerRadius = UDim.new(0,12)

local title = Instance.new("TextLabel", panel)
title.Size = UDim2.new(1,0,0,30)
title.Text = "MiniHub"
title.TextColor3 = Color3.new(1,1,1)
title.BackgroundTransparency = 1
title.Font = Enum.Font.GothamBold
title.TextSize = 18

-- BUTTON FACTORY
local function btn(text,y)
	local b = Instance.new("TextButton",panel)
	b.Size = UDim2.new(0,150,0,28)
	b.Position = UDim2.new(0,15,0,y)
	b.Text = text
	b.Font = Enum.Font.Gotham
	b.TextSize = 12
	b.TextColor3 = Color3.new(1,1,1)
	b.BackgroundColor3 = Color3.fromRGB(30,30,40)
	b.BorderSizePixel = 0
	Instance.new("UICorner",b).CornerRadius = UDim.new(0,8)
	return b
end

-- STATES
local AimAssist = false
local ESPLine = false
local TPPlayer = false
local WeaponPicker = false
local Cosplay = false

-- STORAGE
local Lines = {}

-- BUTTONS
local aimBtn  = btn("AIM ASSIST: OFF",40)
local lineBtn = btn("ESP LINE: OFF",75)
local gunBtn  = btn("PEGAR ARMA: OFF",110)
local tpBtn   = btn("TP PLAYER: OFF",145)
local cosplayBtn = btn("COSPLAY: OFF",180)

--------------------------------------------------
-- AIM ASSIST
--------------------------------------------------
local FOV = 150
local STRENGTH = 0.35

local function getClosest()
	local center = Vector2.new(Camera.ViewportSize.X/2,Camera.ViewportSize.Y/2)
	local closest,dist = nil,math.huge

	for _,plr in pairs(Players:GetPlayers()) do
		if plr ~= player and plr.Character and plr.Character:FindFirstChild("Head") then
			local pos,vis = Camera:WorldToViewportPoint(plr.Character.Head.Position)
			if vis then
				local d = (Vector2.new(pos.X,pos.Y)-center).Magnitude
				if d < dist and d < FOV then
					dist = d
					closest = plr.Character.Head
				end
			end
		end
	end
	return closest
end

RunService.RenderStepped:Connect(function()
	if AimAssist then
		local target = getClosest()
		if target then
			local camPos = Camera.CFrame.Position
			local dir = (target.Position-camPos).Unit
			Camera.CFrame = Camera.CFrame:Lerp(
				CFrame.new(camPos,camPos+dir),
				STRENGTH
			)
		end
	end
end)

--------------------------------------------------
-- ESP LINE
--------------------------------------------------
local hue = 0
local function rgb()
	hue += 0.004
	if hue > 1 then hue = 0 end
	return Color3.fromHSV(hue,1,1)
end

local function clearESP()
	for _,v in pairs(Lines) do
		if v.beam then v.beam:Destroy() end
		if v.a0 then v.a0:Destroy() end
		if v.a1 then v.a1:Destroy() end
	end
	Lines = {}
end

RunService.RenderStepped:Connect(function()
	if not ESPLine then
		clearESP()
		return
	end

	local myChar = player.Character
	local myRoot = myChar and myChar:FindFirstChild("HumanoidRootPart")
	if not myRoot then return end

	local color = rgb()
	clearESP()

	for _,plr in pairs(Players:GetPlayers()) do
		if plr ~= player and plr.Character then
			local root = plr.Character:FindFirstChild("HumanoidRootPart")
			if root then
				local a0 = Instance.new("Attachment", myRoot)
				local a1 = Instance.new("Attachment", root)

				local beam = Instance.new("Beam", workspace)
				beam.Attachment0 = a0
				beam.Attachment1 = a1
				beam.Width0 = 0.02
				beam.Width1 = 0.02
				beam.FaceCamera = true
				beam.Color = ColorSequence.new(color)

				table.insert(Lines,{beam=beam,a0=a0,a1=a1})
			end
		end
	end
end)

--------------------------------------------------
-- PEGAR ARMAS DO MAPA
--------------------------------------------------
local Character = player.Character or player.CharacterAdded:Wait()
local HumanoidRootPart = Character:WaitForChild("HumanoidRootPart")

local POSSIVEIS_ARMAS = {}
local PEGANDO_ARMA = false

local function isWeapon(obj)
	if obj:IsA("Model") then
		for _,v in ipairs(obj:GetDescendants()) do
			if v:IsA("TouchTransmitter") then
				return true
			end
		end
	end
	return false
end

local function scanWeapons()
	table.clear(POSSIVEIS_ARMAS)
	for _,obj in ipairs(workspace:GetDescendants()) do
		if isWeapon(obj) then
			table.insert(POSSIVEIS_ARMAS,obj)
		end
	end
end

local function pegarArma(arma)
	if not arma or PEGANDO_ARMA then return end
	local part = arma:FindFirstChildWhichIsA("BasePart")
	if not part then return end

	PEGANDO_ARMA = true
	HumanoidRootPart.CFrame = part.CFrame + Vector3.new(0,2,0)
	task.wait(0.25)

	firetouchinterest(HumanoidRootPart, part, 0)
	task.wait()
	firetouchinterest(HumanoidRootPart, part, 1)

	PEGANDO_ARMA = false
end

--------------------------------------------------
-- INTERFACE PEGAR ARMA
--------------------------------------------------
local gunFrame = Instance.new("Frame", gui)
gunFrame.Size = UDim2.new(0,200,0,220)
gunFrame.Position = UDim2.new(0.7,0,0.3,0)
gunFrame.BackgroundColor3 = Color3.fromRGB(20,20,30)
gunFrame.Visible = false
gunFrame.BorderSizePixel = 0
Instance.new("UICorner",gunFrame).CornerRadius = UDim.new(0,8)

local gunLayout = Instance.new("UIListLayout", gunFrame)
gunLayout.Padding = UDim.new(0,5)

local function refreshWeapons()
	for _,v in pairs(gunFrame:GetChildren()) do
		if v:IsA("TextButton") then v:Destroy() end
	end

	scanWeapons()

	for _,arma in ipairs(POSSIVEIS_ARMAS) do
		local b = Instance.new("TextButton", gunFrame)
		b.Size = UDim2.new(1,-8,0,26)
		b.Text = arma.Name
		b.Font = Enum.Font.Gotham
		b.TextSize = 11
		b.TextColor3 = Color3.new(1,1,1)
		b.BackgroundColor3 = Color3.fromRGB(35,35,45)
		b.BorderSizePixel = 0
		Instance.new("UICorner",b).CornerRadius = UDim.new(0,6)

		b.MouseButton1Click:Connect(function()
			pegarArma(arma)
		end)
	end
end

--------------------------------------------------
-- TP PLAYER
--------------------------------------------------
local tpFrame = Instance.new("Frame", panel)
tpFrame.Size = UDim2.new(0,120,0,90)
tpFrame.Position = UDim2.new(0,185,0,145)
tpFrame.BackgroundColor3 = Color3.fromRGB(20,20,30)
tpFrame.Visible = false
tpFrame.BorderSizePixel = 0
Instance.new("UICorner",tpFrame).CornerRadius = UDim.new(0,8)

local layout2 = Instance.new("UIListLayout", tpFrame)
layout2.Padding = UDim.new(0,4)

local function refreshTPList()
	for _,v in pairs(tpFrame:GetChildren()) do
		if v:IsA("TextButton") then v:Destroy() end
	end

	for _,plr in pairs(Players:GetPlayers()) do
		if plr ~= player then
			local b = Instance.new("TextButton", tpFrame)
			b.Size = UDim2.new(1,-6,0,22)
			b.Text = plr.Name
			b.Font = Enum.Font.Gotham
			b.TextSize = 11
			b.TextColor3 = Color3.new(1,1,1)
			b.BackgroundColor3 = Color3.fromRGB(35,35,45)
			b.BorderSizePixel = 0
			Instance.new("UICorner",b).CornerRadius = UDim.new(0,6)

			b.MouseButton1Click:Connect(function()
				player.Character:PivotTo(plr.Character.HumanoidRootPart.CFrame * CFrame.new(0,0,-4))
			end)
		end
	end
end

--------------------------------------------------
-- COSPLAY / COPIAR SKIN
--------------------------------------------------
local cosplayFrame = Instance.new("Frame", gui)
cosplayFrame.Size = UDim2.new(0,200,0,220)
cosplayFrame.Position = UDim2.new(0.7,0,0.3,0)
cosplayFrame.BackgroundColor3 = Color3.fromRGB(20,20,30)
cosplayFrame.Visible = false
cosplayFrame.BorderSizePixel = 0
Instance.new("UICorner",cosplayFrame).CornerRadius = UDim.new(0,8)

local cosplayLayout = Instance.new("UIListLayout", cosplayFrame)
cosplayLayout.Padding = UDim.new(0,5)

local function copiarSkin(targetPlayer)
	if not targetPlayer then return end
	if not player.Character then return end

	local humanoid = player.Character:FindFirstChildOfClass("Humanoid")
	if not humanoid then return end

	local success, description = pcall(function()
		return Players:GetHumanoidDescriptionFromUserId(targetPlayer.UserId)
	end)

	if success and description then
		humanoid:ApplyDescription(description)
	end
end

local function refreshCosplay()
	for _,v in pairs(cosplayFrame:GetChildren()) do
		if v:IsA("TextButton") then v:Destroy() end
	end

	for _,plr in pairs(Players:GetPlayers()) do
		if plr ~= player then
			local b = Instance.new("TextButton", cosplayFrame)
			b.Size = UDim2.new(1,-8,0,26)
			b.Text = plr.Name
			b.Font = Enum.Font.Gotham
			b.TextSize = 11
			b.TextColor3 = Color3.new(1,1,1)
			b.BackgroundColor3 = Color3.fromRGB(35,35,45)
			b.BorderSizePixel = 0
			Instance.new("UICorner",b).CornerRadius = UDim.new(0,6)

			b.MouseButton1Click:Connect(function()
				copiarSkin(plr)
			end)
		end
	end
end

--------------------------------------------------
-- TOGGLES
--------------------------------------------------
aimBtn.MouseButton1Click:Connect(function()
	AimAssist = not AimAssist
	aimBtn.Text = AimAssist and "AIM ASSIST: ON" or "AIM ASSIST: OFF"
end)

lineBtn.MouseButton1Click:Connect(function()
	ESPLine = not ESPLine
	lineBtn.Text = ESPLine and "ESP LINE: ON" or "ESP LINE: OFF"
	clearESP()
end)

gunBtn.MouseButton1Click:Connect(function()
	WeaponPicker = not WeaponPicker
	gunBtn.Text = WeaponPicker and "PEGAR ARMA: ON" or "PEGAR ARMA: OFF"
	gunFrame.Visible = WeaponPicker
	if WeaponPicker then refreshWeapons() end
end)

tpBtn.MouseButton1Click:Connect(function()
	TPPlayer = not TPPlayer
	tpBtn.Text = TPPlayer and "TP PLAYER: ON" or "TP PLAYER: OFF"
	tpFrame.Visible = TPPlayer
	if TPPlayer then refreshTPList() end
end)

cosplayBtn.MouseButton1Click:Connect(function()
	Cosplay = not Cosplay
	cosplayBtn.Text = Cosplay and "COSPLAY: ON" or "COSPLAY: OFF"
	cosplayFrame.Visible = Cosplay
	if Cosplay then refreshCosplay() end
end)

Players.PlayerAdded:Connect(refreshTPList)
Players.PlayerRemoving:Connect(refreshTPList)
Players.PlayerAdded:Connect(refreshCosplay)
Players.PlayerRemoving:Connect(refreshCosplay)

print("MiniHub V3 — Aim, ESP, TP, Weapon Picker e Cosplay integrados e funcionais.")
