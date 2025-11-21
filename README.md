-- ============================================
-- VSYNC PANEL - Complete Final Edition (Discord card organized)
-- Red, Black & White Theme (Safe: crash actions simulated)
-- ============================================

local TweenService = game:GetService("TweenService")
local UserInputService = game:GetService("UserInputService")
local RunService = game:GetService("RunService")
local Players = game:GetService("Players")
local ReplicatedStorage = game:GetService("ReplicatedStorage")

local player = Players.LocalPlayer
local ScreenGui = Instance.new("ScreenGui")
ScreenGui.Name = "VsyncPanel"
ScreenGui.ResetOnSpawn = false
ScreenGui.ZIndexBehavior = Enum.ZIndexBehavior.Sibling

if game:GetService("CoreGui"):FindFirstChild("VsyncPanel") then
    game:GetService("CoreGui"):FindFirstChild("VsyncPanel"):Destroy()
end

ScreenGui.Parent = game:GetService("CoreGui")

local Colors = {
    Background = Color3.fromRGB(5, 5, 5),
    Panel = Color3.fromRGB(10, 10, 10),
    Card = Color3.fromRGB(15, 15, 15),
    Primary = Color3.fromRGB(220, 30, 30),
    PrimaryDark = Color3.fromRGB(180, 20, 20),
    PrimaryLight = Color3.fromRGB(255, 80, 80),
    Success = Color3.fromRGB(50, 220, 50),
    TextPrimary = Color3.fromRGB(255, 255, 255),
    TextSecondary = Color3.fromRGB(200, 200, 200),
    TextMuted = Color3.fromRGB(120, 120, 120),
    Border = Color3.fromRGB(80, 20, 20),
}

local State = {
    Minimized = false,
    SavedPositions = {nil, nil, nil, nil},
    CurrentSaveSlot = 1,
    Fly = {Enabled = false, Speed = 50, Connection = nil},
    Noclip = {Enabled = false},
    Speed = {Enabled = false, Speed = 50},
    XRay = {Enabled = false},
    JumpTame = {Enabled = false},
    Spider = {Enabled = false, Connection = nil},
    FOV = {Enabled = false, Value = 70, Connection = nil},
    AutoLaser = {Enabled = false},
    Platform = {Enabled = false},
    LockTimeEsp = {Enabled = false, Connection = nil},
    PlayerEsp = {Enabled = false, Connection = nil},
}

local laserConnection = nil
local lastShootTime = 0
local SHOOT_COOLDOWN = 0.8
local LASER_DETECTION_RANGE = 40
local platformModel = nil
local platformConnection = nil
local xrayOriginalTransparencies = {}
local jumpTameConnection = nil
local defaultFOV = 70

-- UTIL: popup
local function CreatePopup(title, message)
    local PopupGui = Instance.new("Frame")
    PopupGui.Size = UDim2.new(0, 280, 0, 95)
    PopupGui.Position = UDim2.new(1, 20, 1, -120)
    PopupGui.BackgroundColor3 = Color3.fromRGB(20, 5, 5)
    PopupGui.BorderSizePixel = 0
    PopupGui.Parent = ScreenGui

    local Corner = Instance.new("UICorner")
    Corner.CornerRadius = UDim.new(0, 14)
    Corner.Parent = PopupGui

    local Stroke = Instance.new("UIStroke")
    Stroke.Color = Colors.Primary
    Stroke.Thickness = 2
    Stroke.Parent = PopupGui

    local TitleLabel = Instance.new("TextLabel")
    TitleLabel.Size = UDim2.new(1, -15, 0, 25)
    TitleLabel.Position = UDim2.new(0, 10, 0, 6)
    TitleLabel.BackgroundTransparency = 1
    TitleLabel.Text = title
    TitleLabel.Font = Enum.Font.GothamBlack
    TitleLabel.TextSize = 12
    TitleLabel.TextColor3 = Colors.Primary
    TitleLabel.TextXAlignment = Enum.TextXAlignment.Left
    TitleLabel.Parent = PopupGui

    local MessageLabel = Instance.new("TextLabel")
    MessageLabel.Size = UDim2.new(1, -15, 0, 50)
    MessageLabel.Position = UDim2.new(0, 10, 0, 35)
    MessageLabel.BackgroundTransparency = 1
    MessageLabel.Text = message
    MessageLabel.Font = Enum.Font.Gotham
    MessageLabel.TextSize = 10
    MessageLabel.TextColor3 = Colors.TextSecondary
    MessageLabel.TextWrapped = true
    MessageLabel.TextXAlignment = Enum.TextXAlignment.Left
    MessageLabel.Parent = PopupGui

    TweenService:Create(PopupGui, TweenInfo.new(0.3, Enum.EasingStyle.Back), {
        Position = UDim2.new(1, -300, 1, -120)
    }):Play()

    delay(3, function()
        TweenService:Create(PopupGui, TweenInfo.new(0.3, Enum.EasingStyle.Back), {
            Position = UDim2.new(1, 20, 1, -120)
        }):Play()
        wait(0.3)
        PopupGui:Destroy()
    end)
end

local function AddGradient(frame, color1, color2, rotation)
    if frame:FindFirstChild("UIGradient") then
        frame:FindFirstChild("UIGradient"):Destroy()
    end
    local gradient = Instance.new("UIGradient")
    gradient.Color = ColorSequence.new{
        ColorSequenceKeypoint.new(0, color1),
        ColorSequenceKeypoint.new(1, color2)
    }
    gradient.Rotation = rotation or 90
    gradient.Parent = frame
end

-- LASER helpers
local function getClosestPlayerForLaser()
    if not player.Character or not player.Character:FindFirstChild("HumanoidRootPart") then
        return nil
    end
    local myPosition = player.Character.HumanoidRootPart.Position
    local closestPlayer = nil
    local shortestDistance = math.huge
    for _, otherPlayer in pairs(Players:GetPlayers()) do
        if otherPlayer ~= player and otherPlayer.Character and otherPlayer.Character:FindFirstChild("HumanoidRootPart") then
            local distance = (otherPlayer.Character.HumanoidRootPart.Position - myPosition).Magnitude
            if distance < shortestDistance then
                shortestDistance = distance
                closestPlayer = otherPlayer
            end
        end
    end
    return closestPlayer, shortestDistance
end

local function equipLaserCape()
    local backpack = player:FindFirstChild("Backpack")
    local laserCape = nil
    if backpack then laserCape = backpack:FindFirstChild("Laser Cape") end
    if not laserCape and player.Character then laserCape = player.Character:FindFirstChild("Laser Cape") end
    if laserCape and backpack and laserCape.Parent == backpack and player.Character and player.Character:FindFirstChild("Humanoid") then
        pcall(function() player.Character.Humanoid:EquipTool(laserCape) end)
        return true
    elseif laserCape and laserCape.Parent == player.Character then
        return true
    end
    return false
end

local function autoLaserLoop()
    if not State.AutoLaser.Enabled then return end
    if not player.Character or not player.Character:FindFirstChild("HumanoidRootPart") then return end
    if not player.Character:FindFirstChild("Laser Cape") then equipLaserCape() return end
    local currentTime = tick()
    if currentTime - lastShootTime < SHOOT_COOLDOWN then return end
    local closestPlayer, distance = getClosestPlayerForLaser()
    if closestPlayer and distance <= LASER_DETECTION_RANGE then
        lastShootTime = currentTime
        local targetPosition = closestPlayer.Character.HumanoidRootPart.Position
        local args = {[1] = targetPosition, [2] = closestPlayer.Character.HumanoidRootPart}
        pcall(function()
            local ok, net = pcall(function()
                return ReplicatedStorage:WaitForChild("Packages", 1):WaitForChild("Net", 1)
            end)
            if ok and net and net:FindFirstChild("RE/UseItem") then
                net["RE/UseItem"]:FireServer(unpack(args))
            end
        end)
    end
end

local function startAutoLaser()
    State.AutoLaser.Enabled = true
    if laserConnection then laserConnection:Disconnect() end
    laserConnection = RunService.Heartbeat:Connect(autoLaserLoop)
    CreatePopup("Auto Laser Cape", "Ativado!")
end

local function stopAutoLaser()
    State.AutoLaser.Enabled = false
    if laserConnection then pcall(function() laserConnection:Disconnect() end) laserConnection = nil end
    CreatePopup("Auto Laser Cape", "Desativado")
end

-- Platform
local function updatePlatformPosition()
    if not State.Platform.Enabled or not platformModel or not platformModel.Parent then return end
    local rootPart = player.Character and player.Character:FindFirstChild("HumanoidRootPart")
    if not rootPart then return end
    local currentPos = rootPart.Position
    local targetPos = Vector3.new(currentPos.X, currentPos.Y - 3, currentPos.Z)
    if platformModel.Position.Y < currentPos.Y - 10 then
        targetPos = Vector3.new(currentPos.X, currentPos.Y + 5, currentPos.Z)
    end
    local currentPlatformPos = platformModel.Position
    local lerpedPos = currentPlatformPos:lerp(targetPos, 0.1)
    platformModel.Position = lerpedPos
end

local function spawnPlatform()
    if platformModel then platformModel:Destroy() end
    local rootPart = player.Character and player.Character:FindFirstChild("HumanoidRootPart")
    if not rootPart then return end
    platformModel = Instance.new("Part")
    platformModel.Size = Vector3.new(10, 1, 10)
    platformModel.Position = Vector3.new(rootPart.Position.X, rootPart.Position.Y - 3, rootPart.Position.Z)
    platformModel.Anchored = true
    platformModel.Color = Colors.Primary
    platformModel.Material = Enum.Material.Neon
    platformModel.Name = "ScoutPlatform"
    platformModel.Parent = workspace
    if platformConnection then platformConnection:Disconnect() end
    platformConnection = RunService.Heartbeat:Connect(updatePlatformPosition)
end

local function stopPlatform()
    State.Platform.Enabled = false
    if platformConnection then platformConnection:Disconnect() platformConnection = nil end
    if platformModel then platformModel:Destroy() platformModel = nil end
end

-- FOV
local function EnableFOV()
    State.FOV.Enabled = true
    if State.FOV.Connection then State.FOV.Connection:Disconnect() end
    State.FOV.Connection = RunService.Heartbeat:Connect(function()
        if not State.FOV.Enabled then return end
        local camera = workspace.CurrentCamera
        if camera then camera.FieldOfView = State.FOV.Value end
    end)
    CreatePopup("FOV", "Ativado!")
end

local function DisableFOV()
    State.FOV.Enabled = false
    if State.FOV.Connection then State.FOV.Connection:Disconnect() State.FOV.Connection = nil end
    local camera = workspace.CurrentCamera
    if camera then camera.FieldOfView = defaultFOV end
    CreatePopup("FOV", "Desativado")
end

-- Spider / JumpTame
local function EnableSpider()
    State.Spider.Enabled = true
    if State.Spider.Connection then State.Spider.Connection:Disconnect() end
    State.Spider.Connection = RunService.Heartbeat:Connect(function()
        if not State.Spider.Enabled then return end
        local character = player.Character
        if not character then return end
        local rootPart = character:FindFirstChild("HumanoidRootPart")
        local humanoid = character:FindFirstChild("Humanoid")
        if not humanoid or not rootPart then return end
        local rayDirection = rootPart.CFrame.LookVector
        local rayOrigin = rootPart.Position
        local raycastParams = RaycastParams.new()
        raycastParams.FilterType = Enum.RaycastFilterType.Exclude
        raycastParams.FilterDescendantsInstances = {character}
        local rayResult = workspace:Raycast(rayOrigin, rayDirection * 10, raycastParams)
        if rayResult then
            if (UserInputService:IsKeyDown(Enum.KeyCode.W) or UserInputService:IsKeyDown(Enum.KeyCode.A) or
                UserInputService:IsKeyDown(Enum.KeyCode.S) or UserInputService:IsKeyDown(Enum.KeyCode.D)) then
                if UserInputService:IsKeyDown(Enum.KeyCode.Space) then
                    rootPart.AssemblyLinearVelocity = Vector3.new(rootPart.AssemblyLinearVelocity.X, 50, rootPart.AssemblyLinearVelocity.Z)
                    humanoid.PlatformStand = true
                else
                    humanoid.PlatformStand = false
                end
            else
                humanoid.PlatformStand = false
            end
        else
            humanoid.PlatformStand = false
        end
    end)
    CreatePopup("Spider", "Ativado!")
end

local function DisableSpider()
    State.Spider.Enabled = false
    if State.Spider.Connection then State.Spider.Connection:Disconnect() State.Spider.Connection = nil end
    local character = player.Character
    if character then
        local humanoid = character:FindFirstChild("Humanoid")
        if humanoid then humanoid.PlatformStand = false end
    end
    CreatePopup("Spider", "Desativado")
end

local function EnableJumpTame()
    State.JumpTame.Enabled = true
    if jumpTameConnection then jumpTameConnection:Disconnect() end
    jumpTameConnection = RunService.Heartbeat:Connect(function()
        if not State.JumpTame.Enabled then return end
        local character = player.Character
        if not character then return end
        local humanoid = character:FindFirstChild("Humanoid")
        local rootPart = character:FindFirstChild("HumanoidRootPart")
        if humanoid and rootPart then
            if UserInputService:IsKeyDown(Enum.KeyCode.Space) then
                rootPart.AssemblyLinearVelocity = Vector3.new(rootPart.AssemblyLinearVelocity.X, 100, rootPart.AssemblyLinearVelocity.Z)
            end
        end
    end)
    CreatePopup("Jump Tame", "Ativado!")
end

local function DisableJumpTame()
    State.JumpTame.Enabled = false
    if jumpTameConnection then jumpTameConnection:Disconnect() jumpTameConnection = nil end
    CreatePopup("Jump Tame", "Desativado")
end

-- X-Ray base
local function EnableXRayBase()
    State.XRay.Enabled = true
    xrayOriginalTransparencies = {}
    pcall(function()
        for _, plot in pairs(workspace:FindFirstChild("Plots") and workspace.Plots:GetChildren() or {}) do
            if plot.Name ~= nil and plot.Name ~= "" then
                local purchases = plot:FindFirstChild("Purchases")
                if purchases then
                    for _, part in pairs(purchases:GetDescendants()) do
                        if part:IsA("BasePart") then
                            if not xrayOriginalTransparencies[part] then
                                xrayOriginalTransparencies[part] = part.Transparency
                            end
                            part.Transparency = 1
                        end
                    end
                end
            end
        end
    end)
    CreatePopup("Raio X Base", "Ativado!")
end

local function DisableXRayBase()
    State.XRay.Enabled = false
    for part, originalTransparency in pairs(xrayOriginalTransparencies) do
        if part and part.Parent then
            part.Transparency = originalTransparency
        end
    end
    xrayOriginalTransparencies = {}
    CreatePopup("Raio X Base", "Desativado")
end

-- Noclip
local function EnableNoclip()
    local character = player.Character
    if not character then return end
    for _, part in pairs(character:GetDescendants()) do
        if part:IsA("BasePart") then part.CanCollide = false end
    end
    CreatePopup("Noclip", "Ativado!")
end

local function DisableNoclip()
    local character = player.Character
    if not character then return end
    for _, part in pairs(character:GetDescendants()) do
        if part:IsA("BasePart") then part.CanCollide = true end
    end
    CreatePopup("Noclip", "Desativado")
end

-- ESP: create / remove BillboardGui per player reliably
local function CreatePlayerBillboard(otherPlayer)
    if not otherPlayer or not otherPlayer.Character then return end
    local root = otherPlayer.Character:FindFirstChild("HumanoidRootPart")
    if not root then return end
    if root:FindFirstChild("PlayerESP_BillboardGui") then return end

    local billboard = Instance.new("BillboardGui")
    billboard.Name = "PlayerESP_BillboardGui"
    billboard.Size = UDim2.new(0, 150, 0, 40)
    billboard.StudsOffset = Vector3.new(0, 2.5, 0)
    billboard.AlwaysOnTop = true
    billboard.MaxDistance = 1000
    billboard.Parent = root

    local label = Instance.new("TextLabel")
    label.Size = UDim2.new(1, 0, 1, 0)
    label.BackgroundTransparency = 1
    label.Text = otherPlayer.Name
    label.TextColor3 = Colors.Primary
    label.Font = Enum.Font.GothamBold
    label.TextScaled = true
    label.Parent = billboard
end

local function RemovePlayerBillboard(otherPlayer)
    if not otherPlayer or not otherPlayer.Character then return end
    local root = otherPlayer.Character:FindFirstChild("HumanoidRootPart")
    if root and root:FindFirstChild("PlayerESP_BillboardGui") then
        root.PlayerESP_BillboardGui:Destroy()
    end
end

local function EnablePlayerEsp()
    State.PlayerEsp.Enabled = true
    if State.PlayerEsp.Connection then State.PlayerEsp.Connection:Disconnect() end

    for _, otherPlayer in pairs(Players:GetPlayers()) do
        if otherPlayer ~= player then
            CreatePlayerBillboard(otherPlayer)
        end
    end

    State.PlayerEsp.Connection = RunService.Heartbeat:Connect(function()
        if not State.PlayerEsp.Enabled then return end
        for _, otherPlayer in pairs(Players:GetPlayers()) do
            if otherPlayer ~= player and otherPlayer.Character then
                CreatePlayerBillboard(otherPlayer)
            end
        end
    end)

    Players.PlayerRemoving:Connect(function(p)
        RemovePlayerBillboard(p)
    end)

    Players.PlayerAdded:Connect(function(p)
        p.CharacterAdded:Connect(function()
            if State.PlayerEsp.Enabled then
                CreatePlayerBillboard(p)
            end
        end)
    end)

    CreatePopup("Player ESP", "Ativado!")
end

local function DisablePlayerEsp()
    State.PlayerEsp.Enabled = false
    if State.PlayerEsp.Connection then State.PlayerEsp.Connection:Disconnect() State.PlayerEsp.Connection = nil end

    for _, otherPlayer in pairs(Players:GetPlayers()) do
        RemovePlayerBillboard(otherPlayer)
    end

    CreatePopup("Player ESP", "Desativado")
end

-- Lock time
local function EnableLockTimeEsp()
    if State.LockTimeEsp.Connection then State.LockTimeEsp.Connection:Disconnect() end
    State.LockTimeEsp.Enabled = true
    State.LockTimeEsp.Connection = RunService.Heartbeat:Connect(function()
        if not State.LockTimeEsp.Enabled then return end
        pcall(function()
            local timeValue = ReplicatedStorage:FindFirstChild("Time")
            if timeValue and (timeValue:IsA("NumberValue") or timeValue:IsA("IntValue")) then
                timeValue.Value = 12
            end
        end)
    end)
    CreatePopup("Lock Time", "Ativado!")
end

local function DisableLockTimeEsp()
    State.LockTimeEsp.Enabled = false
    if State.LockTimeEsp.Connection then State.LockTimeEsp.Connection:Disconnect() State.LockTimeEsp.Connection = nil end
    CreatePopup("Lock Time", "Desativado")
end

-- Safe "crash" simulations
local function SimulateCrashPlayer(targetPlayer)
    if not targetPlayer then
        CreatePopup("Crash (Simulado)", "Nenhum jogador selecionado.")
        return
    end
    CreatePopup("Crash (Simulado)", "Simulação: " .. targetPlayer.Name .. " recebeu um crash (SIMULADO).")
end

local function SimulateCrashServer()
    CreatePopup("Crash (Simulado)", "Simulação: crash no servidor (SIMULADO).")
end

-- UI helpers
local function CreateToggleButton(parent, x, y, width, height, text, callback)
    local isActive = false
    local Button = Instance.new("TextButton")
    Button.Size = UDim2.new(0, width, 0, height)
    Button.Position = UDim2.new(0, x, 0, y)
    Button.BackgroundColor3 = Color3.fromRGB(20, 20, 20)
    Button.BorderSizePixel = 0
    Button.Text = ""
    Button.AutoButtonColor = false
    Button.Parent = parent

    local Corner = Instance.new("UICorner")
    Corner.CornerRadius = UDim.new(0, 10)
    Corner.Parent = Button

    local Stroke = Instance.new("UIStroke")
    Stroke.Color = Colors.Border
    Stroke.Thickness = 1.5
    Stroke.Parent = Button

    local Label = Instance.new("TextLabel")
    Label.Size = UDim2.new(1, -20, 1, 0)
    Label.Position = UDim2.new(0, 10, 0, 0)
    Label.BackgroundTransparency = 1
    Label.Text = text
    Label.Font = Enum.Font.GothamBold
    Label.TextSize = 10
    Label.TextColor3 = Colors.TextSecondary
    Label.TextXAlignment = Enum.TextXAlignment.Left
    Label.Parent = Button

    local ToggleBall = Instance.new("Frame")
    ToggleBall.Name = "ToggleBall"
    ToggleBall.Size = UDim2.new(0, 16, 0, 16)
    ToggleBall.Position = UDim2.new(1, -20, 0.5, -8)
    ToggleBall.BackgroundColor3 = Color3.fromRGB(80, 80, 80)
    ToggleBall.BorderSizePixel = 0
    ToggleBall.Parent = Button

    local BallCorner = Instance.new("UICorner")
    BallCorner.CornerRadius = UDim.new(0, 4)
    BallCorner.Parent = ToggleBall

    Button.MouseButton1Click:Connect(function()
        isActive = not isActive
        if isActive then
            TweenService:Create(Button, TweenInfo.new(0.25, Enum.EasingStyle.Quad), {BackgroundColor3 = Color3.fromRGB(35, 15, 15)}):Play()
            TweenService:Create(Stroke, TweenInfo.new(0.25), {Color = Colors.Primary, Thickness = 2}):Play()
            TweenService:Create(Label, TweenInfo.new(0.25), {TextColor3 = Colors.Primary}):Play()
            TweenService:Create(ToggleBall, TweenInfo.new(0.25, Enum.EasingStyle.Quad), {BackgroundColor3 = Colors.Success, Position = UDim2.new(1, -62, 0.5, -8)}):Play()
        else
            TweenService:Create(Button, TweenInfo.new(0.25, Enum.EasingStyle.Quad), {BackgroundColor3 = Color3.fromRGB(20, 20, 20)}):Play()
            TweenService:Create(Stroke, TweenInfo.new(0.25), {Color = Colors.Border, Thickness = 1.5}):Play()
            TweenService:Create(Label, TweenInfo.new(0.25), {TextColor3 = Colors.TextSecondary}):Play()
            TweenService:Create(ToggleBall, TweenInfo.new(0.25, Enum.EasingStyle.Quad), {BackgroundColor3 = Color3.fromRGB(80, 80, 80), Position = UDim2.new(1, -20, 0.5, -8)}):Play()
        end
        if callback then callback(isActive) end
    end)
    return Button
end

local function CreateSlider(parent, x, y, width, min, max, value, callback)
    local Container = Instance.new("Frame")
    Container.Size = UDim2.new(0, width, 0, 80)
    Container.Position = UDim2.new(0, x, 0, y)
    Container.BackgroundTransparency = 1
    Container.Parent = parent

    local Label = Instance.new("TextLabel")
    Label.Size = UDim2.new(1, 0, 0, 20)
    Label.BackgroundTransparency = 1
    Label.Text = tostring(value)
    Label.Font = Enum.Font.GothamBold
    Label.TextSize = 12
    Label.TextColor3 = Colors.Primary
    Label.TextXAlignment = Enum.TextXAlignment.Center
    Label.Parent = Container

    local SliderFrame = Instance.new("Frame")
    SliderFrame.Size = UDim2.new(1, 0, 0, 8)
    SliderFrame.Position = UDim2.new(0, 0, 0, 30)
    SliderFrame.BackgroundColor3 = Color3.fromRGB(30, 30, 30)
    SliderFrame.BorderSizePixel = 0
    SliderFrame.Parent = Container

    local SliderCorner = Instance.new("UICorner")
    SliderCorner.CornerRadius = UDim.new(1, 0)
    SliderCorner.Parent = SliderFrame

    local SliderStroke = Instance.new("UIStroke")
    SliderStroke.Color = Colors.Border
    SliderStroke.Thickness = 1
    SliderStroke.Parent = SliderFrame

    AddGradient(SliderFrame, Color3.fromRGB(30, 30, 30), Color3.fromRGB(25, 25, 25), 0)

    local percent = (value - min) / (max - min)

    local Fill = Instance.new("Frame")
    Fill.Size = UDim2.new(percent, 0, 1, 0)
    Fill.BackgroundColor3 = Colors.Primary
    Fill.BorderSizePixel = 0
    Fill.Parent = SliderFrame

    local FillCorner = Instance.new("UICorner")
    FillCorner.CornerRadius = UDim.new(1, 0)
    FillCorner.Parent = Fill

    AddGradient(Fill, Colors.Primary, Colors.PrimaryDark, 0)

    local Knob = Instance.new("Frame")
    Knob.Size = UDim2.new(0, 18, 0, 18)
    Knob.Position = UDim2.new(percent, -9, 0.5, -9)
    Knob.BackgroundColor3 = Colors.Primary
    Knob.BorderSizePixel = 0
    Knob.ZIndex = 2
    Knob.Parent = SliderFrame

    local KnobCorner = Instance.new("UICorner")
    KnobCorner.CornerRadius = UDim.new(0, 4)
    KnobCorner.Parent = Knob

    local KnobStroke = Instance.new("UIStroke")
    KnobStroke.Color = Colors.PrimaryLight
    KnobStroke.Thickness = 1.5
    KnobStroke.Parent = Knob

    local dragging = false

    SliderFrame.InputBegan:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1 then
            dragging = true
        end
    end)

    UserInputService.InputEnded:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1 then
            dragging = false
        end
    end)

    UserInputService.InputChanged:Connect(function(input)
        if dragging and input.UserInputType == Enum.UserInputType.MouseMovement then
            local mousePos = UserInputService:GetMouseLocation()
            local sliderPos = SliderFrame.AbsolutePosition
            local sliderSize = SliderFrame.AbsoluteSize

            if sliderSize.X > 0 then
                local newPercent = math.clamp((mousePos.X - sliderPos.X) / sliderSize.X, 0, 1)
                local newValue = math.floor(min + (max - min) * newPercent)
                Fill.Size = UDim2.new(newPercent, 0, 1, 0)
                Knob.Position = UDim2.new(newPercent, -9, 0.5, -9)
                Label.Text = tostring(newValue)
                if callback then callback(newValue) end
            end
        end
    end)

    return Container
end

local function CreateActionButton(parent, x, y, width, height, text, callback)
    local Button = Instance.new("TextButton")
    Button.Size = UDim2.new(0, width, 0, height)
    Button.Position = UDim2.new(0, x, 0, y)
    Button.BackgroundColor3 = Colors.Primary
    Button.BorderSizePixel = 0
    Button.Text = ""
    Button.AutoButtonColor = false
    Button.Parent = parent

    local Corner = Instance.new("UICorner")
    Corner.CornerRadius = UDim.new(0, 10)
    Corner.Parent = Button

    AddGradient(Button, Colors.Primary, Colors.PrimaryDark, 45)

    local Stroke = Instance.new("UIStroke")
    Stroke.Color = Colors.PrimaryLight
    Stroke.Thickness = 1.5
    Stroke.Parent = Button

    local Label = Instance.new("TextLabel")
    Label.Size = UDim2.new(1, 0, 1, 0)
    Label.BackgroundTransparency = 1
    Label.Text = text
    Label.Font = Enum.Font.GothamBold
    Label.TextSize = 9
    Label.TextColor3 = Colors.TextPrimary
    Label.Parent = Button

    Button.MouseEnter:Connect(function()
        TweenService:Create(Button, TweenInfo.new(0.2, Enum.EasingStyle.Quad), {BackgroundColor3 = Colors.PrimaryLight}):Play()
    end)

    Button.MouseLeave:Connect(function()
        TweenService:Create(Button, TweenInfo.new(0.2, Enum.EasingStyle.Quad), {BackgroundColor3 = Colors.Primary}):Play()
    end)

    if callback then
        Button.MouseButton1Click:Connect(function()
            callback()
        end)
    end

    return Button
end

local function CreateCard(parent, height, title, icon)
    local Card = Instance.new("Frame")
    Card.Name = title or "Card"
    Card.Size = UDim2.new(1, 0, 0, height)
    Card.BackgroundColor3 = Color3.fromRGB(18, 18, 18)
    Card.BorderSizePixel = 0
    Card.Parent = parent

    local CardCorner = Instance.new("UICorner")
    CardCorner.CornerRadius = UDim.new(0, 12)
    CardCorner.Parent = Card

    AddGradient(Card, Color3.fromRGB(18, 18, 18), Color3.fromRGB(20, 20, 20), 135)

    local CardStroke = Instance.new("UIStroke")
    CardStroke.Color = Colors.Border
    CardStroke.Thickness = 1.5
    CardStroke.Parent = Card

    local IconFrame = Instance.new("Frame")
    IconFrame.Size = UDim2.new(0, 28, 0, 28)
    IconFrame.Position = UDim2.new(0, 8, 0, 8)
    IconFrame.BackgroundColor3 = Colors.Primary
    IconFrame.BorderSizePixel = 0
    IconFrame.Parent = Card

    local IconCorner = Instance.new("UICorner")
    IconCorner.CornerRadius = UDim.new(0, 7)
    IconCorner.Parent = IconFrame

    AddGradient(IconFrame, Colors.Primary, Colors.PrimaryDark, 45)

    local IconText = Instance.new("TextLabel")
    IconText.Size = UDim2.new(1, 0, 1, 0)
    IconText.BackgroundTransparency = 1
    IconText.Text = icon or "V"
    IconText.Font = Enum.Font.GothamBold
    IconText.TextSize = 14
    IconText.TextColor3 = Colors.TextPrimary
    IconText.Parent = IconFrame

    local TitleLabel = Instance.new("TextLabel")
    TitleLabel.Size = UDim2.new(1, -44, 0, 28)
    TitleLabel.Position = UDim2.new(0, 44, 0, 8)
    TitleLabel.BackgroundTransparency = 1
    TitleLabel.Text = title
    TitleLabel.Font = Enum.Font.GothamBold
    TitleLabel.TextSize = 11
    TitleLabel.TextColor3 = Colors.TextPrimary
    TitleLabel.TextXAlignment = Enum.TextXAlignment.Left
    TitleLabel.TextYAlignment = Enum.TextYAlignment.Center
    TitleLabel.Parent = Card

    return Card
end

-- MAIN PANEL UI
local MainFrame = Instance.new("Frame")
MainFrame.Name = "MainFrame"
MainFrame.Size = UDim2.new(0, 320, 0, 420)
MainFrame.Position = UDim2.new(0.5, -160, 0.5, -210)
MainFrame.BackgroundTransparency = 1
MainFrame.Parent = ScreenGui

local BackPanel = Instance.new("Frame")
BackPanel.Size = UDim2.new(1, 0, 1, 0)
BackPanel.BackgroundColor3 = Colors.Panel
BackPanel.BorderSizePixel = 0
BackPanel.Parent = MainFrame
BackPanel.ZIndex = 1

local BackCorner = Instance.new("UICorner")
BackCorner.CornerRadius = UDim.new(0, 18)
BackCorner.Parent = BackPanel

AddGradient(BackPanel, Color3.fromRGB(10, 10, 10), Color3.fromRGB(15, 15, 15), 135)

local BackStroke = Instance.new("UIStroke")
BackStroke.Color = Colors.Primary
BackStroke.Thickness = 1.5
BackStroke.Parent = BackPanel

local Header = Instance.new("Frame")
Header.Size = UDim2.new(1, 0, 0, 62)
Header.BackgroundColor3 = Color3.fromRGB(25, 10, 10)
Header.BorderSizePixel = 0
Header.Parent = MainFrame
Header.ZIndex = 2

local HeaderCorner = Instance.new("UICorner")
HeaderCorner.CornerRadius = UDim.new(0, 18)
HeaderCorner.Parent = Header

AddGradient(Header, Color3.fromRGB(35, 10, 10), Color3.fromRGB(15, 10, 10), 90)

local LogoText = Instance.new("TextLabel")
LogoText.Size = UDim2.new(0, 200, 0, 50)
LogoText.Position = UDim2.new(0, 12, 0, 8)
LogoText.BackgroundTransparency = 1
LogoText.Text = "VSYNC"
LogoText.Font = Enum.Font.GothamBlack
LogoText.TextSize = 40
LogoText.TextColor3 = Colors.Primary
LogoText.TextXAlignment = Enum.TextXAlignment.Left
LogoText.TextYAlignment = Enum.TextYAlignment.Center
LogoText.Parent = Header

AddGradient(LogoText, Color3.fromRGB(220, 30, 30), Color3.fromRGB(0, 0, 0), 0)

-- NOTE: Discord button removed from header per request

-- Header controls: minimize & close
local MinimizeButton = Instance.new("TextButton")
MinimizeButton.Size = UDim2.new(0, 34, 0, 34)
MinimizeButton.Position = UDim2.new(1, -74, 0, 14)
MinimizeButton.BackgroundColor3 = Color3.fromRGB(30, 30, 30)
MinimizeButton.BorderSizePixel = 0
MinimizeButton.Text = "━"
MinimizeButton.Font = Enum.Font.GothamBold
MinimizeButton.TextSize = 16
MinimizeButton.TextColor3 = Color3.fromRGB(255, 255, 255)
MinimizeButton.AutoButtonColor = false
MinimizeButton.Parent = Header
MinimizeButton.ZIndex = 3

local MinCorner = Instance.new("UICorner")
MinCorner.CornerRadius = UDim.new(0, 9)
MinCorner.Parent = MinimizeButton

local CloseButton = Instance.new("TextButton")
CloseButton.Size = UDim2.new(0, 34, 0, 34)
CloseButton.Position = UDim2.new(1, -36, 0, 14)
CloseButton.BackgroundColor3 = Color3.fromRGB(30, 30, 30)
CloseButton.BorderSizePixel = 0
CloseButton.Text = "x"
CloseButton.Font = Enum.Font.GothamBlack
CloseButton.TextSize = 18
CloseButton.TextColor3 = Color3.fromRGB(255, 255, 255)
CloseButton.AutoButtonColor = false
CloseButton.Parent = Header
CloseButton.ZIndex = 3

local CloseCorner = Instance.new("UICorner")
CloseCorner.CornerRadius = UDim.new(0, 9)
CloseCorner.Parent = CloseButton

-- NAV FRAME & CONTENT
local NavFrame = Instance.new("Frame")
NavFrame.Size = UDim2.new(1, -20, 0, 38)
NavFrame.Position = UDim2.new(0, 10, 0, 72)
NavFrame.BackgroundColor3 = Color3.fromRGB(22, 22, 22)
NavFrame.BorderSizePixel = 0
NavFrame.Parent = MainFrame
NavFrame.ZIndex = 2

local NavCorner = Instance.new("UICorner")
NavCorner.CornerRadius = UDim.new(0, 10)
NavCorner.Parent = NavFrame

AddGradient(NavFrame, Color3.fromRGB(22, 22, 22), Color3.fromRGB(18, 18, 18), 90)

local NavStroke = Instance.new("UIStroke")
NavStroke.Color = Colors.Border
NavStroke.Thickness = 1.5
NavStroke.Parent = NavFrame

local ContentArea = Instance.new("ScrollingFrame")
ContentArea.Name = "ContentArea"
ContentArea.Size = UDim2.new(1, -16, 1, -126)
ContentArea.Position = UDim2.new(0, 8, 0, 118)
ContentArea.BackgroundTransparency = 1
ContentArea.BorderSizePixel = 0
ContentArea.ScrollBarThickness = 3
ContentArea.ScrollBarImageColor3 = Colors.Primary
ContentArea.ScrollBarImageTransparency = 0.5
ContentArea.CanvasSize = UDim2.new(0, 0, 0, 0)
ContentArea.AutomaticCanvasSize = Enum.AutomaticSize.Y
ContentArea.ScrollingDirection = Enum.ScrollingDirection.Y
ContentArea.Parent = MainFrame
ContentArea.ZIndex = 2

local Pages = {}
local tabButtons = {}
local tabs = {"Dashboard", "Vsync", "Movement", "Visual", "Crash"}
local tabWidth = 56

for i, tab in ipairs(tabs) do
    local TabButton = Instance.new("TextButton")
    TabButton.Name = tab
    TabButton.Size = UDim2.new(0, tabWidth, 0, 26)
    TabButton.Position = UDim2.new(0, 3 + (i-1) * (tabWidth + 2), 0, 6)
    TabButton.BackgroundColor3 = tab == "Dashboard" and Colors.Primary or Color3.fromRGB(22, 22, 22)
    TabButton.BorderSizePixel = 0
    TabButton.Text = tab
    TabButton.Font = Enum.Font.GothamBold
    TabButton.TextSize = 6
    TabButton.TextColor3 = tab == "Dashboard" and Colors.TextPrimary or Colors.TextSecondary
    TabButton.AutoButtonColor = false
    TabButton.Parent = NavFrame
    TabButton.ZIndex = 3

    local TabCorner = Instance.new("UICorner")
    TabCorner.CornerRadius = UDim.new(0, 8)
    TabCorner.Parent = TabButton

    if tab == "Dashboard" then
        AddGradient(TabButton, Colors.Primary, Colors.PrimaryDark, 45)
    end

    tabButtons[tab] = TabButton

    Pages[tab] = Instance.new("Frame")
    Pages[tab].Name = tab
    Pages[tab].Size = UDim2.new(1, 0, 0, 0)
    Pages[tab].BackgroundTransparency = 1
    Pages[tab].Visible = (tab == "Dashboard")
    Pages[tab].AutomaticSize = Enum.AutomaticSize.Y
    Pages[tab].Parent = ContentArea

    local UIList = Instance.new("UIListLayout")
    UIList.Padding = UDim.new(0, 8)
    UIList.SortOrder = Enum.SortOrder.LayoutOrder
    UIList.Parent = Pages[tab]
end

for tabName, button in pairs(tabButtons) do
    button.MouseButton1Click:Connect(function()
        for name, btn in pairs(tabButtons) do
            local isActive = (name == tabName)
            if isActive then
                TweenService:Create(btn, TweenInfo.new(0.25, Enum.EasingStyle.Quad), {
                    BackgroundColor3 = Colors.Primary,
                    TextColor3 = Colors.TextPrimary
                }):Play()
                if not btn:FindFirstChild("UIGradient") then
                    AddGradient(btn, Colors.Primary, Colors.PrimaryDark, 45)
                end
            else
                TweenService:Create(btn, TweenInfo.new(0.25, Enum.EasingStyle.Quad), {
                    BackgroundColor3 = Color3.fromRGB(22, 22, 22),
                    TextColor3 = Colors.TextSecondary
                }):Play()
                if btn:FindFirstChild("UIGradient") then
                    btn.UIGradient:Destroy()
                end
            end
        end

        for pageName, page in pairs(Pages) do
            page.Visible = (pageName == tabName)
        end
    end)
end

-- DASHBOARD content (includes organized Discord card)
local TeleportCard = CreateCard(Pages.Dashboard, 245, "Advanced Teleport", "T")
TeleportCard.LayoutOrder = 1

-- Slot area (existing)
local SlotFrame = Instance.new("Frame")
SlotFrame.Size = UDim2.new(1, -16, 0, 50)
SlotFrame.Position = UDim2.new(0, 8, 0, 44)
SlotFrame.BackgroundTransparency = 1
SlotFrame.Parent = TeleportCard

local slotButtons = {}
for i = 1, 4 do
    local SlotButton = Instance.new("TextButton")
    SlotButton.Size = UDim2.new(0, 70, 0, 50)
    SlotButton.Position = UDim2.new(0, (i-1) * 72, 0, 0)
    SlotButton.BackgroundColor3 = i == 1 and Colors.Primary or Color3.fromRGB(22, 22, 22)
    SlotButton.BorderSizePixel = 0
    SlotButton.Text = ""
    SlotButton.AutoButtonColor = false
    SlotButton.Parent = SlotFrame

    local SlotCorner = Instance.new("UICorner")
    SlotCorner.CornerRadius = UDim.new(0, 10)
    SlotCorner.Parent = SlotButton

    local SlotStroke = Instance.new("UIStroke")
    SlotStroke.Color = i == 1 and Colors.PrimaryLight or Colors.Border
    SlotStroke.Thickness = i == 1 and 2 or 1.5
    SlotStroke.Parent = SlotButton

    if i == 1 then AddGradient(SlotButton, Colors.Primary, Colors.PrimaryDark, 45) end

    local SlotLabel = Instance.new("TextLabel")
    SlotLabel.Size = UDim2.new(1, 0, 0, 20)
    SlotLabel.Position = UDim2.new(0, 0, 0, 10)
    SlotLabel.BackgroundTransparency = 1
    SlotLabel.Text = "Slot " .. i
    SlotLabel.Font = Enum.Font.GothamBold
    SlotLabel.TextSize = 11
    SlotLabel.TextColor3 = i == 1 and Colors.TextPrimary or Colors.TextSecondary
    SlotLabel.Parent = SlotButton

    local StatusDot = Instance.new("Frame")
    StatusDot.Size = UDim2.new(0, 10, 0, 10)
    StatusDot.Position = UDim2.new(0.5, -5, 0, 32)
    StatusDot.BackgroundColor3 = i == 1 and Colors.Success or Color3.fromRGB(80, 80, 80)
    StatusDot.BorderSizePixel = 0
    StatusDot.Parent = SlotButton

    local DotCorner = Instance.new("UICorner")
    DotCorner.CornerRadius = UDim.new(1, 0)
    DotCorner.Parent = StatusDot

    slotButtons[i] = {Button = SlotButton, Stroke = SlotStroke, Label = SlotLabel, Dot = StatusDot}

    SlotButton.MouseButton1Click:Connect(function()
        State.CurrentSaveSlot = i
        for j = 1, 4 do
            local btn = slotButtons[j]
            if j == i then
                TweenService:Create(btn.Button, TweenInfo.new(0.25, Enum.EasingStyle.Quad), {BackgroundColor3 = Colors.Primary}):Play()
                TweenService:Create(btn.Stroke, TweenInfo.new(0.25), {Color = Colors.PrimaryLight, Thickness = 2}):Play()
                TweenService:Create(btn.Label, TweenInfo.new(0.25), {TextColor3 = Colors.TextPrimary}):Play()
                TweenService:Create(btn.Dot, TweenInfo.new(0.25), {BackgroundColor3 = Colors.Success}):Play()
                if not btn.Button:FindFirstChild("UIGradient") then
                    AddGradient(btn.Button, Colors.Primary, Colors.PrimaryDark, 45)
                end
            else
                TweenService:Create(btn.Button, TweenInfo.new(0.25, Enum.EasingStyle.Quad), {BackgroundColor3 = Color3.fromRGB(22, 22, 22)}):Play()
                TweenService:Create(btn.Stroke, TweenInfo.new(0.25), {Color = Colors.Border, Thickness = 1.5}):Play()
                TweenService:Create(btn.Label, TweenInfo.new(0.25), {TextColor3 = Colors.TextSecondary}):Play()
                TweenService:Create(btn.Dot, TweenInfo.new(0.25), {BackgroundColor3 = Color3.fromRGB(80, 80, 80)}):Play()
                if btn.Button:FindFirstChild("UIGradient") then
                    btn.Button.UIGradient:Destroy()
                end
            end
        end
    end)
end

local InfoFrame = Instance.new("Frame")
InfoFrame.Size = UDim2.new(1, -16, 0, 45)
InfoFrame.Position = UDim2.new(0, 8, 0, 102)
InfoFrame.BackgroundColor3 = Color3.fromRGB(22, 22, 22)
InfoFrame.BorderSizePixel = 0
InfoFrame.Parent = TeleportCard

local InfoCorner = Instance.new("UICorner")
InfoCorner.CornerRadius = UDim.new(0, 9)
InfoCorner.Parent = InfoFrame

local InfoStroke = Instance.new("UIStroke")
InfoStroke.Color = Colors.Border
InfoStroke.Thickness = 1.5
InfoStroke.Parent = InfoFrame

AddGradient(InfoFrame, Color3.fromRGB(22, 22, 22), Color3.fromRGB(18, 18, 18), 90)

local InfoText = Instance.new("TextLabel")
InfoText.Size = UDim2.new(1, -16, 1, 0)
InfoText.Position = UDim2.new(0, 8, 0, 0)
InfoText.BackgroundTransparency = 1
InfoText.Text = "Selecione um slot"
InfoText.Font = Enum.Font.Gotham
InfoText.TextSize = 9
InfoText.TextColor3 = Colors.TextSecondary
InfoText.TextXAlignment = Enum.TextXAlignment.Center
InfoText.TextWrapped = true
InfoText.Parent = InfoFrame

CreateActionButton(TeleportCard, 8, 155, 64, 35, "Save", function()
    local character = player.Character
    if character and character:FindFirstChild("HumanoidRootPart") then
        State.SavedPositions[State.CurrentSaveSlot] = character.HumanoidRootPart.CFrame
        CreatePopup("Teleport", "Posição salva!")
    end
end)

CreateActionButton(TeleportCard, 79, 155, 64, 35, "TP", function()
    if State.SavedPositions[State.CurrentSaveSlot] then
        local character = player.Character
        if character and character:FindFirstChild("HumanoidRootPart") then
            character.HumanoidRootPart.CFrame = State.SavedPositions[State.CurrentSaveSlot]
            CreatePopup("Teleport", "Teleportado!")
        end
    else
        CreatePopup("Erro", "Sem posição!")
    end
end)

CreateActionButton(TeleportCard, 150, 155, 142, 35, "Clear", function()
    State.SavedPositions[State.CurrentSaveSlot] = nil
    CreatePopup("Limpeza", "Slot limpo")
end)

-- VSYNC page content
local LaserCard = CreateCard(Pages.Vsync, 115, "Auto Laser Cape", "L")
LaserCard.LayoutOrder = 1
CreateToggleButton(LaserCard, 8, 44, 296, 36, "Enable Auto Laser", function(state)
    if state then startAutoLaser() else stopAutoLaser() end
end)

local PlatformCard = CreateCard(Pages.Vsync, 105, "Platform Scout", "P")
PlatformCard.LayoutOrder = 2
CreateToggleButton(PlatformCard, 8, 44, 296, 36, "Spawn Platform", function(state)
    State.Platform.Enabled = state
    if state then
        spawnPlatform()
        CreatePopup("Platform", "Spawnada!")
    else
        stopPlatform()
        CreatePopup("Platform", "Removida")
    end
end)

local XRayCard = CreateCard(Pages.Vsync, 110, "X-Ray Base", "X")
XRayCard.LayoutOrder = 3
CreateToggleButton(XRayCard, 8, 44, 296, 36, "Enable X-Ray Base", function(state)
    if state then EnableXRayBase() else DisableXRayBase() end
end)

-- MOVEMENT content
local FlyCard = CreateCard(Pages.Movement, 110, "Fly System", "F")
FlyCard.LayoutOrder = 1
CreateToggleButton(FlyCard, 8, 44, 296, 36, "Enable Fly", function(state)
    State.Fly.Enabled = state
    if state then EnableFly() else DisableFly() end
end)

local NoclipCard = CreateCard(Pages.Movement, 105, "Noclip", "N")
NoclipCard.LayoutOrder = 2
CreateToggleButton(NoclipCard, 8, 44, 296, 36, "Enable Noclip", function(state)
    State.Noclip.Enabled = state
    if state then EnableNoclip() else DisableNoclip() end
end)

local JumpCard = CreateCard(Pages.Movement, 105, "Jump Tame", "J")
JumpCard.LayoutOrder = 3
CreateToggleButton(JumpCard, 8, 44, 296, 36, "Enable Jump Tame", function(state)
    if state then EnableJumpTame() else DisableJumpTame() end
end)

local SpiderCard = CreateCard(Pages.Movement, 105, "Spider Climb", "S")
SpiderCard.LayoutOrder = 4
CreateToggleButton(SpiderCard, 8, 44, 296, 36, "Enable Spider", function(state)
    if state then EnableSpider() else DisableSpider() end
end)

-- VISUAL page
local ESPCard = CreateCard(Pages.Visual, 140, "ESP & Camera", "E")
ESPCard.LayoutOrder = 1
CreateToggleButton(ESPCard, 8, 44, 143, 36, "Lock Time", function(state)
    if state then EnableLockTimeEsp() else DisableLockTimeEsp() end
end)
CreateToggleButton(ESPCard, 161, 44, 143, 36, "Players", function(state)
    if state then EnablePlayerEsp() else DisablePlayerEsp() end
end)

local FOVCard = CreateCard(Pages.Visual, 185, "Camera FOV", "C")
FOVCard.LayoutOrder = 2
CreateToggleButton(FOVCard, 8, 44, 296, 36, "Enable FOV", function(state)
    if state then EnableFOV() else DisableFOV() end
end)
CreateSlider(FOVCard, 8, 90, 296, 30, 120, 70, function(value)
    State.FOV.Value = value
    if State.FOV.Enabled then
        local camera = workspace.CurrentCamera
        if camera then camera.FieldOfView = value end
    end
end)

-- CRASH page (simulated)
local CrashCard = CreateCard(Pages.Crash, 160, "Crash (Simulado)", "!")
CrashCard.LayoutOrder = 1
local CrashDesc = Instance.new("TextLabel")
CrashDesc.Size = UDim2.new(1, -16, 0, 30)
CrashDesc.Position = UDim2.new(0, 8, 0, 44)
CrashDesc.BackgroundTransparency = 1
CrashDesc.Text = "Ações de crash são SIMULADAS. Não há nenhuma ação prejudicial aqui."
CrashDesc.Font = Enum.Font.Gotham
CrashDesc.TextSize = 10
CrashDesc.TextColor3 = Colors.TextSecondary
CrashDesc.TextWrapped = true
CrashDesc.Parent = CrashCard

-- Player selector
local PlayerDropdown = Instance.new("TextLabel")
PlayerDropdown.Size = UDim2.new(1, -16, 0, 28)
PlayerDropdown.Position = UDim2.new(0, 8, 0, 80)
PlayerDropdown.BackgroundColor3 = Color3.fromRGB(30,30,30)
PlayerDropdown.Text = "Selecionar jogador..."
PlayerDropdown.TextColor3 = Colors.TextSecondary
PlayerDropdown.Font = Enum.Font.Gotham
PlayerDropdown.TextSize = 10
PlayerDropdown.Parent = CrashCard

local PlayerCorner = Instance.new("UICorner"); PlayerCorner.Parent = PlayerDropdown

local DropdownList = Instance.new("Frame")
DropdownList.Size = UDim2.new(0, PlayerDropdown.AbsoluteSize.X, 0, 120)
DropdownList.Position = UDim2.new(0, 8, 0, 108)
DropdownList.BackgroundColor3 = Color3.fromRGB(25,25,25)
DropdownList.BorderSizePixel = 0
DropdownList.Visible = false
DropdownList.Parent = CrashCard

local DropdownCorner = Instance.new("UICorner"); DropdownCorner.Parent = DropdownList
local UIList = Instance.new("UIListLayout"); UIList.Parent = DropdownList; UIList.Padding = UDim.new(0,4)

local function RefreshPlayerList()
    for _,v in pairs(DropdownList:GetChildren()) do
        if v:IsA("TextButton") then v:Destroy() end
    end
    for _,p in ipairs(Players:GetPlayers()) do
        local btn = Instance.new("TextButton")
        btn.Size = UDim2.new(1, -8, 0, 24)
        btn.BackgroundColor3 = Color3.fromRGB(40,40,40)
        btn.BorderSizePixel = 0
        btn.Text = p.Name
        btn.Font = Enum.Font.Gotham
        btn.TextSize = 12
        btn.TextColor3 = Colors.TextPrimary
        btn.Parent = DropdownList
        btn.MouseButton1Click:Connect(function()
            PlayerDropdown.Text = p.Name
            DropdownList.Visible = false
        end)
    end
end

PlayerDropdown.InputBegan:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseButton1 then
        RefreshPlayerList()
        DropdownList.Visible = not DropdownList.Visible
    end
end)

CreateActionButton(CrashCard, 8, 120, 150, 32, "Crash Player (Simulado)", function()
    local selectedName = PlayerDropdown.Text
    local target = nil
    for _,p in ipairs(Players:GetPlayers()) do if p.Name == selectedName then target = p break end end
    if target then
        SimulateCrashPlayer(target)
    else
        CreatePopup("Crash (Simulado)", "Selecione um jogador válido.")
    end
end)

CreateActionButton(CrashCard, 162, 120, 150, 32, "Crash Server (Simulado)", function()
    SimulateCrashServer()
end)

-- DISCORD CARD (organized, centered, bottom of Dashboard)
local DiscordCard = Instance.new("Frame")
DiscordCard.Name = "DiscordCard"
DiscordCard.Size = UDim2.new(1, 0, 0, 140)
DiscordCard.BackgroundColor3 = Color3.fromRGB(18, 18, 18)
DiscordCard.BorderSizePixel = 0
DiscordCard.Parent = Pages.Dashboard
DiscordCard.LayoutOrder = 50

local CardCorner = Instance.new("UICorner")
CardCorner.CornerRadius = UDim.new(0, 12)
CardCorner.Parent = DiscordCard

AddGradient(DiscordCard, Color3.fromRGB(18,18,18), Color3.fromRGB(20,20,20), 135)

local CardStroke = Instance.new("UIStroke")
CardStroke.Color = Colors.Border
CardStroke.Thickness = 1.5
CardStroke.Parent = DiscordCard

-- Centered big icon "D"
local IconFrame = Instance.new("Frame")
IconFrame.Size = UDim2.new(0, 56, 0, 56)
IconFrame.Position = UDim2.new(0.5, -28, 0, 10)
IconFrame.BackgroundColor3 = Colors.Primary
IconFrame.BorderSizePixel = 0
IconFrame.Parent = DiscordCard

local IconCorner = Instance.new("UICorner")
IconCorner.CornerRadius = UDim.new(0, 10)
IconCorner.Parent = IconFrame

AddGradient(IconFrame, Colors.Primary, Colors.PrimaryDark, 45)

local IconText = Instance.new("TextLabel")
IconText.Size = UDim2.new(1,0,1,0)
IconText.BackgroundTransparency = 1
IconText.Text = "D"
IconText.Font = Enum.Font.GothamBlack
IconText.TextSize = 28
IconText.TextColor3 = Colors.TextPrimary
IconText.TextXAlignment = Enum.TextXAlignment.Center
IconText.TextYAlignment = Enum.TextYAlignment.Center
IconText.Parent = IconFrame

-- Description centered under icon
local DiscordDesc = Instance.new("TextLabel")
DiscordDesc.Size = UDim2.new(1, -24, 0, 22)
DiscordDesc.Position = UDim2.new(0, 12, 0, 76)
DiscordDesc.BackgroundTransparency = 1
DiscordDesc.Text = "Junte-se ao nosso Discord"
DiscordDesc.Font = Enum.Font.Gotham
DiscordDesc.TextSize = 11
DiscordDesc.TextColor3 = Colors.TextSecondary
DiscordDesc.TextXAlignment = Enum.TextXAlignment.Center
DiscordDesc.Parent = DiscordCard

-- Buttons container (centered, two buttons side-by-side)
local ButtonsFrame = Instance.new("Frame")
ButtonsFrame.Size = UDim2.new(1, -32, 0, 40)
ButtonsFrame.Position = UDim2.new(0, 16, 0, 100)
ButtonsFrame.BackgroundTransparency = 1
ButtonsFrame.Parent = DiscordCard

local ButtonsLayout = Instance.new("UIListLayout")
ButtonsLayout.FillDirection = Enum.FillDirection.Horizontal
ButtonsLayout.HorizontalAlignment = Enum.HorizontalAlignment.Center
ButtonsLayout.VerticalAlignment = Enum.VerticalAlignment.Center
ButtonsLayout.Padding = UDim.new(0, 12)
ButtonsLayout.Parent = ButtonsFrame

local function CreateSmallButton(parent, text, bgColor, textColor, callback)
    local btn = Instance.new("TextButton")
    btn.Size = UDim2.new(0, 140, 1, 0)
    btn.BackgroundColor3 = bgColor
    btn.BorderSizePixel = 0
    btn.AutoButtonColor = false
    btn.Font = Enum.Font.GothamBold
    btn.Text = ""
    btn.Parent = parent

    local corner = Instance.new("UICorner")
    corner.CornerRadius = UDim.new(0, 10)
    corner.Parent = btn

    AddGradient(btn, bgColor, bgColor:Lerp(Color3.new(0,0,0), 0.08), 0)

    local stroke = Instance.new("UIStroke")
    stroke.Color = Colors.PrimaryLight
    stroke.Thickness = 1
    stroke.Parent = btn

    local label = Instance.new("TextLabel")
    label.Size = UDim2.new(1, -8, 1, 0)
    label.Position = UDim2.new(0, 8, 0, 0)
    label.BackgroundTransparency = 1
    label.Text = text
    label.Font = Enum.Font.GothamBold
    label.TextSize = 12
    label.TextColor3 = textColor
    label.TextXAlignment = Enum.TextXAlignment.Center
    label.Parent = btn

    if callback then
        btn.MouseButton1Click:Connect(callback)
    end

    return btn
end

-- Left: Copiar Link (dark button)
CreateSmallButton(ButtonsFrame, "Copiar Link", Color3.fromRGB(35,35,35), Colors.TextSecondary, function()
    pcall(function() setclipboard("https://discord.gg/SEULINK") end)
    CreatePopup("Discord", "Link copiado!")
end)

-- Right: Entrar (accent primary)
CreateSmallButton(ButtonsFrame, "Entrar", Colors.Primary, Colors.TextPrimary, function()
    pcall(function() setclipboard("https://discord.gg/SEULINK") end)
    CreatePopup("Discord", "Link copiado!")
end)

-- MINIMIZE / CLOSE logic
MinimizeButton.MouseButton1Click:Connect(function()
    local isMin = not (MainFrame.Size.X.Offset == 280)
    if isMin then
        TweenService:Create(MainFrame, TweenInfo.new(0.3, Enum.EasingStyle.Back), {
            Size = UDim2.new(0, 280, 0, 62)
        }):Play()
        ContentArea.Visible = false
        NavFrame.Visible = false
    else
        TweenService:Create(MainFrame, TweenInfo.new(0.3, Enum.EasingStyle.Back), {
            Size = UDim2.new(0, 320, 0, 420)
        }):Play()
        ContentArea.Visible = true
        NavFrame.Visible = true
    end
end)

CloseButton.MouseButton1Click:Connect(function()
    ScreenGui:Destroy()
end)

-- DRAG
local dragging = false
local dragStart = nil
local startPos = nil

Header.InputBegan:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseButton1 then
        dragging = true
        dragStart = input.Position
        startPos = MainFrame.Position
    end
end)

UserInputService.InputChanged:Connect(function(input)
    if dragging and input.UserInputType == Enum.UserInputType.MouseMovement then
        local delta = input.Position - dragStart
        MainFrame.Position = UDim2.new(
            startPos.X.Scale,
            startPos.X.Offset + delta.X,
            startPos.Y.Scale,
            startPos.Y.Offset + delta.Y
        )
    end
end)

UserInputService.InputEnded:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseButton1 then
        dragging = false
    end
end)

-- ANIMATION ON OPEN
MainFrame.Size = UDim2.new(0, 0, 0, 0)
MainFrame.Position = UDim2.new(0.5, 0, 0.5, 0)

wait(0.1)

TweenService:Create(MainFrame, TweenInfo.new(0.5, Enum.EasingStyle.Back, Enum.EasingDirection.Out), {
    Size = UDim2.new(0, 320, 0, 420),
    Position = UDim2.new(0.5, -160, 0.5, -210)
}):Play()

-- FINAL SUCCESS MESSAGE
print("VSYNC Panel Loaded Successfully!")
CreatePopup("VSYNC", "Painel carregado com sucesso!")

-- CLEANUP
ScreenGui.Destroying:Connect(function()
    if laserConnection then pcall(function() laserConnection:Disconnect() end) end
    if platformConnection then pcall(function() platformConnection:Disconnect() end) end
    if jumpTameConnection then pcall(function() jumpTameConnection:Disconnect() end) end
    if State.PlayerEsp.Connection then pcall(function() State.PlayerEsp.Connection:Disconnect() end) end
    if State.LockTimeEsp.Connection then pcall(function() State.LockTimeEsp.Connection:Disconnect() end) end
end)
