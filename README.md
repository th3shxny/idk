--// CAM AIM TEST MENU
--// Roblox Studio LocalScript
--// Place in: StarterPlayer > StarterPlayerScripts
--// Studio testing only - no exploit/executor functionality

local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local UserInputService = game:GetService("UserInputService")

local LocalPlayer = Players.LocalPlayer

--==================================================
-- SETTINGS
--==================================================

local AimLockEnabled = false
local MenuVisible = true
local SettingsOpen = false
local TeleportOpen = false
local WaitingForKey = false

local ESPEnabled = true

local FOV = 90
local Smoothness = 0.50
local WalkSpeed = 20

local ToggleKey = Enum.KeyCode.Q
local MenuKey = Enum.KeyCode.Z

local CurrentTarget = nil
local ESPObjects = {}

local TeleportSlots = {}
for i = 1, 10 do
	TeleportSlots[i] = nil
end

--==================================================
-- CHARACTER HELPERS
--==================================================

local function getCharacter(player)
	if not player then
		return nil
	end

	local character = player.Character
	if not character then
		return nil
	end

	local humanoid = character:FindFirstChildOfClass("Humanoid")
	local head = character:FindFirstChild("Head")
	local root = character:FindFirstChild("HumanoidRootPart")

	if not humanoid or not head or not root then
		return nil
	end

	if humanoid.Health <= 0 then
		return nil
	end

	return character, humanoid, head, root
end

local function getLocalCharacter()
	return getCharacter(LocalPlayer)
end

--==================================================
-- GUI HELPERS
--==================================================

local function create(className, properties, parent)
	local object = Instance.new(className)

	for property, value in pairs(properties or {}) do
		object[property] = value
	end

	if parent then
		object.Parent = parent
	end

	return object
end

local function corner(parent, radius)
	return create("UICorner", {
		CornerRadius = UDim.new(0, radius or 8)
	}, parent)
end

local function stroke(parent, color, thickness)
	return create("UIStroke", {
		Color = color or Color3.fromRGB(55, 55, 55),
		Thickness = thickness or 1
	}, parent)
end

local function makeButton(parent, text, position, size)
	local button = create("TextButton", {
		BackgroundColor3 = Color3.fromRGB(28, 28, 28),
		BorderSizePixel = 0,
		Position = position,
		Size = size,
		Text = text,
		TextColor3 = Color3.fromRGB(235, 235, 235),
		TextSize = 14,
		Font = Enum.Font.GothamMedium,
		AutoButtonColor = true
	}, parent)

	corner(button, 7)
	stroke(button)

	return button
end

local function makeLabel(parent, text, position, size, textSize)
	return create("TextLabel", {
		BackgroundTransparency = 1,
		Position = position,
		Size = size,
		Text = text,
		TextColor3 = Color3.fromRGB(225, 225, 225),
		TextSize = textSize or 14,
		Font = Enum.Font.GothamMedium,
		TextXAlignment = Enum.TextXAlignment.Left
	}, parent)
end

--==================================================
-- SCREEN GUI
--==================================================

local oldGui = LocalPlayer:WaitForChild("PlayerGui"):FindFirstChild("CamAimTestMenu")
if oldGui then
	oldGui:Destroy()
end

local ScreenGui = create("ScreenGui", {
	Name = "CamAimTestMenu",
	ResetOnSpawn = false,
	ZIndexBehavior = Enum.ZIndexBehavior.Sibling
}, LocalPlayer:WaitForChild("PlayerGui"))

--==================================================
-- MAIN FRAME
--==================================================

local MainFrame = create("Frame", {
	Name = "MainFrame",
	AnchorPoint = Vector2.new(0.5, 0.5),
	Position = UDim2.fromScale(0.5, 0.5),
	Size = UDim2.fromOffset(460, 570),
	BackgroundColor3 = Color3.fromRGB(13, 13, 13),
	BorderSizePixel = 0
}, ScreenGui)

corner(MainFrame, 12)
stroke(MainFrame, Color3.fromRGB(55, 55, 55), 1)

--==================================================
-- TITLE BAR
--==================================================

local TitleBar = create("Frame", {
	Name = "TitleBar",
	Position = UDim2.fromOffset(0, 0),
	Size = UDim2.new(1, 0, 0, 48),
	BackgroundColor3 = Color3.fromRGB(18, 18, 18),
	BorderSizePixel = 0,
	Active = true
}, MainFrame)

corner(TitleBar, 12)

local Title = makeLabel(
	TitleBar,
	"Cam Aim Tester",
	UDim2.fromOffset(16, 0),
	UDim2.new(1, -100, 1, 0),
	17
)

Title.Font = Enum.Font.GothamBold

local HideButton = makeButton(
	TitleBar,
	"—",
	UDim2.new(1, -82, 0, 8),
	UDim2.fromOffset(32, 32)
)

local SettingsButton = makeButton(
	TitleBar,
	"⚙",
	UDim2.new(1, -44, 0, 8),
	UDim2.fromOffset(32, 32)
)

--==================================================
-- DRAGGING
--==================================================

local dragging = false
local dragStart
local startPosition

TitleBar.InputBegan:Connect(function(input)
	if input.UserInputType == Enum.UserInputType.MouseButton1 then
		dragging = true
		dragStart = input.Position
		startPosition = MainFrame.Position
	end
end)

TitleBar.InputEnded:Connect(function(input)
	if input.UserInputType == Enum.UserInputType.MouseButton1 then
		dragging = false
	end
end)

UserInputService.InputChanged:Connect(function(input)
	if dragging and input.UserInputType == Enum.UserInputType.MouseMovement then
		local delta = input.Position - dragStart

		MainFrame.Position = UDim2.new(
			startPosition.X.Scale,
			startPosition.X.Offset + delta.X,
			startPosition.Y.Scale,
			startPosition.Y.Offset + delta.Y
		)
	end
end)

--==================================================
-- MAIN CONTENT
--==================================================

local MainContent = create("Frame", {
	Name = "MainContent",
	Position = UDim2.fromOffset(0, 48),
	Size = UDim2.new(1, 0, 1, -48),
	BackgroundTransparency = 1
}, MainFrame)

--==================================================
-- AIM LOCK
--==================================================

local AimButton = makeButton(
	MainContent,
	"AIM LOCK: OFF",
	UDim2.fromOffset(15, 15),
	UDim2.fromOffset(210, 42)
)

local LockedLabel = makeLabel(
	MainContent,
	"Locked Onto: None",
	UDim2.fromOffset(240, 15),
	UDim2.fromOffset(205, 42),
	13
)

LockedLabel.TextWrapped = true

local function updateAimButton()
	if AimLockEnabled then
		AimButton.Text = "AIM LOCK: ON"
		AimButton.BackgroundColor3 = Color3.fromRGB(40, 40, 40)
	else
		AimButton.Text = "AIM LOCK: OFF"
		AimButton.BackgroundColor3 = Color3.fromRGB(28, 28, 28)
	end
end

--==================================================
-- ESP
--==================================================

local ESPButton = makeButton(
	MainContent,
	"ESP: ON",
	UDim2.fromOffset(15, 70),
	UDim2.fromOffset(430, 38)
)

--==================================================
-- SLIDER CREATOR
--==================================================

local function createSlider(parent, title, y, minimum, maximum, defaultValue, callback)

	local label = makeLabel(
		parent,
		title,
		UDim2.fromOffset(15, y),
		UDim2.fromOffset(180, 25),
		13
	)

	local valueLabel = makeLabel(
		parent,
		tostring(defaultValue),
		UDim2.new(1, -100, 0, y),
		UDim2.fromOffset(85, 25),
		13
	)

	valueLabel.TextXAlignment = Enum.TextXAlignment.Right

	local bar = create("Frame", {
		Position = UDim2.fromOffset(15, y + 28),
		Size = UDim2.fromOffset(430, 7),
		BackgroundColor3 = Color3.fromRGB(35, 35, 35),
		BorderSizePixel = 0
	}, parent)

	corner(bar, 5)

	local fill = create("Frame", {
		Size = UDim2.new(
			(defaultValue - minimum) / (maximum - minimum),
			0,
			1,
			0
		),
		BackgroundColor3 = Color3.fromRGB(130, 130, 130),
		BorderSizePixel = 0
	}, bar)

	corner(fill, 5)

	local knob = create("Frame", {
		AnchorPoint = Vector2.new(0.5, 0.5),
		Position = UDim2.new(
			(defaultValue - minimum) / (maximum - minimum),
			0,
			0.5,
			0
		),
		Size = UDim2.fromOffset(13, 13),
		BackgroundColor3 = Color3.fromRGB(235, 235, 235),
		BorderSizePixel = 0
	}, bar)

	corner(knob, 10)

	local sliderButton = create("TextButton", {
		BackgroundTransparency = 1,
		Position = UDim2.fromOffset(0, -8),
		Size = UDim2.new(1, 0, 1, 23),
		Text = ""
	}, bar)

	local draggingSlider = false

	local function setValueFromX(x)
		local relative = math.clamp(
			(x - bar.AbsolutePosition.X) / bar.AbsoluteSize.X,
			0,
			1
		)

		local value = minimum + ((maximum - minimum) * relative)

		if maximum - minimum <= 10 then
			value = math.round(value * 100) / 100
		else
			value = math.round(value)
		end

		local percentage = (value - minimum) / (maximum - minimum)

		fill.Size = UDim2.new(percentage, 0, 1, 0)
		knob.Position = UDim2.new(percentage, 0, 0.5, 0)
		valueLabel.Text = tostring(value)

		callback(value)
	end

	sliderButton.InputBegan:Connect(function(input)
		if input.UserInputType == Enum.UserInputType.MouseButton1 then
			draggingSlider = true
			setValueFromX(input.Position.X)
		end
	end)

	sliderButton.InputEnded:Connect(function(input)
		if input.UserInputType == Enum.UserInputType.MouseButton1 then
			draggingSlider = false
		end
	end)

	UserInputService.InputChanged:Connect(function(input)
		if draggingSlider and input.UserInputType == Enum.UserInputType.MouseMovement then
			setValueFromX(input.Position.X)
		end
	end)

	return {
		SetValue = function(value)
			value = math.clamp(value, minimum, maximum)

			local percentage = (value - minimum) / (maximum - minimum)

			fill.Size = UDim2.new(percentage, 0, 1, 0)
			knob.Position = UDim2.new(percentage, 0, 0.5, 0)
			valueLabel.Text = tostring(value)

			callback(value)
		end
	}
end

--==================================================
-- SLIDERS
--==================================================

createSlider(
	MainContent,
	"FOV",
	120,
	20,
	180,
	FOV,
	function(value)
		FOV = value
	end
)

createSlider(
	MainContent,
	"Smoothness",
	180,
	0.05,
	1,
	Smoothness,
	function(value)
		Smoothness = value
	end
)

createSlider(
	MainContent,
	"Walk Speed",
	240,
	8,
	100,
	WalkSpeed,
	function(value)
		WalkSpeed = value

		local character, humanoid = getLocalCharacter()

		if humanoid then
			humanoid.WalkSpeed = WalkSpeed
		end
	end
)

--==================================================
-- PLAYER DROPDOWN
--==================================================

local PlayerLabel = makeLabel(
	MainContent,
	"Test Player",
	UDim2.fromOffset(15, 305),
	UDim2.fromOffset(200, 25),
	13
)

local PlayerDropdown = makeButton(
	MainContent,
	"Select Test Player",
	UDim2.fromOffset(15, 330),
	UDim2.fromOffset(430, 38)
)

local SelectedPlayer = nil
local DropdownOpen = false

local PlayerListFrame = create("ScrollingFrame", {
	Name = "PlayerList",
	Position = UDim2.fromOffset(15, 372),
	Size = UDim2.fromOffset(430, 0),
	BackgroundColor3 = Color3.fromRGB(20, 20, 20),
	BorderSizePixel = 0,
	ScrollBarThickness = 5,
	CanvasSize = UDim2.new(),
	Visible = false,
	ZIndex = 20
}, MainContent)

corner(PlayerListFrame, 7)
stroke(PlayerListFrame)

local PlayerListLayout = create("UIListLayout", {
	Padding = UDim.new(0, 2),
	SortOrder = Enum.SortOrder.Name
}, PlayerListFrame)

local function refreshPlayerDropdown()

	for _, child in ipairs(PlayerListFrame:GetChildren()) do
		if child:IsA("TextButton") then
			child:Destroy()
		end
	end

	local count = 0

	for _, targetPlayer in ipairs(Players:GetPlayers()) do

		if targetPlayer ~= LocalPlayer then

			count += 1

			local button = makeButton(
				PlayerListFrame,
				targetPlayer.Name,
				UDim2.new(),
				UDim2.new(1, -8, 0, 34)
			)

			button.ZIndex = 21

			button.Activated:Connect(function()

				SelectedPlayer = targetPlayer
				PlayerDropdown.Text = targetPlayer.Name

				DropdownOpen = false
				PlayerListFrame.Visible = false
				PlayerListFrame.Size = UDim2.fromOffset(430, 0)
			end)
		end
	end

	local height = math.clamp(count * 36, 0, 150)

	PlayerListFrame.CanvasSize = UDim2.fromOffset(0, count * 36)
	PlayerListFrame.Size = UDim2.fromOffset(430, height)
end

PlayerDropdown.Activated:Connect(function()

	DropdownOpen = not DropdownOpen
	PlayerListFrame.Visible = DropdownOpen

	if DropdownOpen then
		refreshPlayerDropdown()
	end
end)

--==================================================
-- TELEPORT BUTTON
--==================================================

local TeleportButton = makeButton(
	MainContent,
	"OPEN TELEPORTER",
	UDim2.fromOffset(15, 430),
	UDim2.fromOffset(430, 40)
)

--==================================================
-- SETTINGS PANEL
--==================================================

local SettingsPanel = create("Frame", {
	Name = "SettingsPanel",
	Position = UDim2.fromOffset(0, 0),
	Size = UDim2.fromScale(1, 1),
	BackgroundColor3 = Color3.fromRGB(13, 13, 13),
	BorderSizePixel = 0,
	Visible = false,
	ZIndex = 50
}, MainFrame)

corner(SettingsPanel, 12)
stroke(SettingsPanel, Color3.fromRGB(55, 55, 55))

local SettingsTitle = makeLabel(
	SettingsPanel,
	"Settings",
	UDim2.fromOffset(18, 15),
	UDim2.fromOffset(300, 35),
	19
)

SettingsTitle.Font = Enum.Font.GothamBold

local HotkeyLabel = makeLabel(
	SettingsPanel,
	"Aim Lock Hotkey",
	UDim2.fromOffset(18, 75),
	UDim2.fromOffset(300, 30),
	14
)

local HotkeyButton = makeButton(
	SettingsPanel,
	"Q",
	UDim2.fromOffset(18, 110),
	UDim2.fromOffset(424, 42)
)

local SettingsInfo = makeLabel(
	SettingsPanel,
	"Click the button, then press any keyboard key.",
	UDim2.fromOffset(18, 160),
	UDim2.fromOffset(424, 30),
	12
)

SettingsInfo.TextColor3 = Color3.fromRGB(150, 150, 150)

local SettingsBack = makeButton(
	SettingsPanel,
	"BACK",
	UDim2.fromOffset(18, 485),
	UDim2.fromOffset(424, 42)
)

--==================================================
-- TELEPORT PANEL
--==================================================

local TeleportPanel = create("Frame", {
	Name = "TeleportPanel",
	Position = UDim2.fromOffset(0, 0),
	Size = UDim2.fromScale(1, 1),
	BackgroundColor3 = Color3.fromRGB(13, 13, 13),
	BorderSizePixel = 0,
	Visible = false,
	ZIndex = 50
}, MainFrame)

corner(TeleportPanel, 12)
stroke(TeleportPanel, Color3.fromRGB(55, 55, 55))

local TeleportTitle = makeLabel(
	TeleportPanel,
	"Teleport Locations",
	UDim2.fromOffset(18, 15),
	UDim2.fromOffset(350, 35),
	19
)

TeleportTitle.Font = Enum.Font.GothamBold

local CurrentPositionLabel = makeLabel(
	TeleportPanel,
	"Current Position: 0, 0, 0",
	UDim2.fromOffset(18, 55),
	UDim2.fromOffset(424, 25),
	11
)

CurrentPositionLabel.TextColor3 = Color3.fromRGB(150, 150, 150)

--==================================================
-- COORDINATE BOXES
--==================================================

local XBox = create("TextBox", {
	PlaceholderText = "X",
	Text = "",
	ClearTextOnFocus = false,
	BackgroundColor3 = Color3.fromRGB(25, 25, 25),
	BorderSizePixel = 0,
	Position = UDim2.fromOffset(18, 90),
	Size = UDim2.fromOffset(130, 38),
	TextColor3 = Color3.fromRGB(235, 235, 235),
	PlaceholderColor3 = Color3.fromRGB(110, 110, 110),
	TextSize = 13,
	Font = Enum.Font.GothamMedium
}, TeleportPanel)

corner(XBox, 7)
stroke(XBox)

local YBox = create("TextBox", {
	PlaceholderText = "Y",
	Text = "",
	ClearTextOnFocus = false,
	BackgroundColor3 = Color3.fromRGB(25, 25, 25),
	BorderSizePixel = 0,
	Position = UDim2.fromOffset(165, 90),
	Size = UDim2.fromOffset(130, 38),
	TextColor3 = Color3.fromRGB(235, 235, 235),
	PlaceholderColor3 = Color3.fromRGB(110, 110, 110),
	TextSize = 13,
	Font = Enum.Font.GothamMedium
}, TeleportPanel)

corner(YBox, 7)
stroke(YBox)

local ZBox = create("TextBox", {
	PlaceholderText = "Z",
	Text = "",
	ClearTextOnFocus = false,
	BackgroundColor3 = Color3.fromRGB(25, 25, 25),
	BorderSizePixel = 0,
	Position = UDim2.fromOffset(312, 90),
	Size = UDim2.fromOffset(130, 38),
	TextColor3 = Color3.fromRGB(235, 235, 235),
	PlaceholderColor3 = Color3.fromRGB(110, 110, 110),
	TextSize = 13,
	Font = Enum.Font.GothamMedium
}, TeleportPanel)

corner(ZBox, 7)
stroke(ZBox)

local CurrentPositionButton = makeButton(
	TeleportPanel,
	"USE CURRENT POSITION",
	UDim2.fromOffset(18, 140),
	UDim2.fromOffset(424, 38)
)

local TeleportCoordinatesButton = makeButton(
	TeleportPanel,
	"TELEPORT",
	UDim2.fromOffset(18, 185),
	UDim2.fromOffset(424, 42)
)

--==================================================
-- TELEPORT FUNCTIONS
--==================================================

local function updateCurrentPosition()

	local character, humanoid, head, root = getLocalCharacter()

	if root then

		local pos = root.Position

		CurrentPositionLabel.Text = string.format(
			"Current Position: %.1f, %.1f, %.1f",
			pos.X,
			pos.Y,
			pos.Z
		)
	end
end

local function useCurrentPosition()

	local character, humanoid, head, root = getLocalCharacter()

	if root then

		XBox.Text = string.format("%.2f", root.Position.X)
		YBox.Text = string.format("%.2f", root.Position.Y)
		ZBox.Text = string.format("%.2f", root.Position.Z)
	end
end

local function getCoordinates()

	local x = tonumber(XBox.Text)
	local y = tonumber(YBox.Text)
	local z = tonumber(ZBox.Text)

	if not x or not y or not z then
		return nil
	end

	return Vector3.new(x, y, z)
end

local function teleportToCoordinates(position)

	if not position then
		return
	end

	local character = LocalPlayer.Character

	if not character then
		return
	end

	local root = character:FindFirstChild("HumanoidRootPart")

	if not root then
		return
	end

	character:PivotTo(CFrame.new(position))
end

CurrentPositionButton.Activated:Connect(function()
	useCurrentPosition()
end)

TeleportCoordinatesButton.Activated:Connect(function()

	local position = getCoordinates()

	if position then
		teleportToCoordinates(position)
	end
end)

--==================================================
-- TELEPORT SAVE SLOTS
--==================================================

local SlotsLabel = makeLabel(
	TeleportPanel,
	"Saved Locations",
	UDim2.fromOffset(18, 242),
	UDim2.fromOffset(250, 25),
	14
)

local SlotsFrame = create("ScrollingFrame", {
	Position = UDim2.fromOffset(18, 272),
	Size = UDim2.fromOffset(424, 175),
	BackgroundTransparency = 1,
	BorderSizePixel = 0,
	ScrollBarThickness = 5,
	CanvasSize = UDim2.fromOffset(0, 0)
}, TeleportPanel)

local SlotsLayout = create("UIListLayout", {
	Padding = UDim.new(0, 5),
	SortOrder = Enum.SortOrder.LayoutOrder
}, SlotsFrame)

local function updateSlotButton(slotNumber, button)

	local data = TeleportSlots[slotNumber]

	if data then

		button.Text = string.format(
			"Slot %d  |  %.1f, %.1f, %.1f",
			slotNumber,
			data.X,
			data.Y,
			data.Z
		)

	else

		button.Text = "Slot " .. slotNumber .. "  |  EMPTY"
	end
end

for slotNumber = 1, 10 do

	local row = create("Frame", {
		BackgroundTransparency = 1,
		Size = UDim2.new(1, -5, 0, 35),
		LayoutOrder = slotNumber
	}, SlotsFrame)

	local slotButton = makeButton(
		row,
		"Slot " .. slotNumber .. " | EMPTY",
		UDim2.fromOffset(0, 0),
		UDim2.new(1, -82, 1, 0)
	)

	local saveButton = makeButton(
		row,
		"SAVE",
		UDim2.new(1, -75, 0, 0),
		UDim2.fromOffset(75, 35)
	)

	slotButton.Activated:Connect(function()

		local data = TeleportSlots[slotNumber]

		if data then

			XBox.Text = tostring(data.X)
			YBox.Text = tostring(data.Y)
			ZBox.Text = tostring(data.Z)
		end
	end)

	saveButton.Activated:Connect(function()

		local position = getCoordinates()

		if position then

			TeleportSlots[slotNumber] = position
			updateSlotButton(slotNumber, slotButton)
		end
	end)
end

SlotsFrame.CanvasSize = UDim2.fromOffset(0, 400)

local TeleportBack = makeButton(
	TeleportPanel,
	"BACK",
	UDim2.fromOffset(18, 485),
	UDim2.fromOffset(424, 42)
)

--==================================================
-- PANEL SWITCHING
--==================================================

local function openSettings()

	SettingsOpen = true
	TeleportOpen = false

	MainContent.Visible = false
	SettingsPanel.Visible = true
	TeleportPanel.Visible = false
end

local function openTeleporter()

	TeleportOpen = true
	SettingsOpen = false

	MainContent.Visible = false
	SettingsPanel.Visible = false
	TeleportPanel.Visible = true

	updateCurrentPosition()
end

local function closePanels()

	SettingsOpen = false
	TeleportOpen = false

	MainContent.Visible = true
	SettingsPanel.Visible = false
	TeleportPanel.Visible = false
end

SettingsButton.Activated:Connect(openSettings)

SettingsBack.Activated:Connect(closePanels)

TeleportButton.Activated:Connect(openTeleporter)

TeleportBack.Activated:Connect(closePanels)

--==================================================
-- HOTKEY SETTINGS
--==================================================

HotkeyButton.Activated:Connect(function()

	WaitingForKey = true
	HotkeyButton.Text = "PRESS A KEY..."
end)

--==================================================
-- AIM TARGETING
--==================================================

local function getClosestTarget()

	local camera = workspace.CurrentCamera

	if not camera then
		return nil
	end

	local mousePosition = UserInputService:GetMouseLocation()

	local viewportSize = camera.ViewportSize
	local screenCenter = Vector2.new(
		viewportSize.X / 2,
		viewportSize.Y / 2
	)

	local fovPixels = math.tan(math.rad(FOV / 2))
	local referenceDistance = math.max(viewportSize.X, viewportSize.Y)

	local radius = math.clamp(
		fovPixels * referenceDistance * 0.45,
		40,
		500
	)

	local closestPlayer = nil
	local closestDistance = math.huge

	for _, targetPlayer in ipairs(Players:GetPlayers()) do

		if targetPlayer ~= LocalPlayer then

			local character, humanoid, head = getCharacter(targetPlayer)

			if character and humanoid and head then

				local screenPosition, onScreen =
					camera:WorldToViewportPoint(head.Position)

				if onScreen and screenPosition.Z > 0 then

					local targetScreenPosition = Vector2.new(
						screenPosition.X,
						screenPosition.Y
					)

					local distanceFromMouse =
						(targetScreenPosition - mousePosition).Magnitude

					local distanceFromCenter =
						(targetScreenPosition - screenCenter).Magnitude

					if distanceFromCenter <= radius
						and distanceFromMouse < closestDistance then

						closestDistance = distanceFromMouse
						closestPlayer = targetPlayer
					end
				end
			end
		end
	end

	return closestPlayer
end

local function isValidTarget(target)

	if not target then
		return false
	end

	local character, humanoid, head, root = getCharacter(target)

	return character ~= nil
		and humanoid ~= nil
		and head ~= nil
		and root ~= nil
end

local function setTarget(target)

	CurrentTarget = target

	if target then
		LockedLabel.Text = "Locked Onto: " .. target.Name
	else
		LockedLabel.Text = "Locked Onto: None"
	end
end

local function acquireTarget()

	local target = getClosestTarget()

	if target then
		setTarget(target)
	end
end

local function toggleAimLock()

	AimLockEnabled = not AimLockEnabled

	updateAimButton()

	if AimLockEnabled then

		acquireTarget()

	else

		setTarget(nil)
	end
end

AimButton.Activated:Connect(toggleAimLock)

--==================================================
-- TARGET SWITCHING
--==================================================

local function checkForTargetSwitch()

	if not AimLockEnabled then
		return
	end

	local camera = workspace.CurrentCamera

	if not camera then
		return
	end

	local mousePosition = UserInputService:GetMouseLocation()

	local bestTarget = nil
	local bestDistance = math.huge

	for _, targetPlayer in ipairs(Players:GetPlayers()) do

		if targetPlayer ~= LocalPlayer then

			local character, humanoid, head =
				getCharacter(targetPlayer)

			if character and humanoid and head then

				local screenPosition, onScreen =
					camera:WorldToViewportPoint(head.Position)

				if onScreen and screenPosition.Z > 0 then

					local targetPosition = Vector2.new(
						screenPosition.X,
						screenPosition.Y
					)

					local distance =
						(targetPosition - mousePosition).Magnitude

					if distance < bestDistance then

						bestDistance = distance
						bestTarget = targetPlayer
					end
				end
			end
		end
	end

	if bestTarget and bestTarget ~= CurrentTarget then

		local currentDistance = math.huge

		if isValidTarget(CurrentTarget) then

			local _, _, currentHead =
				getCharacter(CurrentTarget)

			local screenPosition, onScreen =
				camera:WorldToViewportPoint(currentHead.Position)

			if onScreen and screenPosition.Z > 0 then

				currentDistance = (
					Vector2.new(
						screenPosition.X,
						screenPosition.Y
					) - mousePosition
				).Magnitude
			end
		end

		-- Require the new target to be meaningfully closer.
		if bestDistance + 25 < currentDistance then
			setTarget(bestTarget)
		end
	end
end

--==================================================
-- ESP
--==================================================

local function removeESP(targetPlayer)

	local gui = ESPObjects[targetPlayer]

	if gui then
		gui:Destroy()
		ESPObjects[targetPlayer] = nil
	end
end

local function createESP(targetPlayer)

	if targetPlayer == LocalPlayer then
		return
	end

	removeESP(targetPlayer)

	local character = targetPlayer.Character

	if not character then
		return
	end

	local head = character:FindFirstChild("Head")

	if not head then
		return
	end

	local billboard = Instance.new("BillboardGui")

	billboard.Name = "CamAimESP"
	billboard.Adornee = head
	billboard.AlwaysOnTop = true
	billboard.MaxDistance = 10000
	billboard.Size = UDim2.fromOffset(240, 70)
	billboard.StudsOffset = Vector3.new(0, 3, 0)
	billboard.Parent = head

	local label = Instance.new("TextLabel")

	label.Name = "Info"
	label.BackgroundTransparency = 1
	label.Size = UDim2.fromScale(1, 1)
	label.TextColor3 = Color3.fromRGB(255, 255, 255)
	label.TextStrokeTransparency = 0.3
	label.TextSize = 12
	label.Font = Enum.Font.GothamBold
	label.TextWrapped = true
	label.Parent = billboard

	ESPObjects[targetPlayer] = billboard

	task.spawn(function()

		while billboard.Parent and ESPObjects[targetPlayer] == billboard do

			local character2, humanoid2, head2, root2 =
				getCharacter(targetPlayer)

			if root2 then

				local p = root2.Position

				label.Text = string.format(
					"%s\nUserId: %d\nXYZ: %.1f, %.1f, %.1f",
					targetPlayer.Name,
					targetPlayer.UserId,
					p.X,
					p.Y,
					p.Z
				)

			else

				label.Text = targetPlayer.Name
			end

			task.wait(0.05)
		end
	end)
end

local function refreshESP()

	for _, targetPlayer in ipairs(Players:GetPlayers()) do

		if targetPlayer ~= LocalPlayer then

			if ESPEnabled then
				createESP(targetPlayer)
			else
				removeESP(targetPlayer)
			end
		end
	end
end

ESPButton.Activated:Connect(function()

	ESPEnabled = not ESPEnabled

	if ESPEnabled then
		ESPButton.Text = "ESP: ON"
	else
		ESPButton.Text = "ESP: OFF"
	end

	refreshESP()
end)

--==================================================
-- PLAYER EVENTS
--==================================================

Players.PlayerAdded:Connect(function(targetPlayer)

	targetPlayer.CharacterAdded:Connect(function()
		task.wait(0.5)

		if ESPEnabled then
			createESP(targetPlayer)
		end

		refreshPlayerDropdown()
	end)

	refreshPlayerDropdown()
end)

Players.PlayerRemoving:Connect(function(targetPlayer)

	removeESP(targetPlayer)

	if CurrentTarget == targetPlayer then
		setTarget(nil)
	end

	if SelectedPlayer == targetPlayer then
		SelectedPlayer = nil
		PlayerDropdown.Text = "Select Test Player"
	end

	task.defer(refreshPlayerDropdown)
end)

for _, targetPlayer in ipairs(Players:GetPlayers()) do

	if targetPlayer ~= LocalPlayer then

		targetPlayer.CharacterAdded:Connect(function()

			task.wait(0.5)

			if ESPEnabled then
				createESP(targetPlayer)
			end
		end)
	end
end

--==================================================
-- CHARACTER RESPAWN
--==================================================

LocalPlayer.CharacterAdded:Connect(function(character)

	local humanoid = character:WaitForChild("Humanoid", 10)

	if humanoid then
		humanoid.WalkSpeed = WalkSpeed
	end

	setTarget(nil)

	task.wait(0.5)

	updateCurrentPosition()
end)

--==================================================
-- INPUT
--==================================================

UserInputService.InputBegan:Connect(function(input, gameProcessed)

	if gameProcessed then
		return
	end

	-- Changing aim hotkey
	if WaitingForKey then

		if input.UserInputType == Enum.UserInputType.Keyboard then

			ToggleKey = input.KeyCode
			WaitingForKey = false

			HotkeyButton.Text = ToggleKey.Name
		end

		return
	end

	-- Aim lock hotkey
	if input.KeyCode == ToggleKey then

		toggleAimLock()
		return
	end

	-- Hide/show menu
	if input.KeyCode == MenuKey then

		MenuVisible = not MenuVisible
		MainFrame.Visible = MenuVisible

		-- Hide every panel when menu is hidden
		if not MenuVisible then
			DropdownOpen = false
			PlayerListFrame.Visible = false
		end
	end
end)

--==================================================
-- SELECTED PLAYER TELEPORT
--==================================================

local function teleportToSelectedPlayer()

	if not SelectedPlayer then
		return
	end

	local character = SelectedPlayer.Character

	if not character then
		return
	end

	local root = character:FindFirstChild("HumanoidRootPart")

	if not root then
		return
	end

	local localCharacter = LocalPlayer.Character

	if not localCharacter then
		return
	end

	-- Slightly above/behind the test player.
	local destination =
		root.Position + Vector3.new(0, 3, -5)

	localCharacter:PivotTo(
		CFrame.new(destination, root.Position)
	)
end

local SelectedTeleportButton = makeButton(
	MainContent,
	"TELEPORT TO SELECTED PLAYER",
	UDim2.fromOffset(15, 478),
	UDim2.fromOffset(430, 40)
)

SelectedTeleportButton.Activated:Connect(teleportToSelectedPlayer)

--==================================================
-- RENDER LOOP
--==================================================

RunService.RenderStepped:Connect(function()

	local camera = workspace.CurrentCamera

	if not camera then
		return
	end

	-- Camera FOV
	camera.FieldOfView = FOV

	-- Keep WalkSpeed active
	local localCharacter = LocalPlayer.Character

	if localCharacter then

		local humanoid =
			localCharacter:FindFirstChildOfClass("Humanoid")

		if humanoid and humanoid.Health > 0 then
			humanoid.WalkSpeed = WalkSpeed
		end
	end

	-- Aim lock
	if AimLockEnabled then

		if not isValidTarget(CurrentTarget) then

			acquireTarget()

		else

			checkForTargetSwitch()

			local character, humanoid, head, root =
				getCharacter(CurrentTarget)

			if head then

				-- Look directly at the target.
				-- Higher smoothness = faster tracking.
				local desiredCFrame =
					CFrame.lookAt(
						camera.CFrame.Position,
						head.Position
					)

				local trackingAmount =
					math.clamp(
						Smoothness * 2.5,
						0.12,
						1
					)

				camera.CFrame =
					camera.CFrame:Lerp(
						desiredCFrame,
						trackingAmount
					)
			end
		end
	end
end)

--==================================================
-- POSITION UPDATE
--==================================================

task.spawn(function()

	while ScreenGui.Parent do

		if TeleportOpen then
			updateCurrentPosition()
		end

		task.wait(0.1)
	end
end)

--==================================================
-- INITIALIZE
--==================================================

local function initialize()

	local camera = workspace.CurrentCamera

	if camera then
		camera.FieldOfView = FOV
	end

	local character = LocalPlayer.Character

	if character then

		local humanoid =
			character:FindFirstChildOfClass("Humanoid")

		if humanoid then
			humanoid.WalkSpeed = WalkSpeed
		end
	end

	refreshPlayerDropdown()
	refreshESP()
	updateAimButton()

	print("====================================")
	print("[CAM AIM TEST MENU] Loaded successfully")
	print("[CAM AIM TEST MENU] Aim Key:", ToggleKey.Name)
	print("[CAM AIM TEST MENU] Menu Key:", MenuKey.Name)
	print("[CAM AIM TEST MENU] ESP:", ESPEnabled)
	print("[CAM AIM TEST MENU] Hitboxes: REMOVED")
	print("[CAM AIM TEST MENU] Teleport Slots: 1-10")
	print("====================================")
end

initialize()
