# steal-a-latamrot
steal a latamrot

-- ULTRA PRIVADO: Panel táctil (Walk / Flight 3D / Jump) + Noclip (no suelo)
-- Rango: 16 -> 1000
-- Pegar como LocalScript en StarterPlayer > StarterPlayerScripts

local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local UserInputService = game:GetService("UserInputService")
local TweenService = game:GetService("TweenService")
local workspace = game:GetService("Workspace")

local player = Players.LocalPlayer
local guiParent = player:WaitForChild("PlayerGui")

-- CONFIG
local MIN_VALUE = 16
local MAX_VALUE = 1000

local DEFAULT_WALK = 16
local DEFAULT_FLIGHT = 50
local DEFAULT_JUMP = 50

-- STATE
local character = player.Character or player.CharacterAdded:Wait()
local humanoid = character:WaitForChild("Humanoid")
local hrp = character:WaitForChild("HumanoidRootPart")

local walkSpeed = DEFAULT_WALK
local flightSpeed = DEFAULT_FLIGHT
local jumpPower = DEFAULT_JUMP

local flying = false
local ascendDir = 0 -- -1 down, 0 none, 1 up
local noclip = false

local bv, bg -- BodyVelocity & BodyGyro for flight

-- RAYCAST PARAMS to detect ground (ignore character)
local function groundHitFrom(position, distance)
	local params = RaycastParams.new()
	params.FilterDescendantsInstances = {character}
	params.FilterType = Enum.RaycastFilterType.Blacklist
	params.IgnoreWater = true
	local result = workspace:Raycast(position, Vector3.new(0, -distance, 0), params)
	return result
end

-- Refresh references on respawn
local function refreshRefs(char)
	character = char
	humanoid = char:WaitForChild("Humanoid")
	hrp = char:WaitForChild("HumanoidRootPart")
	humanoid.WalkSpeed = walkSpeed
	humanoid.UseJumpPower = true
	humanoid.JumpPower = jumpPower
end

player.CharacterAdded:Connect(function(char)
	task.wait(0.2)
	refreshRefs(char)
	-- if flying was on, re-enable movers
	if flying then
		task.wait(0.2)
		-- ensure no duplicates
		if bv then pcall(function() bv:Destroy() end) end
		if bg then pcall(function() bg:Destroy() end) end
		-- enable flight again
		-- enableFlight called later if needed
	end
end)

-- GUI CREATION
local screenGui = Instance.new("ScreenGui")
screenGui.Name = "UltraPrivatePanelV3"
screenGui.ResetOnSpawn = false
screenGui.Parent = guiParent
screenGui.DisplayOrder = 9999

local panel = Instance.new("Frame", screenGui)
panel.Size = UDim2.new(0, 400, 0, 300)
panel.Position = UDim2.new(0.65, 0, 0.65, 0)
panel.AnchorPoint = Vector2.new(0.5, 0.5)
panel.BackgroundColor3 = Color3.fromRGB(12,16,28)
panel.BackgroundTransparency = 0.04
panel.BorderSizePixel = 0
local panelCorner = Instance.new("UICorner", panel)
panelCorner.CornerRadius = UDim.new(0, 16)

local gradient = Instance.new("UIGradient", panel)
gradient.Color = ColorSequence.new{
	ColorSequenceKeypoint.new(0, Color3.fromRGB(0,170,255)),
	ColorSequenceKeypoint.new(0.5, Color3.fromRGB(130,40,255)),
	ColorSequenceKeypoint.new(1, Color3.fromRGB(0,220,140))
}
gradient.Rotation = 0
task.spawn(function()
	while screenGui.Parent do
		gradient.Rotation = (gradient.Rotation + 0.6) % 360
		task.wait(0.02)
	end
end)

local title = Instance.new("TextLabel", panel)
title.Size = UDim2.new(1,0,0,44)
title.Position = UDim2.new(0,0,0,0)
title.BackgroundTransparency = 1
title.Text = "🔧 Panel Privado: Caminata · Vuelo 3D · Salto · Noclip"
title.Font = Enum.Font.GothamBold
title.TextScaled = true
title.TextColor3 = Color3.fromRGB(245,245,245)

local closeBtn = Instance.new("TextButton", panel)
closeBtn.Size = UDim2.new(0,36,0,32)
closeBtn.Position = UDim2.new(0.94,0,0.05,0)
closeBtn.Text = "✖"
closeBtn.Font = Enum.Font.GothamBold
closeBtn.TextScaled = true
closeBtn.BackgroundColor3 = Color3.fromRGB(230,80,80)
local closeCorner = Instance.new("UICorner", closeBtn)
closeCorner.CornerRadius = UDim.new(1,0)
closeBtn.MouseButton1Click:Connect(function() panel.Visible = false end)

-- helper to create labels & sliders
local function createLabel(parent, text, y)
	local l = Instance.new("TextLabel", parent)
	l.Size = UDim2.new(1, -20, 0, 24)
	l.Position = UDim2.new(0, 10, 0, y)
	l.BackgroundTransparency = 1
	l.Font = Enum.Font.GothamSemibold
	l.TextScaled = true
	l.TextColor3 = Color3.fromRGB(240,240,240)
	l.Text = text
	return l
end

local function createSlider(parent, y)
	local bg = Instance.new("Frame", parent)
	bg.Size = UDim2.new(0.94, 0, 0, 18)
	bg.Position = UDim2.new(0.03, 0, 0, y)
	bg.BackgroundColor3 = Color3.fromRGB(40,40,60)
	bg.BorderSizePixel = 0
	local bgCorner = Instance.new("UICorner", bg)
	bgCorner.CornerRadius = UDim.new(1, 0)
	local bar = Instance.new("Frame", bg)
	bar.Size = UDim2.new(0,0,1,0)
	bar.BackgroundColor3 = Color3.fromRGB(0,170,255)
	local barCorner = Instance.new("UICorner", bar)
	barCorner.CornerRadius = UDim.new(1,0)
	return bg, bar
end

-- Walk slider
local walkLabel = createLabel(panel, "Velocidad Caminata: "..walkSpeed, 56)
local walkBg, walkBar = createSlider(panel, 92)

-- Flight slider
local flightLabel = createLabel(panel, "Velocidad Vuelo: "..flightSpeed, 122)
local flightBg, flightBar = createSlider(panel, 158)

-- Jump slider
local jumpLabel = createLabel(panel, "Fuerza Salto: "..jumpPower, 188)
local jumpBg, jumpBar = createSlider(panel, 224)

-- Buttons: Fly toggle, Up, Down, Noclip toggle, Reset
local function mkButton(text, posX)
	local b = Instance.new("TextButton", panel)
	b.Size = UDim2.new(0, 86, 0, 36)
	b.Position = UDim2.new(posX, 0, 0.78, 0)
	b.Text = text
	b.Font = Enum.Font.GothamBold
	b.TextScaled = true
	b.BackgroundColor3 = Color3.fromRGB(65,65,85)
	b.TextColor3 = Color3.fromRGB(245,245,245)
	local c = Instance.new("UICorner", b)
	c.CornerRadius = UDim.new(0,8)
	return b
end

local flyBtn = mkButton("✈️ Vuelo: OFF", 0.04)
local upBtn = mkButton("⬆️ Subir", 0.28)
local downBtn = mkButton("⬇️ Bajar", 0.52)
local noclipBtn = mkButton("🚧 Noclip: OFF", 0.76)
local resetBtn = mkButton("↺ Reset", 0.88)

-- Panel drag (touch & mouse)
local draggingPanel = false
local dragStartPos, panelStartPos
panel.InputBegan:Connect(function(input)
	if input.UserInputType == Enum.UserInputType.Touch or input.UserInputType == Enum.UserInputType.MouseButton1 then
		draggingPanel = true
		dragStartPos = input.Position
		panelStartPos = panel.Position
		input.Changed:Connect(function()
			if input.UserInputState == Enum.UserInputState.End then
				draggingPanel = false
			end
		end)
	end
end)
panel.InputChanged:Connect(function(input)
	if draggingPanel and (input.UserInputType == Enum.UserInputType.Touch or input.UserInputType == Enum.UserInputType.MouseMovement) then
		local delta = input.Position - dragStartPos
		panel.Position = UDim2.new(panelStartPos.X.Scale, panelStartPos.X.Offset + delta.X, panelStartPos.Y.Scale, panelStartPos.Y.Offset + delta.Y)
	end
end)

-- SLIDER BEHAVIOR
local function attachSlider(bgFrame, barFrame, labelRef, fieldName)
	local dragging = false
	bgFrame.InputBegan:Connect(function(inp)
		if inp.UserInputType == Enum.UserInputType.Touch or inp.UserInputType == Enum.UserInputType.MouseButton1 then
			dragging = true
		end
	end)
	bgFrame.InputEnded:Connect(function(inp)
		if inp.UserInputType == Enum.UserInputType.Touch or inp.UserInputType == Enum.UserInputType.MouseButton1 then
			dragging = false
		end
	end)
	UserInputService.InputChanged:Connect(function(inp)
		if dragging and (inp.UserInputType == Enum.UserInputType.Touch or inp.UserInputType == Enum.UserInputType.MouseMovement) then
			local rel = math.clamp((inp.Position.X - bgFrame.AbsolutePosition.X) / bgFrame.AbsoluteSize.X, 0, 1)
			local newVal = math.floor(MIN_VALUE + (MAX_VALUE - MIN_VALUE) * rel)
			if fieldName == "walk" then
				walkSpeed = newVal
				if humanoid then humanoid.WalkSpeed = walkSpeed end
				labelRef.Text = "Velocidad Caminata: "..walkSpeed
			elseif fieldName == "flight" then
				flightSpeed = newVal
				labelRef.Text = "Velocidad Vuelo: "..flightSpeed
			elseif fieldName == "jump" then
				jumpPower = newVal
				if humanoid then
					humanoid.UseJumpPower = true
					humanoid.JumpPower = jumpPower
				end
				labelRef.Text = "Fuerza Salto: "..jumpPower
			end
			local percent = (newVal - MIN_VALUE)/(MAX_VALUE - MIN_VALUE)
			barFrame.Size = UDim2.new(percent, 0, 1, 0)
		end
	end)
end

attachSlider(walkBg, walkBar, walkLabel, "walk")
attachSlider(flightBg, flightBar, flightLabel, "flight")
attachSlider(jumpBg, jumpBar, jumpLabel, "jump")

-- init bars
local function initBars()
	local pWalk = (walkSpeed - MIN_VALUE)/(MAX_VALUE - MIN_VALUE)
	walkBar.Size = UDim2.new(pWalk, 0, 1, 0)
	walkLabel.Text = "Velocidad Caminata: "..walkSpeed
	local pFly = (flightSpeed - MIN_VALUE)/(MAX_VALUE - MIN_VALUE)
	flightBar.Size = UDim2.new(pFly, 0, 1, 0)
	flightLabel.Text = "Velocidad Vuelo: "..flightSpeed
	local pJump = (jumpPower - MIN_VALUE)/(MAX_VALUE - MIN_VALUE)
	jumpBar.Size = UDim2.new(pJump, 0, 1, 0)
	jumpLabel.Text = "Fuerza Salto: "..jumpPower
end
initBars()

-- FLIGHT (3D)
local function enableFlight()
	if flying then return end
	-- clean
	if bv then pcall(function() bv:Destroy() end) end
	if bg then pcall(function() bg:Destroy() end) end

	flying = true
	flyBtn.Text = "✈️ Vuelo: ON"
	flyBtn.BackgroundColor3 = Color3.fromRGB(80,200,120)

	bv = Instance.new("BodyVelocity")
	bv.Name = "UltraBV"
	bv.MaxForce = Vector3.new(1e6, 1e6, 1e6)
	bv.Velocity = Vector3.new(0,0,0)
	bv.P = 1e5
	bv.Parent = hrp

	bg = Instance.new("BodyGyro")
	bg.Name = "UltraBG"
	bg.MaxTorque = Vector3.new(1e6, 1e6, 1e6)
	bg.CFrame = hrp.CFrame
	bg.P = 1e5
	bg.Parent = hrp

	humanoid.PlatformStand = false
end

local function disableFlight()
	if not flying then return end
	flying = false
	flyBtn.Text = "✈️ Vuelo: OFF"
	flyBtn.BackgroundColor3 = Color3.fromRGB(65,65,85)
	if bv then pcall(function() bv:Destroy() end) end
	if bg then pcall(function() bg:Destroy() end) end
	bv = nil
	bg = nil
	-- stop vertical velocity
	if hrp then
		local v = hrp.Velocity
		hrp.Velocity = Vector3.new(v.X, 0, v.Z)
	end
end

upBtn.InputBegan:Connect(function(i) if i.UserInputType == Enum.UserInputType.Touch or i.UserInputType == Enum.UserInputType.MouseButton1 then ascendDir = 1 end end)
upBtn.InputEnded:Connect(function(i) if i.UserInputType == Enum.UserInputType.Touch or i.UserInputType == Enum.UserInputType.MouseButton1 then ascendDir = 0 end end)
downBtn.InputBegan:Connect(function(i) if i.UserInputType == Enum.UserInputType.Touch or i.UserInputType == Enum.UserInputType.MouseButton1 then ascendDir = -1 end end)
downBtn.InputEnded:Connect(function(i) if i.UserInputType == Enum.UserInputType.Touch or i.UserInputType == Enum.UserInputType.MouseButton1 then ascendDir = 0 end end)

flyBtn.MouseButton1Click:Connect(function() if flying then disableFlight() else enableFlight() end end)

-- Noclip functions
local function setCharacterCollisions(enable)
	-- enable = true -> character collides with world
	for _, part in ipairs(character:GetDescendants()) do
		if part:IsA("BasePart") and part.Name ~= "HumanoidRootPart" then
			part.CanCollide = enable
		end
	end
	-- hrp collision we control separately
	if hrp then hrp.CanCollide = enable end
end

local function tryEnableGroundCollision()
	-- raycast down a reasonable distance to detect ground
	local downDist = 6
	local r = groundHitFrom(hrp.Position, downDist)
	if r and r.Position then
		-- distance from HRP to hit point
		local dist = hrp.Position.Y - r.Position.Y
		-- if near ground (<= 3 stud), enable collisions so you stand and don't fall through
		if dist <= 3 then
			setCharacterCollisions(true)
			return true
		end
	end
	-- else keep collisions off so you can pass walls
	setCharacterCollisions(false)
	return false
end

noclipBtn.MouseButton1Click:Connect(function()
	noclip = not noclip
	if noclip then
		noclipBtn.Text = "🚧 Noclip: ON"
		noclipBtn.BackgroundColor3 = Color3.fromRGB(200,160,60)
		-- initially disable collisions, flight logic will handle ground detection
		setCharacterCollisions(false)
	else
		noclipBtn.Text = "🚧 Noclip: OFF"
		noclipBtn.BackgroundColor3 = Color3.fromRGB(65,65,85)
		-- restore collisions
		setCharacterCollisions(true)
	end
end)

-- Reset
resetBtn.MouseButton1Click:Connect(function()
	walkSpeed = DEFAULT_WALK
	flightSpeed = DEFAULT_FLIGHT
	jumpPower = DEFAULT_JUMP
	if humanoid then
		humanoid.WalkSpeed = walkSpeed
		humanoid.UseJumpPower = true
		humanoid.JumpPower = jumpPower
	end
	initBars()
end)

-- CORE LOOP: flight movement, noclip ground-check, sync walk/jump
RunService.RenderStepped:Connect(function(dt)
	if not character or not humanoid or not hrp then return end

	-- keep humanoid values consistent
	if humanoid.WalkSpeed ~= walkSpeed then humanoid.WalkSpeed = walkSpeed end
	if humanoid.JumpPower ~= jumpPower then
		humanoid.UseJumpPower = true
		humanoid.JumpPower = jumpPower
	end

	-- Noclip handling: if noclip active, try to enable/disable collisions based on proximity to ground
	if noclip then
		-- if very near ground, re-enable collisions so you stand; else disable to pass through walls
		tryEnableGroundCollision()
	end

	-- Flight handling
	if flying and bv and bg then
		local cam = workspace.CurrentCamera
		local moveDir = humanoid.MoveDirection -- mobile joystick or keyboard
		local look = cam and cam.CFrame or hrp.CFrame

		local forward = Vector3.new(look.LookVector.X, 0, look.LookVector.Z)
		if forward.Magnitude == 0 then forward = Vector3.new(0,0,-1) end
		forward = forward.Unit
		local right = Vector3.new(look.RightVector.X, 0, look.RightVector.Z).Unit

		local horizontal = (forward * moveDir.Z + right * moveDir.X)
		if horizontal.Magnitude > 1 then horizontal = horizontal.Unit end

		local vertical = ascendDir * math.max(1, flightSpeed * 0.7)
		local target = horizontal * flightSpeed + Vector3.new(0, vertical, 0)

		local cur = hrp.Velocity
		local smooth = cur:Lerp(target, math.clamp(12 * dt, 0, 1))
		bv.Velocity = Vector3.new(smooth.X, smooth.Y, smooth.Z)

		if horizontal.Magnitude > 0.1 then
			local targetLook = CFrame.new(hrp.Position, hrp.Position + Vector3.new(horizontal.X, 0, horizontal.Z))
			bg.CFrame = bg.CFrame:Lerp(targetLook, math.clamp(12 * dt, 0, 1))
		else
			bg.CFrame = bg.CFrame:Lerp(hrp.CFrame, math.clamp(6 * dt, 0, 1))
		end
	end
end)

-- Final apply initial values
humanoid.WalkSpeed = walkSpeed
humanoid.UseJumpPower = true
humanoid.JumpPower = jumpPower

-- Safety clamp loop & UI sync
task.spawn(function()
	while screenGui.Parent do
		-- clamp values
		walkSpeed = math.clamp(walkSpeed, MIN_VALUE, MAX_VALUE)
		flightSpeed = math.clamp(flightSpeed, MIN_VALUE, MAX_VALUE)
		jumpPower = math.clamp(jumpPower, MIN_VALUE, MAX_VALUE)

		-- update labels & bars
		walkLabel.Text = "Velocidad Caminata: "..walkSpeed
		flightLabel.Text = "Velocidad Vuelo: "..flightSpeed
		jumpLabel.Text = "Fuerza Salto: "..jumpPower

		walkBar.Size = UDim2.new((walkSpeed - MIN_VALUE)/(MAX_VALUE - MIN_VALUE), 0, 1, 0)
		flightBar.Size = UDim2.new((flightSpeed - MIN_VALUE)/(MAX_VALUE - MIN_VALUE), 0, 1, 0)
		jumpBar.Size = UDim2.new((jumpPower - MIN_VALUE)/(MAX_VALUE - MIN_VALUE), 0, 1, 0)

		-- ensure humanoid synced
		if humanoid and humanoid.WalkSpeed ~= walkSpeed then humanoid.WalkSpeed = walkSpeed end
		if humanoid and humanoid.JumpPower ~= jumpPower then
			humanoid.UseJumpPower = true
			humanoid.JumpPower = jumpPower
		end

		task.wait(0.12)
	end
end)

-- End of script
