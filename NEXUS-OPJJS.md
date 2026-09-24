-- Services
local Players = game:GetService("Players")
local UserInputService = game:GetService("UserInputService")
local RunService = game:GetService("RunService")
local TweenService = game:GetService("TweenService")

local LocalPlayer = Players.LocalPlayer
local Camera = workspace.CurrentCamera

-- Global Target Exposer
_G.LockOnTarget = nil

-- System Variables
local isLockOnEnabled = false
local isInstaDashEnabled = false
local isLockedOn = false

local lockKey = Enum.KeyCode.E
local instaDashKey = Enum.KeyCode.Q

local targetCharacter = nil
local bindingLockKey = false
local bindingDashKey = false

-- Dash Hold & Cooldown Variables
local isHoldingDashKey = false
local lastDashTime = 0
local DASH_COOLDOWN = 0.25

-- Theme Configurations
local Themes = {
	Cyberpunk = {
		Accent = Color3.fromRGB(0, 230, 255),
		MainBg = Color3.fromRGB(12, 14, 22),
		CardBg = Color3.fromRGB(18, 22, 34),
		Stroke = Color3.fromRGB(40, 50, 75)
	},
	Midnight = {
		Accent = Color3.fromRGB(170, 80, 255),
		MainBg = Color3.fromRGB(16, 12, 24),
		CardBg = Color3.fromRGB(24, 18, 38),
		Stroke = Color3.fromRGB(60, 40, 85)
	},
	BloodMoon = {
		Accent = Color3.fromRGB(255, 45, 75),
		MainBg = Color3.fromRGB(20, 10, 12),
		CardBg = Color3.fromRGB(32, 16, 20),
		Stroke = Color3.fromRGB(80, 35, 45)
	},
	Matrix = {
		Accent = Color3.fromRGB(0, 255, 120),
		MainBg = Color3.fromRGB(10, 18, 14),
		CardBg = Color3.fromRGB(15, 28, 20),
		Stroke = Color3.fromRGB(30, 65, 45)
	}
}

local currentTheme = Themes.Cyberpunk
local EXPANDED_SIZE = UDim2.new(0, 320, 0, 360)

-- Create ScreenGui
local screenGui = Instance.new("ScreenGui")
screenGui.Name = "CyberLockGui"
screenGui.ResetOnSpawn = false
screenGui.Parent = LocalPlayer:WaitForChild("PlayerGui")

-- Main Glassmorphism Frame
local mainFrame = Instance.new("Frame")
mainFrame.Name = "MainFrame"
mainFrame.Size = EXPANDED_SIZE
mainFrame.Position = UDim2.new(0.04, 0, 0.28, 0)
mainFrame.BackgroundColor3 = currentTheme.MainBg
mainFrame.BackgroundTransparency = 0.12
mainFrame.BorderSizePixel = 0
mainFrame.Active = true
mainFrame.Draggable = true
mainFrame.ClipsDescendants = true
mainFrame.Parent = screenGui

local corner = Instance.new("UICorner")
corner.CornerRadius = UDim.new(0, 18)
corner.Parent = mainFrame

local glowStroke = Instance.new("UIStroke")
glowStroke.Color = currentTheme.Accent
glowStroke.Thickness = 2
glowStroke.Transparency = 0.25
glowStroke.Parent = mainFrame

local padding = Instance.new("UIPadding")
padding.PaddingTop = UDim.new(0, 16)
padding.PaddingBottom = UDim.new(0, 16)
padding.PaddingLeft = UDim.new(0, 16)
padding.PaddingRight = UDim.new(0, 16)
padding.Parent = mainFrame

-- Header Container
local headerContainer = Instance.new("Frame")
headerContainer.Size = UDim2.new(1, 0, 0, 36)
headerContainer.BackgroundTransparency = 1
headerContainer.Parent = mainFrame

local title = Instance.new("TextLabel")
title.Size = UDim2.new(0.7, 0, 1, 0)
title.BackgroundTransparency = 1
title.Text = "NEXUS // OPJJS"
title.TextColor3 = Color3.fromRGB(255, 255, 255)
title.TextSize = 16
title.Font = Enum.Font.Michroma
title.TextXAlignment = Enum.TextXAlignment.Left
title.Parent = headerContainer

local statusDot = Instance.new("Frame")
statusDot.Size = UDim2.new(0, 10, 0, 10)
statusDot.Position = UDim2.new(0.74, 0, 0.35, 0)
statusDot.BackgroundColor3 = Color3.fromRGB(255, 50, 80)
statusDot.Parent = headerContainer

local dotCorner = Instance.new("UICorner")
dotCorner.CornerRadius = UDim.new(1, 0)
dotCorner.Parent = statusDot

-- Close Button [X]
local closeBtn = Instance.new("TextButton")
closeBtn.Name = "CloseBtn"
closeBtn.Size = UDim2.new(0, 26, 0, 26)
closeBtn.Position = UDim2.new(0.88, 0, 0.1, 0)
closeBtn.BackgroundColor3 = Color3.fromRGB(255, 45, 75)
closeBtn.Text = "✕"
closeBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
closeBtn.TextSize = 13
closeBtn.Font = Enum.Font.GothamBold
closeBtn.Parent = headerContainer

local closeBtnCorner = Instance.new("UICorner")
closeBtnCorner.CornerRadius = UDim.new(0, 8)
closeBtnCorner.Parent = closeBtn

-- Divider
local divider = Instance.new("Frame")
divider.Size = UDim2.new(1, 0, 0, 2)
divider.Position = UDim2.new(0, 0, 0.13, 0)
divider.BackgroundColor3 = currentTheme.Accent
divider.BackgroundTransparency = 0.6
divider.BorderSizePixel = 0
divider.Parent = mainFrame

-- Scrolling Canvas Area
local scrollFrame = Instance.new("ScrollingFrame")
scrollFrame.Size = UDim2.new(1, 0, 0.84, 0)
scrollFrame.Position = UDim2.new(0, 0, 0.16, 0)
scrollFrame.BackgroundTransparency = 1
scrollFrame.BorderSizePixel = 0
scrollFrame.ScrollBarThickness = 4
scrollFrame.ScrollBarImageColor3 = currentTheme.Accent
scrollFrame.CanvasSize = UDim2.new(0, 0, 0, 370)
scrollFrame.Parent = mainFrame

local listLayout = Instance.new("UIListLayout")
listLayout.SortOrder = Enum.SortOrder.LayoutOrder
listLayout.Padding = UDim.new(0, 12)
listLayout.Parent = scrollFrame

-- Track UI elements for dynamic color re-skinning
local themeableStrokes = {}
local themeableTextBtns = {}
local cardContainers = {}

-- Card Builder
local function createModule(layoutOrder, labelText, defaultKeyText)
	local container = Instance.new("Frame")
	container.Size = UDim2.new(0.96, 0, 0, 108)
	container.LayoutOrder = layoutOrder
	container.BackgroundColor3 = currentTheme.CardBg
	container.BackgroundTransparency = 0.25
	container.Parent = scrollFrame
	table.insert(cardContainers, container)

	local moduleCorner = Instance.new("UICorner")
	moduleCorner.CornerRadius = UDim.new(0, 12)
	moduleCorner.Parent = container

	local moduleStroke = Instance.new("UIStroke")
	moduleStroke.Color = currentTheme.Stroke
	moduleStroke.Thickness = 1.2
	moduleStroke.Parent = container
	table.insert(themeableStrokes, moduleStroke)

	local label = Instance.new("TextLabel")
	label.Size = UDim2.new(0.65, 0, 0, 32)
	label.Position = UDim2.new(0.06, 0, 0.08, 0)
	label.BackgroundTransparency = 1
	label.Text = labelText
	label.TextColor3 = Color3.fromRGB(210, 225, 250)
	label.TextSize = 13
	label.Font = Enum.Font.GothamBold
	label.TextXAlignment = Enum.TextXAlignment.Left
	label.Parent = container

	local toggleBtn = Instance.new("TextButton")
	toggleBtn.Size = UDim2.new(0.28, 0, 0, 28)
	toggleBtn.Position = UDim2.new(0.66, 0, 0.1, 0)
	toggleBtn.BackgroundColor3 = Color3.fromRGB(255, 50, 80)
	toggleBtn.Text = "OFF"
	toggleBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
	toggleBtn.TextSize = 11
	toggleBtn.Font = Enum.Font.GothamBold
	toggleBtn.Parent = container

	local toggleBtnCorner = Instance.new("UICorner")
	toggleBtnCorner.CornerRadius = UDim.new(0, 8)
	toggleBtnCorner.Parent = toggleBtn

	local keyBtn = Instance.new("TextButton")
	keyBtn.Size = UDim2.new(0.88, 0, 0, 36)
	keyBtn.Position = UDim2.new(0.06, 0, 0.55, 0)
	keyBtn.BackgroundColor3 = Color3.fromRGB(10, 12, 18)
	keyBtn.Text = defaultKeyText
	keyBtn.TextColor3 = currentTheme.Accent
	keyBtn.TextSize = 12
	keyBtn.Font = Enum.Font.Code
	keyBtn.Parent = container
	table.insert(themeableTextBtns, keyBtn)

	local keyBtnCorner = Instance.new("UICorner")
	keyBtnCorner.CornerRadius = UDim.new(0, 8)
	keyBtnCorner.Parent = keyBtn

	local keyBtnStroke = Instance.new("UIStroke")
	keyBtnStroke.Color = currentTheme.Accent
	keyBtnStroke.Thickness = 1
	keyBtnStroke.Transparency = 0.7
	keyBtnStroke.Parent = keyBtn

	return container, toggleBtn, keyBtn
end

local lockModule, lockToggleBtn, lockKeyBtn = createModule(1, "TARGET LOCK", "KEYBIND: [ E ]")
local dashModule, dashToggleBtn, dashKeyBtn = createModule(2, "INSTA BACK-DASH", "KEYBIND: [ Q ]")

-- Theme Selector Card
local themeModule = Instance.new("Frame")
themeModule.Size = UDim2.new(0.96, 0, 0, 110)
themeModule.LayoutOrder = 3
themeModule.BackgroundColor3 = currentTheme.CardBg
themeModule.BackgroundTransparency = 0.25
themeModule.Parent = scrollFrame
table.insert(cardContainers, themeModule)

local themeCorner = Instance.new("UICorner")
themeCorner.CornerRadius = UDim.new(0, 12)
themeCorner.Parent = themeModule

local themeStroke = Instance.new("UIStroke")
themeStroke.Color = currentTheme.Stroke
themeStroke.Thickness = 1.2
themeStroke.Parent = themeModule
table.insert(themeableStrokes, themeStroke)

local themeLabel = Instance.new("TextLabel")
themeLabel.Size = UDim2.new(1, 0, 0, 28)
themeLabel.Position = UDim2.new(0.06, 0, 0.08, 0)
themeLabel.BackgroundTransparency = 1
themeLabel.Text = "THEME SELECTOR"
themeLabel.TextColor3 = Color3.fromRGB(210, 225, 250)
themeLabel.TextSize = 13
themeLabel.Font = Enum.Font.GothamBold
themeLabel.TextXAlignment = Enum.TextXAlignment.Left
themeLabel.Parent = themeModule

local themeBtnGrid = Instance.new("Frame")
themeBtnGrid.Size = UDim2.new(0.88, 0, 0, 50)
themeBtnGrid.Position = UDim2.new(0.06, 0, 0.42, 0)
themeBtnGrid.BackgroundTransparency = 1
themeBtnGrid.Parent = themeModule

local gridLayout = Instance.new("UIGridLayout")
gridLayout.CellSize = UDim2.new(0.46, 0, 0, 22)
gridLayout.CellPadding = UDim2.new(0.08, 0, 0, 6)
gridLayout.Parent = themeBtnGrid

-- Apply Theme Utility Function
local function applyTheme(theme)
	currentTheme = theme
	mainFrame.BackgroundColor3 = theme.MainBg
	glowStroke.Color = theme.Accent
	divider.BackgroundColor3 = theme.Accent
	scrollFrame.ScrollBarImageColor3 = theme.Accent

	for _, card in ipairs(cardContainers) do
		card.BackgroundColor3 = theme.CardBg
	end
	for _, stroke in ipairs(themeableStrokes) do
		stroke.Color = theme.Stroke
	end
	for _, btn in ipairs(themeableTextBtns) do
		btn.TextColor3 = theme.Accent
	end
end

-- Create Theme Selector Buttons
local function createThemeBtn(name, themeData)
	local btn = Instance.new("TextButton")
	btn.Name = name
	btn.BackgroundColor3 = themeData.Accent
	btn.Text = name:upper()
	btn.TextColor3 = Color3.fromRGB(10, 10, 15)
	btn.TextSize = 10
	btn.Font = Enum.Font.GothamBold
	btn.Parent = themeBtnGrid

	local btnCorner = Instance.new("UICorner")
	btnCorner.CornerRadius = UDim.new(0, 6)
	btnCorner.Parent = btn

	btn.MouseButton1Click:Connect(function()
		applyTheme(themeData)
	end)
end

createThemeBtn("Cyber", Themes.Cyberpunk)
createThemeBtn("Midnight", Themes.Midnight)
createThemeBtn("Blood", Themes.BloodMoon)
createThemeBtn("Matrix", Themes.Matrix)

-- Target Search Function
local function getClosestTarget()
	local closestTarget = nil
	local shortestDistance = math.huge
	
	if not LocalPlayer.Character or not LocalPlayer.Character:FindFirstChild("HumanoidRootPart") then
		return nil
	end
	
	local myPos = LocalPlayer.Character.HumanoidRootPart.Position

	for _, obj in ipairs(workspace:GetDescendants()) do
		if obj:IsA("Model") and obj ~= LocalPlayer.Character then
			local humanoid = obj:FindFirstChildOfClass("Humanoid")
			local rootPart = obj:FindFirstChild("HumanoidRootPart")
			
			if humanoid and rootPart and humanoid.Health > 0 then
				local distance = (rootPart.Position - myPos).Magnitude
				if distance < shortestDistance and distance < 150 then
					shortestDistance = distance
					closestTarget = obj
				end
			end
		end
	end
	return closestTarget
end

-- Perform Instant Back-Dash
local function performInstaBackDash()
	if not isInstaDashEnabled then return end
	if not targetCharacter or not targetCharacter:FindFirstChild("HumanoidRootPart") then return end
	if os.clock() - lastDashTime < DASH_COOLDOWN then return end
	
	local myChar = LocalPlayer.Character
	if not myChar or not myChar:FindFirstChild("HumanoidRootPart") then return end
	
	local myRoot = myChar.HumanoidRootPart
	local targetRoot = targetCharacter.HumanoidRootPart
	
	local targetLook = targetRoot.CFrame.LookVector
	local behindPosition = targetRoot.Position - (targetLook * 4.5)
	local newCFrame = CFrame.new(behindPosition, targetRoot.Position)
	
	lastDashTime = os.clock()
	
	local tweenInfo = TweenInfo.new(0.08, Enum.EasingStyle.Quad, Enum.EasingDirection.Out)
	local tween = TweenService:Create(myRoot, tweenInfo, {CFrame = newCFrame})
	tween:Play()
end

-- Status Dot Indicator
local function updateStatusDot()
	if isLockOnEnabled or isInstaDashEnabled then
		statusDot.BackgroundColor3 = Color3.fromRGB(0, 255, 150)
	else
		statusDot.BackgroundColor3 = Color3.fromRGB(255, 50, 80)
	end
end

-- Animated Minimize
local isMinimized = false
local function toggleMinimize()
	isMinimized = not isMinimized
	
	local targetSize = isMinimized and UDim2.new(0, 0, 0, 0) or EXPANDED_SIZE
	local targetTransparency = isMinimized and 1 or 0.12
	local targetStrokeTrans = isMinimized and 1 or 0.25
	
	local tweenInfo = TweenInfo.new(0.25, Enum.EasingStyle.Quad, Enum.EasingDirection.Out)
	
	if not isMinimized then
		mainFrame.Visible = true
	end
	
	local sizeTween = TweenService:Create(mainFrame, tweenInfo, {
		Size = targetSize,
		BackgroundTransparency = targetTransparency
	})
	
	TweenService:Create(glowStroke, tweenInfo, {Transparency = targetStrokeTrans}):Play()
	sizeTween:Play()
	
	sizeTween.Completed:Connect(function()
		if isMinimized then
			mainFrame.Visible = false
		end
	end)
end

-- Close/Destroy GUI
closeBtn.MouseButton1Click:Connect(function()
	local closeTween = TweenService:Create(mainFrame, TweenInfo.new(0.2, Enum.EasingStyle.Quad, Enum.EasingDirection.In), {
		Size = UDim2.new(0, 0, 0, 0),
		BackgroundTransparency = 1
	})
	closeTween:Play()
	closeTween.Completed:Connect(function()
		_G.LockOnTarget = nil
		screenGui:Destroy()
	end)
end)

-- Button Listeners
lockToggleBtn.MouseButton1Click:Connect(function()
	isLockOnEnabled = not isLockOnEnabled
	if isLockOnEnabled then
		lockToggleBtn.BackgroundColor3 = Color3.fromRGB(0, 255, 150)
		lockToggleBtn.Text = "ACTIVE"
		lockToggleBtn.TextColor3 = Color3.fromRGB(10, 20, 15)
	else
		lockToggleBtn.BackgroundColor3 = Color3.fromRGB(255, 50, 80)
		lockToggleBtn.Text = "OFF"
		lockToggleBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
		isLockedOn = false
		targetCharacter = nil
		_G.LockOnTarget = nil
	end
	updateStatusDot()
end)

dashToggleBtn.MouseButton1Click:Connect(function()
	isInstaDashEnabled = not isInstaDashEnabled
	if isInstaDashEnabled then
		dashToggleBtn.BackgroundColor3 = Color3.fromRGB(0, 255, 150)
		dashToggleBtn.Text = "ACTIVE"
		dashToggleBtn.TextColor3 = Color3.fromRGB(10, 20, 15)
	else
		dashToggleBtn.BackgroundColor3 = Color3.fromRGB(255, 50, 80)
		dashToggleBtn.Text = "OFF"
		dashToggleBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
	end
	updateStatusDot()
end)

lockKeyBtn.MouseButton1Click:Connect(function()
	bindingLockKey = true
	lockKeyBtn.Text = "AWAITING INPUT..."
end)

dashKeyBtn.MouseButton1Click:Connect(function()
	bindingDashKey = true
	dashKeyBtn.Text = "AWAITING INPUT..."
end)

-- Input Event Listeners
UserInputService.InputBegan:Connect(function(input, gameProcessed)
	if gameProcessed then return end
	
	-- Left Control: Animate Minimize / Expand
	if input.KeyCode == Enum.KeyCode.LeftControl then
		toggleMinimize()
		return
	end
	
	-- Keybind Remappings
	if bindingLockKey and input.UserInputType == Enum.UserInputType.Keyboard then
		lockKey = input.KeyCode
		lockKeyBtn.Text = "KEYBIND: [ " .. lockKey.Name .. " ]"
		bindingLockKey = false
		return
	end

	if bindingDashKey and input.UserInputType == Enum.UserInputType.Keyboard then
		instaDashKey = input.KeyCode
		dashKeyBtn.Text = "KEYBIND: [ " .. instaDashKey.Name .. " ]"
		bindingDashKey = false
		return
	end
	
	-- Target Lock Toggle
	if isLockOnEnabled and input.KeyCode == lockKey then
		if isLockedOn then
			isLockedOn = false
			targetCharacter = nil
			_G.LockOnTarget = nil
		else
			targetCharacter = getClosestTarget()
			if targetCharacter then
				isLockedOn = true
				_G.LockOnTarget = targetCharacter
			end
		end
		return
	end

	-- Insta-Dash Key Hold
	if input.KeyCode == instaDashKey then
		isHoldingDashKey = true
		performInstaBackDash()
	end
end)

UserInputService.InputEnded:Connect(function(input)
	if input.KeyCode == instaDashKey then
		isHoldingDashKey = false
	end
end)

-- Render Loop
RunService.RenderStepped:Connect(function()
	if isHoldingDashKey then
		performInstaBackDash()
	end

	if isLockOnEnabled and isLockedOn and targetCharacter and targetCharacter:FindFirstChild("HumanoidRootPart") then
		local targetHumanoid = targetCharacter:FindFirstChildOfClass("Humanoid")
		if targetHumanoid and targetHumanoid.Health > 0 then
			local targetPos = targetCharacter.HumanoidRootPart.Position
			local cameraPos = Camera.CFrame.Position
			
			Camera.CFrame = CFrame.new(cameraPos, targetPos)
			_G.LockOnTarget = targetCharacter
		else
			isLockedOn = false
			targetCharacter = nil
			_G.LockOnTarget = nil
		end
	else
		_G.LockOnTarget = nil
	end
end)
