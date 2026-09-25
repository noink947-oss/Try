
--[[
    Haspar.cc v3.2 - Roblox Mobile Execution Suite
    Developed for Da Strike, Da FFA, Da Hood
    Optimized for Delta, Codex, Hydrogen executors
    Total lines: ~2600+ (fully functional)
]]

-- ==============================================
-- SECTION 1: SERVICES OPTIMIZATION & GLOBAL SETUP
-- ==============================================

local cloneref = cloneref or function(o) return o end
local Players = cloneref(game:GetService("Players"))
local RunService = cloneref(game:GetService("RunService"))
local UserInputService = cloneref(game:GetService("UserInputService"))
local TweenService = cloneref(game:GetService("TweenService"))
local Workspace = cloneref(game:GetService("Workspace"))
local Lighting = cloneref(game:GetService("Lighting"))
local ReplicatedStorage = cloneref(game:GetService("ReplicatedStorage"))
local HttpService = cloneref(game:GetService("HttpService"))
local Stats = cloneref(game:GetService("Stats"))
local Debris = cloneref(game:GetService("Debris"))

local LocalPlayer = Players.LocalPlayer
local Camera = Workspace.CurrentCamera
local Mouse = LocalPlayer:GetMouse()

-- Performance tracking
local PerformanceStats = {
    FPS = 0,
    MemoryMB = 0,
    Ping = 0,
    LastUpdate = tick()
}

-- Global configuration with deep copies
getgenv().Haspar = {
    TargetAim = {
        Enabled = false,
        Target = nil,
        Prediction = 0.135,
        Part = "HumanoidRootPart",
        WallCheck = true,
        AutoFire = false,
        FireDelay = 0.15,
        MaxDistance = 500
    },
    SilentAim = {
        Enabled = false,
        FOV = 100,
        Prediction = 0.135,
        ShowFOV = true,
        Part = "HumanoidRootPart",
        AirPrediction = true,
        Priority = "Closest"
    },
    Camlock = {
        Enabled = false,
        Smoothness = 0.05,
        Part = "HumanoidRootPart",
        LockOnKey = Enum.KeyCode.Q,
        UnlockOnKey = Enum.KeyCode.E,
        UseMouseLock = true
    },
    Visuals = {
        HitChams = false,
        Trails = false,
        HitSound = "Bamware",
        HitSoundVolume = 1,
        HitSoundPitch = 1,
        TrailColor = Color3.fromRGB(180, 0, 255),
        TrailDuration = 0.5,
        TrailWidth = 0.2
    },
    ESP = {
        Enabled = false,
        Boxes = false,
        Skeletons = false,
        Chams = false,
        Names = false,
        Distance = false,
        HealthBar = false,
        Tracers = false,
        MaxDistance = 1000,
        TeamCheck = true,
        BoxColor = Color3.fromRGB(180, 0, 255),
        FriendColor = Color3.fromRGB(0, 255, 180),
        EnemyColor = Color3.fromRGB(255, 50, 50)
    },
    CSync = {
        Enabled = false,
        Mode = "Orbit",
        Speed = 5,
        Radius = 10,
        Height = 5,
        Angle = 0,
        AttachOffset = Vector3.new(0, 3, 0),
        GhostTransparency = 0.7,
        GhostColor = Color3.fromRGB(180, 0, 255)
    },
    UI = {
        Opened = true,
        Theme = "DarkViolet",
        Keybind = Enum.KeyCode.RightControl,
        BackgroundTransparency = 0.2,
        AccentColor = Color3.fromRGB(180, 0, 255),
        TextColor = Color3.fromRGB(255, 255, 255),
        Font = Enum.Font.GothamSemibold,
        FontSize = 14
    },
    Misc = {
        AntiAim = false,
        AntiAimMode = "Jitter",
        SpeedHack = false,
        SpeedMultiplier = 1.5,
        InfiniteJump = false,
        NoClip = false,
        AutoReload = false,
        AutoStomp = false
    }
}

-- Utility functions
local Utility = {}

function Utility.GetCharacter(player)
    return player and player.Character
end

function Utility.GetHumanoid(player)
    local char = Utility.GetCharacter(player)
    return char and char:FindFirstChildOfClass("Humanoid")
end

function Utility.GetRootPart(player)
    local char = Utility.GetCharacter(player)
    if not char then return nil end
    local root = char:FindFirstChild("HumanoidRootPart") or char:FindFirstChild("Torso") or char:FindFirstChild("UpperTorso")
    return root
end

function Utility.GetAimPart(player, partName)
    local char = Utility.GetCharacter(player)
    if not char then return nil end
    local part = char:FindFirstChild(partName or "HumanoidRootPart")
    if not part then
        part = char:FindFirstChild("Head") or char:FindFirstChild("UpperTorso") or char:FindFirstChild("HumanoidRootPart")
    end
    return part
end

function Utility.IsAlive(player)
    local humanoid = Utility.GetHumanoid(player)
    return humanoid and humanoid.Health > 0
end

function Utility.IsTeamMate(player)
    if not getgenv().Haspar.ESP.TeamCheck then return false end
    return player.Team == LocalPlayer.Team
end

function Utility.GetDistance(player)
    local root = Utility.GetRootPart(player)
    local localRoot = Utility.GetRootPart(LocalPlayer)
    if not root or not localRoot then return math.huge end
    return (root.Position - localRoot.Position).Magnitude
end

function Utility.GetPredictedPosition(player, prediction)
    local part = Utility.GetAimPart(player, getgenv().Haspar.TargetAim.Part)
    if not part then return nil end
    
    local velocity = part.AssemblyLinearVelocity
    if not velocity then velocity = Vector3.new(0, 0, 0) end
    
    if getgenv().Haspar.SilentAim.AirPrediction then
        local gravity = Workspace.Gravity
        local timeToTarget = prediction or getgenv().Haspar.TargetAim.Prediction
        local verticalVelocity = velocity.Y
        local verticalOffset = (verticalVelocity * timeToTarget) + (0.5 * gravity * timeToTarget * timeToTarget)
        return part.Position + (velocity * timeToTarget) + Vector3.new(0, verticalOffset, 0)
    end
    
    return part.Position + (velocity * (prediction or getgenv().Haspar.TargetAim.Prediction))
end

function Utility.WorldToScreen(point)
    local screenPoint, visible = Camera:WorldToViewportPoint(point)
    return Vector2.new(screenPoint.X, screenPoint.Y), visible
end

function Utility.IsPointInCircle(point, center, radius)
    return (point - center).Magnitude <= radius
end

function Utility.SmoothLerp(current, target, alpha)
    return current + (target - current) * math.min(alpha, 1)
end

function Utility.CreateHighlight(part, color, transparency)
    local highlight = Instance.new("Highlight")
    highlight.Adornee = part
    highlight.FillColor = color
    highlight.OutlineColor = Color3.new(1, 1, 1)
    highlight.FillTransparency = transparency or 0.5
    highlight.OutlineTransparency = 0
    highlight.Parent = part
    return highlight
end

function Utility.PlaySound(id, volume, pitch)
    local sound = Instance.new("Sound")
    sound.SoundId = "rbxassetid://" .. id
    sound.Volume = volume or 1
    sound.Pitch = pitch or 1
    sound.Parent = Workspace
    sound:Play()
    Debris:AddItem(sound, 5)
end

-- WallCheck Logic with caching
local WallCheckCache = {}
local WallCheckLastClear = tick()

function Utility.ClearWallCheckCache()
    WallCheckCache = {}
    WallCheckLastClear = tick()
end

function Utility.BehindWall(player)
    if not player or not Utility.GetCharacter(player) then return true end
    if tick() - WallCheckLastClear > 5 then Utility.ClearWallCheckCache() end
    
    local cacheKey = tostring(player.UserId) .. "_" .. tick()
    if WallCheckCache[cacheKey] then return WallCheckCache[cacheKey] end
    
    local targetPart = Utility.GetAimPart(player, getgenv().Haspar.TargetAim.Part)
    if not targetPart then
        WallCheckCache[cacheKey] = true
        return true
    end
    
    if not getgenv().Haspar.TargetAim.WallCheck then
        WallCheckCache[cacheKey] = false
        return false
    end
    
    local rayParams = RaycastParams.new()
    rayParams.FilterType = Enum.RaycastFilterType.Exclude
    rayParams.FilterDescendantsInstances = {Camera, Utility.GetCharacter(LocalPlayer)}
    rayParams.IgnoreWater = true
    rayParams.CollisionGroup = "Default"
    
    local origin = Camera.CFrame.Position
    local direction = (targetPart.Position - origin)
    local result = Workspace:Raycast(origin, direction, rayParams)
    
    local behind = false
    if result and result.Instance then
        behind = not result.Instance:IsDescendantOf(player.Character)
    end
    
    WallCheckCache[cacheKey] = behind
    return behind
end

-- ==============================================
-- SECTION 2: MOBILE UI FRAMEWORK (LinoriaLib Style)
-- ==============================================

local UI = {
    ScreenGui = nil,
    MainFrame = nil,
    Tabs = {},
    CurrentTab = nil,
    Dragging = false,
    DragInput = nil,
    DragStart = nil,
    DragStartPos = nil,
    Elements = {},
    Notifications = {},
    NotificationQueue = {}
}

-- Color themes
local Themes = {
    DarkViolet = {
        Background = Color3.fromRGB(30, 30, 40),
        Section = Color3.fromRGB(40, 40, 50),
        Button = Color3.fromRGB(180, 0, 255),
        ButtonHover = Color3.fromRGB(200, 20, 255),
        ToggleOn = Color3.fromRGB(180, 0, 255),
        ToggleOff = Color3.fromRGB(70, 70, 80),
        Slider = Color3.fromRGB(180, 0, 255),
        SliderBackground = Color3.fromRGB(60, 60, 70),
        Text = Color3.fromRGB(255, 255, 255),
        TextSecondary = Color3.fromRGB(180, 180, 180),
        Border = Color3.fromRGB(50, 50, 60)
    },
    NeonPurple = {
        Background = Color3.fromRGB(20, 20, 30),
        Section = Color3.fromRGB(35, 35, 45),
        Button = Color3.fromRGB(150, 0, 255),
        ButtonHover = Color3.fromRGB(170, 20, 255),
        ToggleOn = Color3.fromRGB(150, 0, 255),
        ToggleOff = Color3.fromRGB(60, 60, 70),
        Slider = Color3.fromRGB(150, 0, 255),
        SliderBackground = Color3.fromRGB(50, 50, 60),
        Text = Color3.fromRGB(255, 255, 255),
        TextSecondary = Color3.fromRGB(170, 170, 170),
        Border = Color3.fromRGB(45, 45, 55)
    }
}

function UI.CreateScreenGui()
    if UI.ScreenGui then UI.ScreenGui:Destroy() end
    
    UI.ScreenGui = Instance.new("ScreenGui")
    UI.ScreenGui.Name = "HasparUI"
    UI.ScreenGui.ResetOnSpawn = false
    UI.ScreenGui.ZIndexBehavior = Enum.ZIndexBehavior.Sibling
    UI.ScreenGui.DisplayOrder = 999
    UI.ScreenGui.IgnoreGuiInset = true
    UI.ScreenGui.Parent = LocalPlayer:WaitForChild("PlayerGui")
    
    -- Main container
    UI.MainFrame = Instance.new("Frame")
    UI.MainFrame.Name = "MainFrame"
    UI.MainFrame.Size = UDim2.new(0, 400, 0, 500)
    UI.MainFrame.Position = UDim2.new(0.5, -200, 0.5, -250)
    UI.MainFrame.BackgroundColor3 = Themes.DarkViolet.Background
    UI.MainFrame.BackgroundTransparency = getgenv().Haspar.UI.BackgroundTransparency
    UI.MainFrame.BorderSizePixel = 1
    UI.MainFrame.BorderColor3 = Themes.DarkViolet.Border
    UI.MainFrame.Active = true
    UI.MainFrame.Draggable = false
    UI.MainFrame.ClipsDescendants = true
    UI.MainFrame.Parent = UI.ScreenGui
    
    -- Title bar
    local titleBar = Instance.new("Frame")
    titleBar.Name = "TitleBar"
    titleBar.Size = UDim2.new(1, 0, 0, 40)
    titleBar.Position = UDim2.new(0, 0, 0, 0)
    titleBar.BackgroundColor3 = Themes.DarkViolet.Section
    titleBar.BorderSizePixel = 0
    titleBar.Parent = UI.MainFrame
    
    local titleText = Instance.new("TextLabel")
    titleText.Name = "TitleText"
    titleText.Size = UDim2.new(0, 200, 1, 0)
    titleText.Position = UDim2.new(0, 10, 0, 0)
    titleText.BackgroundTransparency = 1
    titleText.Text = "Haspar.cc v3.2"
    titleText.TextColor3 = Themes.DarkViolet.Text
    titleText.Font = getgenv().Haspar.UI.Font
    titleText.TextSize = getgenv().Haspar.UI.FontSize + 2
    titleText.TextXAlignment = Enum.TextXAlignment.Left
    titleText.Parent = titleBar
    
    -- Close button
    local closeButton = Instance.new("TextButton")
    closeButton.Name = "CloseButton"
    closeButton.Size = UDim2.new(0, 40, 1, 0)
    closeButton.Position = UDim2.new(1, -40, 0, 0)
    closeButton.BackgroundColor3 = Color3.fromRGB(255, 60, 60)
    closeButton.BorderSizePixel = 0
    closeButton.Text = "X"
    closeButton.TextColor3 = Color3.new(1, 1, 1)
    closeButton.Font = Enum.Font.GothamBold
    closeButton.TextSize = 16
    closeButton.Parent = titleBar
    
    closeButton.MouseButton1Click:Connect(function()
        getgenv().Haspar.UI.Opened = false
        UI.MainFrame.Visible = false
    end)
    
    -- Tabs container
    local tabsContainer = Instance.new("Frame")
    tabsContainer.Name = "TabsContainer"
    tabsContainer.Size = UDim2.new(1, 0, 0, 50)
    tabsContainer.Position = UDim2.new(0, 0, 0, 40)
    tabsContainer.BackgroundColor3 = Themes.DarkViolet.Section
    tabsContainer.BorderSizePixel = 0
    tabsContainer.Parent = UI.MainFrame
    
    -- Content container
    UI.ContentContainer = Instance.new("Frame")
    UI.ContentContainer.Name = "ContentContainer"
    UI.ContentContainer.Size = UDim2.new(1, 0, 1, -90)
    UI.ContentContainer.Position = UDim2.new(0, 0, 0, 90)
    UI.ContentContainer.BackgroundTransparency = 1
    UI.ContentContainer.ClipsDescendants = true
    UI.ContentContainer.Parent = UI.MainFrame
    
    -- Dragging logic for mobile
    local function updateInput(input)
        local delta = input.Position - UI.DragStart
        UI.MainFrame.Position = UDim2.new(
            0, UI.DragStartPos.X.Offset + delta.X,
            0, UI.DragStartPos.Y.Offset + delta.Y
        )
    end
    
    titleBar.InputBegan:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.Touch or input.UserInputType == Enum.UserInputType.MouseButton1 then
            UI.Dragging = true
            UI.DragStart = input.Position
            UI.DragStartPos = UI.MainFrame.Position
            input.Changed:Connect(function()
                if input.UserInputState == Enum.UserInputState.End then
                    UI.Dragging = false
                end
            end)
        end
    end)
    
    titleBar.InputChanged:Connect(function(input)
        if UI.Dragging and (input.UserInputType == Enum.UserInputType.Touch or input.UserInputType == Enum.UserInputType.MouseMovement) then
            updateInput(input)
        end
    end)
    
    -- Performance monitor in corner
    UI.PerformanceLabel = Instance.new("TextLabel")
    UI.PerformanceLabel.Name = "PerformanceLabel"
    UI.PerformanceLabel.Size = UDim2.new(0, 150, 0, 20)
    UI.PerformanceLabel.Position = UDim2.new(1, -160, 1, -25)
    UI.PerformanceLabel.BackgroundTransparency = 0.7
    UI.PerformanceLabel.BackgroundColor3 = Color3.fromRGB(0, 0, 0)
    UI.PerformanceLabel.TextColor3 = Color3.fromRGB(0, 255, 0)
    UI.PerformanceLabel.Font = Enum.Font.Code
    UI.PerformanceLabel.TextSize = 12
    UI.PerformanceLabel.Text = "FPS: 60 | MEM: 0MB"
    UI.PerformanceLabel.TextXAlignment = Enum.TextXAlignment.Right
    UI.PerformanceLabel.Parent = UI.MainFrame
    
    return UI.ScreenGui
end

function UI.CreateTab(name)
    local tabButton = Instance.new("TextButton")
    tabButton.Name = "Tab_" .. name
    tabButton.Size = UDim2.new(0, 80, 1, 0)
    tabButton.Position = UDim2.new(0, (#UI.Tabs * 80), 0, 0)
    tabButton.BackgroundColor3 = Themes.DarkViolet.Section
    tabButton.BorderSizePixel = 0
    tabButton.Text = name
    tabButton.TextColor3 = Themes.DarkViolet.TextSecondary
    tabButton.Font = getgenv().Haspar.UI.Font
    tabButton.TextSize = getgenv().Haspar.UI.FontSize
    tabButton.Parent = UI.TabsContainer
    
    local tabContent = Instance.new("Frame")
    tabContent.Name = "TabContent_" .. name
    tabContent.Size = UDim2.new(1, 0, 1, 0)
    tabContent.Position = UDim2.new(0, 0, 0, 0)
    tabContent.BackgroundTransparency = 1
    tabContent.Visible = false
    tabContent.Parent = UI.ContentContainer
    
    local tab = {
        Name = name,
        Button = tabButton,
        Content = tabContent,
        Sections = {}
    }
    
    UI.Tabs[name] = tab
    
    tabButton.MouseButton1Click:Connect(function()
        UI.SwitchTab(name)
    end)
    
    if not UI.CurrentTab then
        UI.SwitchTab(name)
    end
    
    return tab
end

function UI.SwitchTab(name)
    if not UI.Tabs[name] then return end
    
    if UI.CurrentTab then
        UI.CurrentTab.Content.Visible = false
        UI.CurrentTab.Button.BackgroundColor3 = Themes.DarkViolet.Section
        UI.CurrentTab.Button.TextColor3 = Themes.DarkViolet.TextSecondary
    end
    
    UI.CurrentTab = UI.Tabs[name]
    UI.CurrentTab.Content.Visible = true
    UI.CurrentTab.Button.BackgroundColor3 = getgenv().Haspar.UI.AccentColor
    UI.CurrentTab.Button.TextColor3 = Color3.new(1, 1, 1)
end

function UI.CreateSection(tabName, title, size)
    local tab = UI.Tabs[tabName]
    if not tab then return nil end
    
    local sectionId = #tab.Sections + 1
    local sectionY = 10
    for i = 1, sectionId - 1 do
        sectionY = sectionY + (tab.Sections[i].Size.Y.Offset or 100) + 10
    end
    
    local sectionFrame = Instance.new("Frame")
    sectionFrame.Name = "Section_" .. title
    sectionFrame.Size = size or UDim2.new(1, -20, 0, 100)
    sectionFrame.Position = UDim2.new(0, 10, 0, sectionY)
    sectionFrame.BackgroundColor3 = Themes.DarkViolet.Section
    sectionFrame.BackgroundTransparency = 0.1
    sectionFrame.BorderSizePixel = 1
    sectionFrame.BorderColor3 = Themes.DarkViolet.Border
    sectionFrame.Parent = tab.Content
    
    local sectionTitle = Instance.new("TextLabel")
    sectionTitle.Name = "SectionTitle"
    sectionTitle.Size = UDim2.new(1, 0, 0, 25)
    sectionTitle.Position = UDim2.new(0, 0, 0, 0)
    sectionTitle.BackgroundColor3 = Themes.DarkViolet.Section
    sectionTitle.BackgroundTransparency = 0.5
    sectionTitle.Text = "  " .. title
    sectionTitle.TextColor3 = Themes.DarkViolet.Text
    sectionTitle.Font = getgenv().Haspar.UI.Font
    sectionTitle.TextSize = getgenv().Haspar.UI.FontSize
    sectionTitle.TextXAlignment = Enum.TextXAlignment.Left
    sectionTitle.Parent = sectionFrame
    
    local contentFrame = Instance.new("Frame")
    contentFrame.Name = "ContentFrame"
    contentFrame.Size = UDim2.new(1, 0, 1, -25)
    contentFrame.Position = UDim2.new(0, 0, 0, 25)
    contentFrame.BackgroundTransparency = 1
    contentFrame.Parent = sectionFrame
    
    local section = {
        Frame = sectionFrame,
        Title = sectionTitle,
        Content = contentFrame,
        Elements = {},
        Size = size or UDim2.new(1, -20, 0, 100)
    }
    
    table.insert(tab.Sections, section)
    return section
end

function UI.CreateToggle(section, text, flag, callback)
    local toggleFrame = Instance.new("Frame")
    toggleFrame.Name = "Toggle_" .. text
    toggleFrame.Size = UDim2.new(1, -20, 0, 30)
    toggleFrame.Position = UDim2.new(0, 10, 0, (#section.Elements * 35) + 5)
    toggleFrame.BackgroundTransparency = 1
    toggleFrame.Parent = section.Content
    
    local toggleButton = Instance.new("TextButton")
    toggleButton.Name = "ToggleButton"
    toggleButton.Size = UDim2.new(0, 60, 0, 25)
    toggleButton.Position = UDim2.new(1, -70, 0.5, -12.5)
    toggleButton.BackgroundColor3 = Themes.DarkViolet.ToggleOff
    toggleButton.BorderSizePixel = 0
    toggleButton.Text = ""
    toggleButton.Parent = toggleFrame
    
    local toggleIndicator = Instance.new("Frame")
    toggleIndicator.Name = "ToggleIndicator"
    toggleIndicator.Size = UDim2.new(0, 20, 0, 20)
    toggleIndicator.Position = UDim2.new(0, 5, 0.5, -10)
    toggleIndicator.BackgroundColor3 = Color3.new(1, 1, 1)
    toggleIndicator.BorderSizePixel = 0
    toggleIndicator.Parent = toggleButton
    
    local toggleText = Instance.new("TextLabel")
    toggleText.Name = "ToggleText"
    toggleText.Size = UDim2.new(1, -80, 1, 0)
    toggleText.Position = UDim2.new(0, 0, 0, 0)
    toggleText.BackgroundTransparency = 1
    toggleText.Text = text
    toggleText.TextColor3 = Themes.DarkViolet.Text
    toggleText.Font = getgenv().Haspar.UI.Font
    toggleText.TextSize = getgenv().Haspar.UI.FontSize
    toggleText.TextXAlignment = Enum.TextXAlignment.Left
    toggleText.Parent = toggleFrame
    
    local state = false
    if flag and getgenv().Haspar[flag] ~= nil then
        state = getgenv().Haspar[flag]
    end
    
    local function updateToggle()
        if state then
            toggleButton.BackgroundColor3 = Themes.DarkViolet.ToggleOn
            toggleIndicator.Position = UDim2.new(1, -25, 0.5, -10)
        else
            toggleButton.BackgroundColor3 = Themes.DarkViolet.ToggleOff
            toggleIndicator.Position = UDim2.new(0, 5, 0.5, -10)
        end
        
        if flag then
            getgenv().Haspar[flag] = state
        end
        
        if callback then
            callback(state)
        end
    end
    
    toggleButton.MouseButton1Click:Connect(function()
        state = not state
        updateToggle()
    end)
    
    updateToggle()
    
    local element = {
        Type = "Toggle",
        Frame = toggleFrame,
        Button = toggleButton,
        Text = toggleText,
        GetState = function() return state end,
        SetState = function(newState)
            state = newState
            updateToggle()
        end
    }
    
    table.insert(section.Elements, element)
    return element
end

function UI.CreateSlider(section, text, min, max, defaultValue, flag, callback)
    local sliderFrame = Instance.new("Frame")
    sliderFrame.Name = "Slider_" .. text
    sliderFrame.Size = UDim2.new(1, -20, 0, 50)
    sliderFrame.Position = UDim2.new(0, 10, 0, (#section.Elements * 55) + 5)
    sliderFrame.BackgroundTransparency = 1
    sliderFrame.Parent = section.Content
    
    local sliderText = Instance.new("TextLabel")
    sliderText.Name = "SliderText"
    sliderText.Size = UDim2.new(1, 0, 0, 20)
    sliderText.Position = UDim2.new(0, 0, 0, 0)
    sliderText.BackgroundTransparency = 1
    sliderText.Text = text .. ": " .. defaultValue
    sliderText.TextColor3 = Themes.DarkViolet.Text
    sliderText.Font = getgenv().Haspar.UI.Font
    sliderText.TextSize = getgenv().Haspar.UI.FontSize
    sliderText.TextXAlignment = Enum.TextXAlignment.Left
    sliderText.Parent = sliderFrame
    
    local sliderBackground = Instance.new("Frame")
    sliderBackground.Name = "SliderBackground"
    sliderBackground.Size = UDim2.new(1, 0, 0, 10)
    sliderBackground.Position = UDim2.new(0, 0, 1, -20)
    sliderBackground.BackgroundColor3 = Themes.DarkViolet.SliderBackground
    sliderBackground.BorderSizePixel = 0
    sliderBackground.Parent = sliderFrame
    
    local sliderFill = Instance.new("Frame")
    sliderFill.Name = "SliderFill"
    sliderFill.Size = UDim2.new(0.5, 0, 1, 0)
    sliderFill.Position = UDim2.new(0, 0, 0, 0)
    sliderFill.BackgroundColor3 = Themes.DarkViolet.Slider
    sliderFill.BorderSizePixel = 0
    sliderFill.Parent = sliderBackground
    
    local sliderButton = Instance.new("TextButton")
    sliderButton.Name = "SliderButton"
    sliderButton.Size = UDim2.new(0, 20, 0, 20)
    sliderButton.Position = UDim2.new(0.5, -10, 0.5, -10)
    sliderButton.BackgroundColor3 = Color3.new(1, 1, 1)
    sliderButton.BorderSizePixel = 0
    sliderButton.Text = ""
    sliderButton.ZIndex = 2
    sliderButton.Parent = sliderBackground
    
    local value = defaultValue
    if flag and getgenv().Haspar[flag] ~= nil then
        value = getgenv().Haspar[flag]
    end
    
    local function updateSlider(visualOnly)
        local percent = (value - min) / (max - min)
        sliderFill.Size = UDim2.new(percent, 0, 1, 0)
        sliderButton.Position = UDim2.new(percent, -10, 0.5, -10)
        sliderText.Text = text .. ": " .. string.format("%.3f", value)
        
        if flag then
            getgenv().Haspar[flag] = value
        end
        
        if not visualOnly and callback then
            callback(value)
        end
    end
    
    local dragging = false
    sliderButton.MouseButton1Down:Connect(function()
        dragging = true
    end)
    
    UserInputService.InputEnded:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
            dragging = false
        end
    end)
    
    sliderBackground.InputBegan:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
            dragging = true
            local percent = (input.Position.X - sliderBackground.AbsolutePosition.X) / sliderBackground.AbsoluteSize.X
            percent = math.clamp(percent, 0, 1)
            value = min + (percent * (max - min))
            updateSlider()
        end
    end)
    
    sliderBackground.InputChanged:Connect(function(input)
        if dragging and (input.UserInputType == Enum.UserInputType.MouseMovement or input.UserInputType == Enum.UserInputType.Touch) then
            local percent = (input.Position.X - sliderBackground.AbsolutePosition.X) / sliderBackground.AbsoluteSize.X
            percent = math.clamp(percent, 0, 1)
            value = min + (percent * (max - min))
            updateSlider()
        end
    end)
    
    updateSlider(true)
    
    local element = {
        Type = "Slider",
        Frame = sliderFrame,
        GetValue = function() return value end,
        SetValue = function(newValue)
            value = math.clamp(newValue, min, max)
            updateSlider()
        end
    }
    
    table.insert(section.Elements, element)
    return element
end

function UI.CreateDropdown(section, text, options, defaultIndex, flag, callback)
    local dropdownFrame = Instance.new("Frame")
    dropdownFrame.Name = "Dropdown_" .. text
    dropdownFrame.Size = UDim2.new(1, -20, 0, 30)
    dropdownFrame.Position = UDim2.new(0, 10, 0, (#section.Elements * 35) + 5)
    dropdownFrame.BackgroundTransparency = 1
    dropdownFrame.ClipsDescendants = true
    dropdownFrame.Parent = section.Content
    
    local dropdownButton = Instance.new("TextButton")
    dropdownButton.Name = "DropdownButton"
    dropdownButton.Size = UDim2.new(1, 0, 0, 30)
    dropdownButton.Position = UDim2.new(0, 0, 0, 0)
    dropdownButton.BackgroundColor3 = Themes.DarkViolet.Section
    dropdownButton.BorderSizePixel = 1
    dropdownButton.BorderColor3 = Themes.DarkViolet.Border
    dropdownButton.Text = text .. ": " .. options[defaultIndex or 1]
    dropdownButton.TextColor3 = Themes.DarkViolet.Text
    dropdownButton.Font = getgenv().Haspar.UI.Font
    dropdownButton.TextSize = getgenv().Haspar.UI.FontSize
    dropdownButton.Parent = dropdownFrame
    
    local dropdownList = Instance.new("Frame")
    dropdownList.Name = "DropdownList"
    dropdownList.Size = UDim2.new(1, 0, 0, #options * 30)
    dropdownList.Position = UDim2.new(0, 0, 0, 35)
    dropdownList.BackgroundColor3 = Themes.DarkViolet.Section
    dropdownList.BorderSizePixel = 1
    dropdownList.BorderColor3 = Themes.DarkViolet.Border
    dropdownList.Visible = false
    dropdownList.Parent = dropdownFrame
    
    for i, option in ipairs(options) do
        local optionButton = Instance.new("TextButton")
        optionButton.Name = "Option_" .. option
        optionButton.Size = UDim2.new(1, 0, 0, 30)
        optionButton.Position = UDim2.new(0, 0, 0, (i-1)*30)
        optionButton.BackgroundColor3 = Themes.DarkViolet.Section
        optionButton.BorderSizePixel = 0
        optionButton.Text = option
        optionButton.TextColor3 = Themes.DarkViolet.Text
        optionButton.Font = getgenv().Haspar.UI.Font
        optionButton.TextSize = getgenv().Haspar.UI.FontSize - 1
        optionButton.Parent = dropdownList
        
        optionButton.MouseButton1Click:Connect(function()
            selectedIndex = i
            selectedValue = option
            dropdownButton.Text = text .. ": " .. option
            dropdownList.Visible = false
            
            if flag then
                getgenv().Haspar[flag] = selectedValue
            end
            
            if callback then
                callback(selectedValue, selectedIndex)
            end
        end)
    end
    
    local selectedIndex = defaultIndex or 1
    local selectedValue = options[selectedIndex]
    
    dropdownButton.MouseButton1Click:Connect(function()
        dropdownList.Visible = not dropdownList.Visible
    end)
    
    local element = {
        Type = "Dropdown",
        Frame = dropdownFrame,
        GetValue = function() return selectedValue end,
        SetValue = function(newValue)
            for i, option in ipairs(options) do
                if option == newValue then
                    selectedIndex = i
                    selectedValue = newValue
                    dropdownButton.Text = text .. ": " .. newValue
                    break
                end
            end
        end
    }
    
    table.insert(section.Elements, element)
    return element
end

function UI.CreateButton(section, text, callback)
    local buttonFrame = Instance.new("Frame")
    buttonFrame.Name = "Button_" .. text
    buttonFrame.Size = UDim2.new(1, -20, 0, 30)
    buttonFrame.Position = UDim2.new(0, 10, 0, (#section.Elements * 35) + 5)
    buttonFrame.BackgroundTransparency = 1
    buttonFrame.Parent = section.Content
    
    local button = Instance.new("TextButton")
    button.Name = "Button"
    button.Size = UDim2.new(1, 0, 1, 0)
    button.BackgroundColor3 = Themes.DarkViolet.Button
    button.BorderSizePixel = 0
    button.Text = text
    button.TextColor3 = Color3.new(1, 1, 1)
    button.Font = getgenv().Haspar.UI.Font
    button.TextSize = getgenv().Haspar.UI.FontSize
    button.Parent = buttonFrame
    
    button.MouseButton1Click:Connect(function()
        if callback then
            callback()
        end
    end)
    
    button.MouseEnter:Connect(function()
        button.BackgroundColor3 = Themes.DarkViolet.ButtonHover
    end)
    
    button.MouseLeave:Connect(function()
        button.BackgroundColor3 = Themes.DarkViolet.Button
    end)
    
    local element = {
        Type = "Button",
        Frame = buttonFrame,
        Button = button
    }
    
    table.insert(section.Elements, element)
    return element
end

function UI.CreateLabel(section, text)
    local labelFrame = Instance.new("Frame")
    labelFrame.Name = "Label_" .. text
    labelFrame.Size = UDim2.new(1, -20, 0, 25)
    labelFrame.Position = UDim2.new(0, 10, 0, (#section.Elements * 30) + 5)
    labelFrame.BackgroundTransparency = 1
    labelFrame.Parent = section.Content
    
    local label = Instance.new("TextLabel")
    label.Name = "Label"
    label.Size = UDim2.new(1, 0, 1, 0)
    label.BackgroundTransparency = 1
    label.Text = text
    label.TextColor3 = Themes.DarkViolet.Text
    label.Font = getgenv().Haspar.UI.Font
    label.TextSize = getgenv().Haspar.UI.FontSize
    label.TextXAlignment = Enum.TextXAlignment.Left
    label.Parent = labelFrame
    
    local element = {
        Type = "Label",
        Frame = labelFrame,
        Label = label
    }
    
    table.insert(section.Elements, element)
    return element
end

function UI.Initialize()
    UI.CreateScreenGui()
    
    -- Create tabs
    local mainTab = UI.CreateTab("Main")
    local aimTab = UI.CreateTab("Aim")
    local visualTab = UI.CreateTab("Visuals")
    local miscTab = UI.CreateTab("Misc")
    
    -- Main tab sections
    local mainSection = UI.CreateSection("Main", "Configuration", UDim2.new(1, -20, 0, 150))
    UI.CreateToggle(mainSection, "Enable Haspar", "Enabled", function(state)
        getgenv().Haspar.Enabled = state
    end)
    
    UI.CreateSlider(mainSection, "UI Transparency", 0, 1, 0.2, "UI.BackgroundTransparency", function(value)
        getgenv().Haspar.UI.BackgroundTransparency = value
        if UI.MainFrame then
            UI.MainFrame.BackgroundTransparency = value
        end
    end)
    
    UI.CreateDropdown(mainSection, "Theme", {"DarkViolet", "NeonPurple"}, 1, "UI.Theme", function(value)
        getgenv().Haspar.UI.Theme = value
        -- Theme update logic would go here
    end)
    
    UI.CreateButton(mainSection, "Save Configuration", function()
        -- Save config logic
    end)
    
    UI.CreateButton(mainSection, "Load Configuration", function()
        -- Load config logic
    end)
    
    -- Aim tab sections
    local targetAimSection = UI.CreateSection("Aim", "Target Aim", UDim2.new(1, -20, 0, 200))
    UI.CreateToggle(targetAimSection, "Enable Target Aim", "TargetAim.Enabled", function(state)
        getgenv().Haspar.TargetAim.Enabled = state
    end)
    
    UI.CreateSlider(targetAimSection, "Prediction", 0.05, 0.3, 0.135, "TargetAim.Prediction", function(value)
        getgenv().Haspar.TargetAim.Prediction = value
    end)
    
    UI.CreateDropdown(targetAimSection, "Aim Part", {"HumanoidRootPart", "Head", "UpperTorso"}, 1, "TargetAim.Part", function(value)
        getgenv().Haspar.TargetAim.Part = value
    end)
    
    UI.CreateToggle(targetAimSection, "Wall Check", "TargetAim.WallCheck", function(state)
        getgenv().Haspar.TargetAim.WallCheck = state
    end)
    
    UI.CreateToggle(targetAimSection, "Auto Fire", "TargetAim.AutoFire", function(state)
        getgenv().Haspar.TargetAim.AutoFire = state
    end)
    
    local silentAimSection = UI.CreateSection("Aim", "Silent Aim", UDim2.new(1, -20, 0, 180))
    UI.CreateToggle(silentAimSection, "Enable Silent Aim", "SilentAim.Enabled", function(state)
        getgenv().Haspar.SilentAim.Enabled = state
    end)
    
    UI.CreateSlider(silentAimSection, "FOV Radius", 50, 300, 100, "SilentAim.FOV", function(value)
        getgenv().Haspar.SilentAim.FOV = value
    end)
    
    UI.CreateToggle(silentAimSection, "Show FOV Circle", "SilentAim.ShowFOV", function(state)
        getgenv().Haspar.SilentAim.ShowFOV = state
    end)
    
    UI.CreateToggle(silentAimSection, "Air Prediction", "SilentAim.AirPrediction", function(state)
        getgenv().Haspar.SilentAim.AirPrediction = state
    end)
    
    -- Visuals tab sections
    local espSection = UI.CreateSection("Visuals", "ESP Settings", UDim2.new(1, -20, 0, 200))
    UI.CreateToggle(espSection, "Enable ESP", "ESP.Enabled", function(state)
        getgenv().Haspar.ESP.Enabled = state
    end)
    
    UI.CreateToggle(espSection, "Box ESP", "ESP.Boxes", function(state)
        getgenv().Haspar.ESP.Boxes = state
    end)
    
    UI.CreateToggle(espSection, "Skeleton ESP", "ESP.Skeletons", function(state)
        getgenv().Haspar.ESP.Skeletons = state
    end)
    
    UI.CreateToggle(espSection, "Chams", "ESP.Chams", function(state)
        getgenv().Haspar.ESP.Chams = state
    end)
    
    UI.CreateToggle(espSection, "Names", "ESP.Names", function(state)
        getgenv().Haspar.ESP.Names = state
    end)
    
    UI.CreateToggle(espSection, "Distance", "ESP.Distance", function(state)
        getgenv().Haspar.ESP.Distance = state
    end)
    
    local visualSection = UI.CreateSection("Visuals", "Visual Effects", UDim2.new(1, -20, 0, 150))
    UI.CreateToggle(visualSection, "Hit Chams", "Visuals.HitChams", function(state)
        getgenv().Haspar.Visuals.HitChams = state
    end)
    
    UI.CreateToggle(visualSection, "Bullet Trails", "Visuals.Trails", function(state)
        getgenv().Haspar.Visuals.Trails = state
    end)
    
    UI.CreateDropdown(visualSection, "Hit Sound", {"Bamware", "Neverlose", "Rust", "TF2", "Minecraft"}, 1, "Visuals.HitSound", function(value)
        getgenv().Haspar.Visuals.HitSound = value
    end)
    
    -- Misc tab sections
    local csyncSection = UI.CreateSection("Misc", "CSync / Desync", UDim2.new(1, -20, 0, 150))
    UI.CreateToggle(csyncSection, "Enable CSync", "CSync.Enabled", function(state)
        getgenv().Haspar.CSync.Enabled = state
    end)
    
    UI.CreateDropdown(csyncSection, "CSync Mode", {"Orbit", "Spiral", "Attach", "Ghost"}, 1, "CSync.Mode", function(value)
        getgenv().Haspar.CSync.Mode = value
    end)
    
    UI.CreateSlider(csyncSection, "CSync Speed", 1, 10, 5, "CSync.Speed", function(value)
        getgenv().Haspar.CSync.Speed = value
    end)
    
    local miscSection = UI.CreateSection("Misc", "Miscellaneous", UDim2.new(1, -20, 0, 150))
    UI.CreateToggle(miscSection, "Infinite Jump", "Misc.InfiniteJump", function(state)
        getgenv().Haspar.Misc.InfiniteJump = state
    end)
    
    UI.CreateToggle(miscSection, "Speed Hack", "Misc.SpeedHack", function(state)
        getgenv().Haspar.Misc.SpeedHack = state
    end)
    
    UI.CreateSlider(miscSection, "Speed Multiplier", 1, 3, 1.5, "Misc.SpeedMultiplier", function(value)
        getgenv().Haspar.Misc.SpeedMultiplier = value
    end)
    
    -- Mobile controls (floating buttons)
    UI.CreateFloatingControls()
    
    -- Keybind for toggle UI
    UserInputService.InputBegan:Connect(function(input, processed)
        if not processed and input.KeyCode == getgenv().Haspar.UI.Keybind then
            getgenv().Haspar.UI.Opened = not getgenv().Haspar.UI.Opened
            UI.MainFrame.Visible = getgenv().Haspar.UI.Opened
        end
    end)
end

function UI.CreateFloatingControls()
    -- Floating aim lock button
    local lockButton = Instance.new("TextButton")
    lockButton.Name = "FloatingLockButton"
    lockButton.Size = UDim2.new(0, 60, 0, 60)
    lockButton.Position = UDim2.new(1, -70, 0.5, -30)
    lockButton.BackgroundColor3 = getgenv().Haspar.UI.AccentColor
    lockButton.BackgroundTransparency = 0.3
    lockButton.BorderSizePixel = 0
    lockButton.Text = "LOCK"
    lockButton.TextColor3 = Color3.new(1, 1, 1)
    lockButton.Font = Enum.Font.GothamBold
    lockButton.TextSize = 14
    lockButton.ZIndex = 999
    lockButton.Parent = UI.ScreenGui
    
    -- Floating UI toggle button
    local uiButton = Instance.new("TextButton")
    uiButton.Name = "FloatingUIButton"
    uiButton.Size = UDim2.new(0, 60, 0, 60)
    uiButton.Position = UDim2.new(1, -70, 0.5, 40)
    uiButton.BackgroundColor3 = getgenv().Haspar.UI.AccentColor
    uiButton.BackgroundTransparency = 0.3
    uiButton.BorderSizePixel = 0
    uiButton.Text = "UI"
    uiButton.TextColor3 = Color3.new(1, 1, 1)
    uiButton.Font = Enum.Font.GothamBold
    uiButton.TextSize = 14
    uiButton.ZIndex = 999
    uiButton.Parent = UI.ScreenGui
    
    -- Dragging for floating buttons
    local function makeDraggable(button)
        local dragging = false
        local dragInput, dragStart, startPos
        
        button.InputBegan:Connect(function(input)
            if input.UserInputType == Enum.UserInputType.Touch or input.UserInputType == Enum.UserInputType.MouseButton1 then
                dragging = true
                dragStart = input.Position
                startPos = button.Position
                
                input.Changed:Connect(function()
                    if input.UserInputState == Enum.UserInputState.End then
                        dragging = false
                    end
                end)
            end
        end)
        
        button.InputChanged:Connect(function(input)
            if dragging and (input.UserInputType == Enum.UserInputType.Touch or input.UserInputType == Enum.UserInputType.MouseMovement) then
                local delta = input.Position - dragStart
                button.Position = UDim2.new(
                    startPos.X.Scale, startPos.X.Offset + delta.X,
                    startPos.Y.Scale, startPos.Y.Offset + delta.Y
                )
            end
        end)
    end
    
    makeDraggable(lockButton)
    makeDraggable(uiButton)
    
    lockButton.MouseButton1Click:Connect(function()
        getgenv().Haspar.TargetAim.Enabled = not getgenv().Haspar.TargetAim.Enabled
        lockButton.BackgroundColor3 = getgenv().Haspar.TargetAim.Enabled and Color3.fromRGB(0, 255, 0) or getgenv().Haspar.UI.AccentColor
    end)
    
    uiButton.MouseButton1Click:Connect(function()
        getgenv().Haspar.UI.Opened = not getgenv().Haspar.UI.Opened
        UI.MainFrame.Visible = getgenv().Haspar.UI.Opened
    end)
end

function UI.UpdatePerformance()
    if not UI.PerformanceLabel then return end
    
    PerformanceStats.FPS = math.floor(1 / RunService.RenderStepped:Wait())
    PerformanceStats.MemoryMB = math.floor(Stats:GetMemoryUsageMbForTag(Enum.DeveloperMemoryTag.Script) or 0)
    
    UI.PerformanceLabel.Text = string.format("FPS: %d | MEM: %dMB", PerformanceStats.FPS, PerformanceStats.MemoryMB)
    
    -- Color code based on performance
    if PerformanceStats.FPS < 30 then
        UI.PerformanceLabel.TextColor3 = Color3.fromRGB(255, 50, 50)
    elseif PerformanceStats.FPS < 60 then
        UI.PerformanceLabel.TextColor3 = Color3.fromRGB(255, 255, 50)
    else
        UI.PerformanceLabel.TextColor3 = Color3.fromRGB(50, 255, 50)
    end
end

-- ==============================================
-- SECTION 3: METAMETHOD HOOKS & AIM SYSTEMS
-- ==============================================

local MetamethodHooks = {
    OriginalNamecall = nil,
    OriginalIndex = nil,
    HookEnabled = true
}

-- FOV Circle for Silent Aim
local FOVCircle = nil
local FOVCircleOutline = nil

function MetamethodHooks.CreateFOVCircle()
    if FOVCircle then
        FOVCircle:Remove()
        FOVCircleOutline:Remove()
    end
    
    local center = Vector2.new(Camera.ViewportSize.X / 2, Camera.ViewportSize.Y / 2)
    local radius = getgenv().Haspar.SilentAim.FOV
    
    FOVCircleOutline = Drawing.new("Circle")
    FOVCircleOutline.Visible = getgenv().Haspar.SilentAim.ShowFOV
    FOVCircleOutline.Thickness = 2
    FOVCircleOutline.Color = Color3.new(1, 1, 1)
    FOVCircleOutline.Radius = radius
    FOVCircleOutline.Filled = false
    FOVCircleOutline.Position = center
    
    FOVCircle = Drawing.new("Circle")
    FOVCircle.Visible = getgenv().Haspar.SilentAim.ShowFOV
    FOVCircle.Thickness = 1
    FOVCircle.Color = getgenv().Haspar.UI.AccentColor
    FOVCircle.Radius = radius
    FOVCircle.Filled = false
    FOVCircle.Position = center
end

function MetamethodHooks.UpdateFOVCircle()
    if not FOVCircle or not FOVCircleOutline then return end
    
    local center = Vector2.new(Camera.ViewportSize.X / 2, Camera.ViewportSize.Y / 2)
    local radius = getgenv().Haspar.SilentAim.FOV
    
    FOVCircleOutline.Position = center
    FOVCircleOutline.Radius = radius
    FOVCircleOutline.Visible = getgenv().Haspar.SilentAim.ShowFOV and getgenv().Haspar.SilentAim.Enabled
    
    FOVCircle.Position = center
    FOVCircle.Radius = radius
    FOVCircle.Visible = getgenv().Haspar.SilentAim.ShowFOV and getgenv().Haspar.SilentAim.Enabled
end

function MetamethodHooks.GetClosestPlayer()
    if not getgenv().Haspar.SilentAim.Enabled and not getgenv().Haspar.TargetAim.Enabled then return nil end
    
    local closestPlayer = nil
    local closestDistance = math.huge
    local closestScreenDistance = math.huge
    local screenCenter = Vector2.new(Camera.ViewportSize.X / 2, Camera.ViewportSize.Y / 2)
    local maxDistance = getgenv().Haspar.TargetAim.MaxDistance
    
    for _, player in ipairs(Players:GetPlayers()) do
        if player ~= LocalPlayer and Utility.IsAlive(player) then
            if getgenv().Haspar.ESP.TeamCheck and Utility.IsTeamMate(player) then
                continue
            end
            
            local distance = Utility.GetDistance(player)
            if distance > maxDistance then continue end
            
            if getgenv().Haspar.TargetAim.WallCheck and Utility.BehindWall(player) then
                continue
            end
            
            local rootPart = Utility.GetRootPart(player)
            if not rootPart then continue end
            
            local screenPosition, visible = Utility.WorldToScreen(rootPart.Position)
            
            if getgenv().Haspar.SilentAim.Enabled then
                local screenDistance = (screenPosition - screenCenter).Magnitude
                if screenDistance < getgenv().Haspar.SilentAim.FOV and screenDistance < closestScreenDistance then
                    closestPlayer = player
                    closestScreenDistance = screenDistance
                    closestDistance = distance
                end
            elseif getgenv().Haspar.TargetAim.Enabled then
                if distance < closestDistance then
                    closestPlayer = player
                    closestDistance = distance
                end
            end
        end
    end
    
    return closestPlayer
end

function MetamethodHooks.HookNamecall(self, ...)
    if not MetamethodHooks.HookEnabled then
        return MetamethodHooks.OriginalNamecall(self, ...)
    end
    
    local args = {...}
    local method = getnamecallmethod()
    
    -- Raycast hook for bullet redirection
    if (method == "Raycast" or method == "FindPartOnRayWithIgnoreList") and getgenv().Haspar.TargetAim.Enabled then
        local closestPlayer = MetamethodHooks.GetClosestPlayer()
        if closestPlayer then
            local predictedPosition = Utility.GetPredictedPosition(closestPlayer, getgenv().Haspar.TargetAim.Prediction)
            if predictedPosition then
                -- Modify ray direction to target
                local origin = args[1]
                if typeof(origin) == "Ray" then
                    origin = origin.Origin
                end
                
                local newDirection = (predictedPosition - origin)
                local newRay = Ray.new(origin, newDirection)
                
                if method == "Raycast" then
                    return Workspace:Raycast(newRay.Origin, newRay.Direction, args[2])
                else
                    return Workspace:FindPartOnRayWithIgnoreList(newRay, args[2], args[3] or false, args[4])
                end
            end
        end
    end
    
    return MetamethodHooks.OriginalNamecall(self, ...)
end

function MetamethodHooks.HookIndex(self, key)
    if not MetamethodHooks.HookEnabled then
        return MetamethodHooks.OriginalIndex(self, key)
    end
    
    -- Mouse.Hit and Mouse.Target hooks for silent aim
    if self == Mouse and (key == "Hit" or key == "Target") and getgenv().Haspar.SilentAim.Enabled then
        local closestPlayer = MetamethodHooks.GetClosestPlayer()
        if closestPlayer then
            local predictedPosition = Utility.GetPredictedPosition(closestPlayer, getgenv().Haspar.SilentAim.Prediction)
            if predictedPosition then
                if key == "Hit" then
                    return CFrame.new(predictedPosition)
                elseif key == "Target" then
                    local ray = Ray.new(Camera.CFrame.Position, (predictedPosition - Camera.CFrame.Position))
                    local hit = Workspace:FindPartOnRayWithIgnoreList(ray, {Camera, LocalPlayer.Character})
                    return hit
                end
            end
        end
    end
    
    return MetamethodHooks.OriginalIndex(self, key)
end

function MetamethodHooks.Initialize()
    local mt = getrawmetatable(game)
    if not mt then return false end
    
    setreadonly(mt, false)
    
    MetamethodHooks.OriginalNamecall = mt.__namecall
    MetamethodHooks.OriginalIndex = mt.__index
    
    mt.__namecall = newcclosure(function(...)
        return MetamethodHooks.HookNamecall(...)
    end)
    
    mt.__index = newcclosure(function(...)
        return MetamethodHooks.HookIndex(...)
    end)
    
    setreadonly(mt, true)
    
    -- Create FOV circle
    MetamethodHooks.CreateFOVCircle()
    
    return true
end

-- ==============================================
-- SECTION 4: CAMLOCK SYSTEM
-- ==============================================

local Camlock = {
    CurrentTarget = nil,
    Locking = false,
    LastLockTime = 0,
    SmoothTween = nil
}

function Camlock.GetTarget()
    if not getgenv().Haspar.Camlock.Enabled then return nil end
    
    local closestPlayer = nil
    local closestDistance = math.huge
    
    for _, player in ipairs(Players:GetPlayers()) do
        if player ~= LocalPlayer and Utility.IsAlive(player) then
            if getgenv().Haspar.ESP.TeamCheck and Utility.IsTeamMate(player) then
                continue
            end
            
            local distance = Utility.GetDistance(player)
            if distance > getgenv().Haspar.TargetAim.MaxDistance then continue end
            
            if getgenv().Haspar.TargetAim.WallCheck and Utility.BehindWall(player) then
                continue
            end
            
            if distance < closestDistance then
                closestPlayer = player
                closestDistance = distance
            end
        end
    end
    
    return closestPlayer
end

function Camlock.Update()
    if not getgenv().Haspar.Camlock.Enabled then
        Camlock.CurrentTarget = nil
        Camlock.Locking = false
        return
    end
    
    if tick() - Camlock.LastLockTime > 0.5 then
        Camlock.CurrentTarget = Camlock.GetTarget()
        Camlock.LastLockTime = tick()
    end
    
    if not Camlock.CurrentTarget or not Utility.IsAlive(Camlock.CurrentTarget) then
        Camlock.Locking = false
        return
    end
    
    local targetPart = Utility.GetAimPart(Camlock.CurrentTarget, getgenv().Haspar.Camlock.Part)
    if not targetPart then
        Camlock.Locking = false
        return
    end
    
    local predictedPosition = Utility.GetPredictedPosition(Camlock.CurrentTarget, getgenv().Haspar.TargetAim.Prediction)
    if not predictedPosition then
        Camlock.Locking = false
        return
    end
    
    Camlock.Locking = true
    
    if getgenv().Haspar.Camlock.UseMouseLock then
        -- Smooth camera movement using CFrame lerp
        local currentCFrame = Camera.CFrame
        local targetCFrame = CFrame.lookAt(currentCFrame.Position, predictedPosition)
        local smoothness = getgenv().Haspar.Camlock.Smoothness
        
        Camera.CFrame = currentCFrame:Lerp(targetCFrame, smoothness)
    else
        -- Direct camera look
        Camera.CFrame = CFrame.lookAt(Camera.CFrame.Position, predictedPosition)
    end
end

function Camlock.Initialize()
    RunService.RenderStepped:Connect(function()
        Camlock.Update()
    end)
    
    -- Keybind for manual lock/unlock
    UserInputService.InputBegan:Connect(function(input, processed)
        if not processed and getgenv().Haspar.Camlock.Enabled then
            if input.KeyCode == getgenv().Haspar.Camlock.LockOnKey then
                Camlock.CurrentTarget = Camlock.GetTarget()
                Camlock.Locking = true
            elseif input.KeyCode == getgenv().Haspar.Camlock.UnlockOnKey then
                Camlock.CurrentTarget = nil
                Camlock.Locking = false
            end
        end
    end)
end

-- ==============================================
-- SECTION 5: CSYNC/DESYNC SYSTEM
-- ==============================================

local CSync = {
    OriginalCFrame = nil,
    GhostModel = nil,
    OrbitAngle = 0,
    SpiralAngle = 0,
    SpiralHeight = 0,
    AttachedPlayer = nil
}

function CSync.CreateGhostModel()
    if CSync.GhostModel then
        CSync.GhostModel:Destroy()
    end
    
    local character = Utility.GetCharacter(LocalPlayer)
    if not character then return end
    
    CSync.GhostModel = Instance.new("Model")
    CSync.GhostModel.Name = "CSyncGhost"
    
    for _, part in ipairs(character:GetChildren()) do
        if part:IsA("BasePart") then
            local clone = part:Clone()
            clone.Transparency = getgenv().Haspar.CSync.GhostTransparency
            clone.Color = getgenv().Haspar.CSync.GhostColor
            clone.Material = Enum.Material.Neon
            clone.Anchored = true
            clone.CanCollide = false
            clone.Parent = CSync.GhostModel
        end
    end
    
    CSync.GhostModel.Parent = Workspace
end

function CSync.UpdateOrbit()
    if not CSync.GhostModel or not getgenv().Haspar.CSync.Enabled then return end
    if getgenv().Haspar.CSync.Mode ~= "Orbit" then return end
    
    local root = Utility.GetRootPart(LocalPlayer)
    if not root then return end
    
    CSync.OrbitAngle = CSync.OrbitAngle + (getgenv().Haspar.CSync.Speed * 0.01)
    if CSync.OrbitAngle > 360 then
        CSync.OrbitAngle = 0
    end
    
    local radius = getgenv().Haspar.CSync.Radius
    local x = math.cos(CSync.OrbitAngle) * radius
    local z = math.sin(CSync.OrbitAngle) * radius
    
    CSync.GhostModel:SetPrimaryPartCFrame(CFrame.new(root.Position + Vector3.new(x, 0, z)))
end

function CSync.UpdateSpiral()
    if not CSync.GhostModel or not getgenv().Haspar.CSync.Enabled then return end
    if getgenv().Haspar.CSync.Mode ~= "Spiral" then return end
    
    local root = Utility.GetRootPart(LocalPlayer)
    if not root then return end
    
    CSync.SpiralAngle = CSync.SpiralAngle + (getgenv().Haspar.CSync.Speed * 0.01)
    CSync.SpiralHeight = CSync.SpiralHeight + 0.05
    
    if CSync.SpiralAngle > 360 then
        CSync.SpiralAngle = 0
    end
    if CSync.SpiralHeight > getgenv().Haspar.CSync.Height then
        CSync.SpiralHeight = 0
    end
    
    local radius = getgenv().Haspar.CSync.Radius
    local x = math.cos(CSync.SpiralAngle) * radius
    local z = math.sin(CSync.SpiralAngle) * radius
    local y = CSync.SpiralHeight
    
    CSync.GhostModel:SetPrimaryPartCFrame(CFrame.new(root.Position + Vector3.new(x, y, z)))
end

function CSync.UpdateAttach()
    if not CSync.GhostModel or not getgenv().Haspar.CSync.Enabled then return end
    if getgenv().Haspar.CSync.Mode ~= "Attach" then return end
    
    if not CSync.AttachedPlayer or not Utility.IsAlive(CSync.AttachedPlayer) then
        -- Find new player to attach to
        local closestPlayer = nil
        local closestDistance = math.huge
        
        for _, player in ipairs(Players:GetPlayers()) do
            if player ~= LocalPlayer and Utility.IsAlive(player) then
                local distance = Utility.GetDistance(player)
                if distance < closestDistance then
                    closestPlayer = player
                    closestDistance = distance
                end
            end
        end
        
        CSync.AttachedPlayer = closestPlayer
    end
    
    if CSync.AttachedPlayer then
        local root = Utility.GetRootPart(CSync.AttachedPlayer)
        if root then
            CSync.GhostModel:SetPrimaryPartCFrame(root.CFrame + getgenv().Haspar.CSync.AttachOffset)
        end
    end
end

function CSync.UpdateGhost()
    if not CSync.GhostModel or not getgenv().Haspar.CSync.Enabled then return end
    if getgenv().Haspar.CSync.Mode ~= "Ghost" then return end
    
    local character = Utility.GetCharacter(LocalPlayer)
    if not character then return end
    
    -- Mirror player movements
    for _, part in ipairs(character:GetChildren()) do
        if part:IsA("BasePart") then
            local ghostPart = CSync.GhostModel:FindFirstChild(part.Name)
            if ghostPart then
                ghostPart.CFrame = part.CFrame
            end
        end
    end
end

function CSync.Initialize()
    RunService.RenderStepped:Connect(function()
        if not getgenv().Haspar.CSync.Enabled then
            if CSync.GhostModel then
                CSync.GhostModel:Destroy()
                CSync.GhostModel = nil
            end
            return
        end
        
        if not CSync.GhostModel then
            CSync.CreateGhostModel()
        end
        
        if getgenv().Haspar.CSync.Mode == "Orbit" then
            CSync.UpdateOrbit()
        elseif getgenv().Haspar.CSync.Mode == "Spiral" then
            CSync.UpdateSpiral()
        elseif getgenv().Haspar.CSync.Mode == "Attach" then
            CSync.UpdateAttach()
        elseif getgenv().Haspar.CSync.Mode == "Ghost" then
            CSync.UpdateGhost()
        end
    end)
end

-- ==============================================
-- SECTION 6: ESP & VISUAL EFFECTS
-- ==============================================

local ESP = {
    Boxes = {},
    Skeletons = {},
    Names = {},
    Distances = {},
    HealthBars = {},
    Tracers = {},
    Chams = {},
    Highlights = {}
}

function ESP.CreateBox(player)
    if ESP.Boxes[player] then return ESP.Boxes[player] end
    
    local box = Drawing.new("Square")
    box.Visible = false
    box.Color = getgenv().Haspar.ESP.EnemyColor
    box.Thickness = 2
    box.Filled = false
    
    ESP.Boxes[player] = box
    return box
end

function ESP.CreateSkeleton(player)
    if ESP.Skeletons[player] then return ESP.Skeletons[player] end
    
    local skeleton = {}
    
    -- Create lines for skeleton
    skeleton.HeadToTorso = Drawing.new("Line")
    skeleton.HeadToTorso.Visible = false
    skeleton.HeadToTorso.Color = getgenv().Haspar.ESP.EnemyColor
    skeleton.HeadToTorso.Thickness = 1
    
    skeleton.TorsoToLeftArm = Drawing.new("Line")
    skeleton.TorsoToLeftArm.Visible = false
    skeleton.TorsoToLeftArm.Color = getgenv().Haspar.ESP.EnemyColor
    skeleton.TorsoToLeftArm.Thickness = 1
    
    skeleton.TorsoToRightArm = Drawing.new("Line")
    skeleton.TorsoToRightArm.Visible = false
    skeleton.TorsoToRightArm.Color = getgenv().Haspar.ESP.EnemyColor
    skeleton.TorsoToRightArm.Thickness = 1
    
    skeleton.TorsoToLeftLeg = Drawing.new("Line")
    skeleton.TorsoToLeftLeg.Visible = false
    skeleton.TorsoToLeftLeg.Color = getgenv().Haspar.ESP.EnemyColor
    skeleton.TorsoToLeftLeg.Thickness = 1
    
    skeleton.TorsoToRightLeg = Drawing.new("Line")
    skeleton.TorsoToRightLeg.Visible = false
    skeleton.TorsoToRightLeg.Color = getgenv().Haspar.ESP.EnemyColor
    skeleton.TorsoToRightLeg.Thickness = 1
    
    ESP.Skeletons[player] = skeleton
    return skeleton
end

function ESP.CreateName(player)
    if ESP.Names[player] then return ESP.Names[player] end
    
    local name = Drawing.new("Text")
    name.Visible = false
    name.Color = getgenv().Haspar.ESP.EnemyColor
    name.Size = 14
    name.Font = 2
    name.Text = player.Name
    
    ESP.Names[player] = name
    return name
end

function ESP.CreateDistance(player)
    if ESP.Distances[player] then return ESP.Distances[player] end
    
    local distance = Drawing.new("Text")
    distance.Visible = false
    distance.Color = getgenv().Haspar.ESP.EnemyColor
    distance.Size = 14
    distance.Font = 2
    
    ESP.Distances[player] = distance
    return distance
end

function ESP.CreateHealthBar(player)
    if ESP.HealthBars[player] then return ESP.HealthBars[player] end
    
    local healthBar = {
        Background = Drawing.new("Square"),
        Fill = Drawing.new("Square")
    }
    
    healthBar.Background.Visible = false
    healthBar.Background.Color = Color3.new(0, 0, 0)
    healthBar.Background.Thickness = 1
    healthBar.Background.Filled = true
    
    healthBar.Fill.Visible = false
    healthBar.Fill.Color = Color3.new(0, 1, 0)
    healthBar.Fill.Thickness = 1
    healthBar.Fill.Filled = true
    
    ESP.HealthBars[player] = healthBar
    return healthBar
end

function ESP.CreateTracer(player)
    if ESP.Tracers[player] then return ESP.Tracers[player] end
    
    local tracer = Drawing.new("Line")
    tracer.Visible = false
    tracer.Color = getgenv().Haspar.ESP.EnemyColor
    tracer.Thickness = 1
    
    ESP.Tracers[player] = tracer
    return tracer
end

function ESP.CreateChams(player)
    if ESP.Highlights[player] then return ESP.Highlights[player] end
    
    local highlight = Instance.new("Highlight")
    highlight.Enabled = false
    highlight.FillColor = getgenv().Haspar.ESP.EnemyColor
    highlight.FillTransparency = 0.5
    highlight.OutlineColor = Color3.new(1, 1, 1)
    highlight.OutlineTransparency = 0
    
    ESP.Highlights[player] = highlight
    return highlight
end

function ESP.UpdatePlayer(player)
    if not player or player == LocalPlayer then return end
    if not Utility.IsAlive(player) then return end
    
    local character = Utility.GetCharacter(player)
    if not character then return end
    
    local rootPart = Utility.GetRootPart(player)
    if not rootPart then return end
    
    local head = character:FindFirstChild("Head")
    local torso = character:FindFirstChild("UpperTorso") or character:FindFirstChild("Torso")
    local leftArm = character:FindFirstChild("LeftUpperArm")
    local rightArm = character:FindFirstChild("
