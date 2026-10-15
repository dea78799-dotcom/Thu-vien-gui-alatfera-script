--[[
    ============================================
    ALATFERA LIB V14 - PRO EDITION (FIXED)
    Notify X, History, Mini Search, Copy/Paste Config
    ============================================
]]

local AlatferaLib = {}
AlatferaLib.__index = AlatferaLib

local CoreGui = game:GetService("CoreGui")
local Players = game:GetService("Players")
local LocalPlayer = Players.LocalPlayer
local UserInputService = game:GetService("UserInputService")
local TweenService = game:GetService("TweenService")
local HttpService = game:GetService("HttpService")
local RunService = game:GetService("RunService")
local TextService = game:GetService("TextService")
local Stats = game:GetService("Stats")
local Lighting = game:GetService("Lighting")

local ParentContainer = (gethui and gethui()) or CoreGui or LocalPlayer:WaitForChild("PlayerGui")

local ColorPresets = {
    blue    = Color3.fromRGB(0, 170, 255),   red     = Color3.fromRGB(245, 65, 85),
    green   = Color3.fromRGB(45, 200, 110),  purple  = Color3.fromRGB(160, 90, 245),
    orange  = Color3.fromRGB(255, 130, 40),  yellow  = Color3.fromRGB(250, 200, 40),
    cyan    = Color3.fromRGB(0, 220, 220),   pink    = Color3.fromRGB(255, 110, 180),
    rose    = Color3.fromRGB(240, 50, 100),  emerald = Color3.fromRGB(16, 185, 129),
    indigo  = Color3.fromRGB(99, 102, 241),  violet  = Color3.fromRGB(139, 92, 246),
    amber   = Color3.fromRGB(245, 158, 11),  teal    = Color3.fromRGB(20, 184, 166),
    neon    = Color3.fromRGB(57, 255, 20),   dark    = Color3.fromRGB(15, 17, 23),
}

local function ParseColor(val, default)
    if type(val) == "string" then
        if val:sub(1,1) == "#" then
            pcall(function() default = Color3.fromHex(val) end)
            return default or ColorPresets.blue
        elseif ColorPresets[val:lower()] then
            return ColorPresets[val:lower()]
        end
    elseif typeof(val) == "Color3" then
        return val
    end
    return default or ColorPresets.blue
end

local function CopyToClipboard(st)
    if setclipboard then setclipboard(st)
    elseif toclipboard then toclipboard(st) end
end

local function GetFromClipboard()
    if getclipboard then
        local ok, res = pcall(getclipboard)
        if ok and type(res) == "string" then return res end
    end
    return ""
end

local function CreateRipple(button)
    button.ClipsDescendants = true
    local m = UserInputService:GetMouseLocation()
    local xPos = m.X - button.AbsolutePosition.X
    local yPos = m.Y - button.AbsolutePosition.Y
    local ripple = Instance.new("Frame")
    ripple.BackgroundColor3 = Color3.fromRGB(255, 255, 255)
    ripple.BackgroundTransparency = 0.75
    ripple.BorderSizePixel = 0
    ripple.Position = UDim2.new(0, xPos, 0, yPos)
    ripple.Size = UDim2.new(0, 0, 0, 0)
    ripple.ZIndex = 5
    ripple.Parent = button
    local c = Instance.new("UICorner"); c.CornerRadius = UDim.new(1, 0); c.Parent = ripple
    local maxSize = math.max(button.AbsoluteSize.X, button.AbsoluteSize.Y) * 2.5
    local tween = TweenService:Create(ripple, TweenInfo.new(0.55, Enum.EasingStyle.Quad, Enum.EasingDirection.Out), {
        Size = UDim2.new(0, maxSize, 0, maxSize),
        Position = UDim2.new(0, xPos - maxSize/2, 0, yPos - maxSize/2),
        BackgroundTransparency = 1,
    })
    tween:Play()
    tween.Completed:Connect(function() ripple:Destroy() end)
end

local function AttachButtonEffects(button, trackFn, hoverColor)
    local scale = Instance.new("UIScale"); scale.Scale = 1; scale.Parent = button
    trackFn(button.MouseEnter:Connect(function()
        TweenService:Create(scale, TweenInfo.new(0.18, Enum.EasingStyle.Quart), {Scale = 1.02}):Play()
        if hoverColor then TweenService:Create(button, TweenInfo.new(0.18), {BackgroundColor3 = hoverColor}):Play() end
    end))
    trackFn(button.MouseLeave:Connect(function()
        TweenService:Create(scale, TweenInfo.new(0.18, Enum.EasingStyle.Quart), {Scale = 1}):Play()
    end))
    trackFn(button.MouseButton1Down:Connect(function()
        TweenService:Create(scale, TweenInfo.new(0.1), {Scale = 0.97}):Play()
    end))
    trackFn(button.MouseButton1Up:Connect(function()
        TweenService:Create(scale, TweenInfo.new(0.15, Enum.EasingStyle.Back, Enum.EasingDirection.Out), {Scale = 1.02}):Play()
    end))
end

function AlatferaLib.CreateWindow(config)
    config = config or {}
    local hubTitle       = config.Title or "Alatfera Script"
    local hubVersion     = config.Version or "v14.0"
    local showPlaytime   = (config.ShowPlaytime == nil) and true or config.ShowPlaytime
    local scriptNameText = config.LoadingScriptName or "Alatfera Script"
    local statusText     = config.LoadingStatus or "Đang tải giao diện..."

    local cornerRadius = config.CornerRadius or 12
    local themeColor   = ParseColor(config.ThemeColor, ColorPresets.blue)
    local enableStroke = (config.Stroke == nil) and true or config.Stroke
    local strokeColor  = ParseColor(config.StrokeColor, Color3.fromRGB(45, 50, 65))

    local notifyFontSize  = config.NotifyFontSize or 12
    local notifyTitleSize = config.NotifyTitleSize or (notifyFontSize + 1)

    local useKeySystem = config.KeySystem or false
    local correctKey   = config.Key or "AlatferaKey123"
    local discordLink  = config.DiscordLink or "https://discord.gg/alatfera"
    local maxAttempts  = config.MaxAttempts or 10
    local remainingAttempts = maxAttempts

    local enableBlur = (config.BlurBackground == nil) and true or config.BlurBackground
    local enableParticles = (config.ParticleEffects == nil) and true or config.ParticleEffects
    local blurSize = config.BlurSize or 20
    local particleCount = config.ParticleCount or 15
    local particleColor = ParseColor(config.ParticleColor, themeColor)

    local configFile = config.ConfigFile or "Alatfera_Config.json"
    local savedData = {}
    if readfile and isfile and isfile(configFile) then
        pcall(function() savedData = HttpService:JSONDecode(readfile(configFile)) end)
    end
    local function SaveCurrentConfig()
        if writefile then pcall(function() writefile(configFile, HttpService:JSONEncode(savedData)) end) end
    end

    if ParentContainer:FindFirstChild("AlatferaGui") then ParentContainer["AlatferaGui"]:Destroy() end

    local ScreenGui = Instance.new("ScreenGui")
    ScreenGui.Name = "AlatferaGui"
    ScreenGui.ResetOnSpawn = false
    ScreenGui.IgnoreGuiInset = true
    ScreenGui.DisplayOrder = 999
    ScreenGui.ZIndexBehavior = Enum.ZIndexBehavior.Sibling
    ScreenGui.Parent = ParentContainer

    local connections = {}
    local function Track(c) table.insert(connections, c); return c end
    local presetAppliers = {}
    local themeTargets = {}

    local function registerTheme(instance, property)
        table.insert(themeTargets, {obj = instance, prop = property})
    end

    local function hexStringOf(c)
        return string.format("#%02X%02X%02X",
            math.floor(c.R * 255 + 0.5), math.floor(c.G * 255 + 0.5), math.floor(c.B * 255 + 0.5))
    end

    -- ============ KEY SYSTEM ============
    if useKeySystem then
        local keyPassed = (savedData["_SavedKey"] == correctKey)
        if not keyPassed then
            local KeyFrame = Instance.new("Frame")
            KeyFrame.Size = UDim2.new(0, 360, 0, 220)
            KeyFrame.Position = UDim2.new(0.5, -180, 0.5, -110)
            KeyFrame.BackgroundColor3 = Color3.fromRGB(15, 17, 23)
            KeyFrame.BorderSizePixel = 0
            KeyFrame.Parent = ScreenGui
            local KC = Instance.new("UICorner"); KC.CornerRadius = UDim.new(0, cornerRadius); KC.Parent = KeyFrame
            if enableStroke then
                local KS = Instance.new("UIStroke"); KS.Color = strokeColor; KS.Thickness = 1.5; KS.Parent = KeyFrame
            end
            local kScale = Instance.new("UIScale"); kScale.Scale = 0.85; kScale.Parent = KeyFrame
            TweenService:Create(kScale, TweenInfo.new(0.35, Enum.EasingStyle.Back, Enum.EasingDirection.Out), {Scale = 1}):Play()

            local KTitle = Instance.new("TextLabel")
            KTitle.Size = UDim2.new(1, 0, 0, 34); KTitle.Position = UDim2.new(0, 0, 0, 12)
            KTitle.BackgroundTransparency = 1; KTitle.Text = "🔑 XÁC NHẬN KEY"
            KTitle.TextColor3 = themeColor; KTitle.Font = Enum.Font.GothamBold; KTitle.TextSize = 17
            KTitle.Parent = KeyFrame
            registerTheme(KTitle, "TextColor3")

            local KSub = Instance.new("TextLabel")
            KSub.Size = UDim2.new(1, -20, 0, 20); KSub.Position = UDim2.new(0, 10, 0, 42)
            KSub.BackgroundTransparency = 1; KSub.Text = "Nhập Key để truy cập " .. hubTitle
            KSub.TextColor3 = Color3.fromRGB(160, 165, 180); KSub.Font = Enum.Font.Gotham; KSub.TextSize = 13
            KSub.Parent = KeyFrame

            local KeyInput = Instance.new("TextBox")
            KeyInput.Size = UDim2.new(1, -30, 0, 36); KeyInput.Position = UDim2.new(0, 15, 0, 70)
            KeyInput.BackgroundColor3 = Color3.fromRGB(24, 27, 36); KeyInput.Text = ""
            KeyInput.PlaceholderText = "Dán Key tại đây..."
            KeyInput.TextColor3 = Color3.fromRGB(240, 240, 245)
            KeyInput.PlaceholderColor3 = Color3.fromRGB(110, 115, 130)
            KeyInput.Font = Enum.Font.Gotham; KeyInput.TextSize = 13
            KeyInput.Parent = KeyFrame
            local IC = Instance.new("UICorner"); IC.CornerRadius = UDim.new(0, cornerRadius - 2); IC.Parent = KeyInput

            local StatusLabel = Instance.new("TextLabel")
            StatusLabel.Size = UDim2.new(1, -20, 0, 18); StatusLabel.Position = UDim2.new(0, 10, 0, 110)
            StatusLabel.BackgroundTransparency = 1; StatusLabel.Text = "Còn lại " .. remainingAttempts .. " lần thử"
            StatusLabel.TextColor3 = Color3.fromRGB(140, 145, 160); StatusLabel.Font = Enum.Font.Gotham; StatusLabel.TextSize = 12
            StatusLabel.Parent = KeyFrame

            local SubmitBtn = Instance.new("TextButton")
            SubmitBtn.Size = UDim2.new(0.46, 0, 0, 34); SubmitBtn.Position = UDim2.new(0, 15, 0, 136)
            SubmitBtn.BackgroundColor3 = themeColor; SubmitBtn.Text = "Xác Nhận"
            SubmitBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
            SubmitBtn.Font = Enum.Font.GothamBold; SubmitBtn.TextSize = 13
            SubmitBtn.Parent = KeyFrame
            local SC = Instance.new("UICorner"); SC.CornerRadius = UDim.new(0, cornerRadius - 2); SC.Parent = SubmitBtn
            registerTheme(SubmitBtn, "BackgroundColor3")
            AttachButtonEffects(SubmitBtn, Track)

            local DiscordBtn = Instance.new("TextButton")
            DiscordBtn.Size = UDim2.new(0.46, 0, 0, 34); DiscordBtn.Position = UDim2.new(0.54, -5, 0, 136)
            DiscordBtn.BackgroundColor3 = Color3.fromRGB(70, 80, 200); DiscordBtn.Text = "📋 Copy Discord"
            DiscordBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
            DiscordBtn.Font = Enum.Font.GothamBold; DiscordBtn.TextSize = 12
            DiscordBtn.Parent = KeyFrame
            local DC = Instance.new("UICorner"); DC.CornerRadius = UDim.new(0, cornerRadius - 2); DC.Parent = DiscordBtn
            AttachButtonEffects(DiscordBtn, Track)

            Track(DiscordBtn.MouseButton1Click:Connect(function()
                CopyToClipboard(discordLink)
                DiscordBtn.Text = "✓ Đã Copy!"
                task.wait(1.5)
                DiscordBtn.Text = "📋 Copy Discord"
            end))

            local dragK, startK, posK
            Track(KeyFrame.InputBegan:Connect(function(input)
                if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
                    dragK = true; startK = input.Position; posK = KeyFrame.Position
                end
            end))
            Track(UserInputService.InputChanged:Connect(function(input)
                if dragK and (input.UserInputType == Enum.UserInputType.MouseMovement or input.UserInputType == Enum.UserInputType.Touch) then
                    local delta = input.Position - startK
                    KeyFrame.Position = UDim2.new(posK.X.Scale, posK.X.Offset + delta.X, posK.Y.Scale, posK.Y.Offset + delta.Y)
                end
            end))
            Track(KeyFrame.InputEnded:Connect(function(input)
                if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then dragK = false end
            end))

            local function VerifyKey()
                if KeyInput.Text == correctKey then
                    StatusLabel.Text = "✓ Key hợp lệ! Đang mở..."
                    StatusLabel.TextColor3 = Color3.fromRGB(45, 200, 110)
                    savedData["_SavedKey"] = correctKey
                    SaveCurrentConfig()
                    TweenService:Create(kScale, TweenInfo.new(0.3, Enum.EasingStyle.Back, Enum.EasingDirection.In), {Scale = 0}):Play()
                    task.wait(0.3)
                    KeyFrame:Destroy()
                    keyPassed = true
                else
                    remainingAttempts = remainingAttempts - 1
                    local origPos = KeyFrame.Position
                    for i = 1, 4 do
                        KeyFrame.Position = origPos + UDim2.new(0, (i % 2 == 0 and 8 or -8), 0, 0)
                        task.wait(0.04)
                    end
                    KeyFrame.Position = origPos
                    if remainingAttempts <= 0 then
                        StatusLabel.Text = "❌ Sai quá " .. maxAttempts .. " lần!"
                        StatusLabel.TextColor3 = Color3.fromRGB(245, 65, 85)
                        task.wait(1)
                        LocalPlayer:Kick("Alatfera Security: Sai Key quá " .. maxAttempts .. " lần!")
                    else
                        StatusLabel.Text = "❌ Sai Key! Còn " .. remainingAttempts .. "/" .. maxAttempts .. " lần"
                        StatusLabel.TextColor3 = Color3.fromRGB(245, 65, 85)
                    end
                end
            end

            Track(SubmitBtn.MouseButton1Click:Connect(VerifyKey))
            Track(KeyInput.FocusLost:Connect(function(enterPressed)
                if enterPressed then VerifyKey() end
            end))

            while not keyPassed do
                if not ScreenGui.Parent then return end
                task.wait(0.1)
            end
        end
    end

    -- ============ LOADING ============
    local LoadingFrame = Instance.new("Frame")
    LoadingFrame.Size = UDim2.new(0, 280, 0, 110)
    LoadingFrame.Position = UDim2.new(0.5, -140, 0.5, -55)
    LoadingFrame.BackgroundColor3 = Color3.fromRGB(15, 17, 23)
    LoadingFrame.BorderSizePixel = 0
    LoadingFrame.Parent = ScreenGui
    local LC = Instance.new("UICorner"); LC.CornerRadius = UDim.new(0, cornerRadius); LC.Parent = LoadingFrame
    if enableStroke then
        local LS = Instance.new("UIStroke"); LS.Color = strokeColor; LS.Thickness = 1.5; LS.Parent = LoadingFrame
    end
    local lScale = Instance.new("UIScale"); lScale.Scale = 0.85; lScale.Parent = LoadingFrame
    TweenService:Create(lScale, TweenInfo.new(0.3, Enum.EasingStyle.Back, Enum.EasingDirection.Out), {Scale = 1}):Play()

    local LoadTitle = Instance.new("TextLabel")
    LoadTitle.Size = UDim2.new(1, 0, 0, 30); LoadTitle.Position = UDim2.new(0, 0, 0, 12)
    LoadTitle.BackgroundTransparency = 1; LoadTitle.Text = scriptNameText
    LoadTitle.TextColor3 = themeColor; LoadTitle.Font = Enum.Font.GothamBold; LoadTitle.TextSize = 17
    LoadTitle.Parent = LoadingFrame

    local LoadStatus = Instance.new("TextLabel")
    LoadStatus.Size = UDim2.new(1, 0, 0, 20); LoadStatus.Position = UDim2.new(0, 0, 0, 40)
    LoadStatus.BackgroundTransparency = 1; LoadStatus.Text = statusText
    LoadStatus.TextColor3 = Color3.fromRGB(160, 165, 180); LoadStatus.Font = Enum.Font.Gotham; LoadStatus.TextSize = 12
    LoadStatus.Parent = LoadingFrame

    local BarBackground = Instance.new("Frame")
    BarBackground.Size = UDim2.new(0.8, 0, 0, 5); BarBackground.Position = UDim2.new(0.1, 0, 0, 72)
    BarBackground.BackgroundColor3 = Color3.fromRGB(30, 34, 46); BarBackground.BorderSizePixel = 0
    BarBackground.Parent = LoadingFrame
    local BC = Instance.new("UICorner"); BC.CornerRadius = UDim.new(1, 0); BC.Parent = BarBackground

    local BarFill = Instance.new("Frame")
    BarFill.Size = UDim2.new(0, 0, 1, 0); BarFill.BackgroundColor3 = themeColor
    BarFill.BorderSizePixel = 0; BarFill.Parent = BarBackground
    local FC = Instance.new("UICorner"); FC.CornerRadius = UDim.new(1, 0); FC.Parent = BarFill

    TweenService:Create(BarFill, TweenInfo.new(1.0, Enum.EasingStyle.Quart, Enum.EasingDirection.Out), {Size = UDim2.new(1, 0, 1, 0)}):Play()

    task.spawn(function()
        while LoadingFrame.Parent do
            local ring = Instance.new("Frame")
            ring.Size = UDim2.new(0, 8, 0, 8)
            ring.Position = UDim2.new(0.5, -4, 1, -22)
            ring.BackgroundTransparency = 1
            ring.Parent = LoadingFrame
            local rc = Instance.new("UICorner"); rc.CornerRadius = UDim.new(1, 0); rc.Parent = ring
            local rs = Instance.new("UIStroke"); rs.Color = themeColor; rs.Thickness = 2; rs.Parent = ring
            TweenService:Create(ring, TweenInfo.new(1.2, Enum.EasingStyle.Quad, Enum.EasingDirection.Out), {
                Size = UDim2.new(0, 50, 0, 50),
                Position = UDim2.new(0.5, -25, 1, -43),
            }):Play()
            TweenService:Create(rs, TweenInfo.new(1.2, Enum.EasingStyle.Quad, Enum.EasingDirection.Out), {
                Transparency = 1, Thickness = 0.5,
            }):Play()
            task.wait(1.2)
            ring:Destroy()
        end
    end)

    task.wait(1.2)
    TweenService:Create(lScale, TweenInfo.new(0.25, Enum.EasingStyle.Back, Enum.EasingDirection.In), {Scale = 0}):Play()
    task.wait(0.25)
    LoadingFrame:Destroy()

    -- ============ MAIN WINDOW ============
    local Window = {}
    Window.ThemeColor = themeColor

    local MainFrame = Instance.new("Frame")
    MainFrame.Size = UDim2.new(0.70, 0, 0.88, 0)
    MainFrame.Position = UDim2.new(0.15, 0, 0.06, 0)
    MainFrame.BackgroundColor3 = Color3.fromRGB(15, 17, 23)
    MainFrame.BorderSizePixel = 0
    MainFrame.ClipsDescendants = true
    MainFrame.Parent = ScreenGui
    local MC = Instance.new("UICorner"); MC.CornerRadius = UDim.new(0, cornerRadius); MC.Parent = MainFrame
    if enableStroke then
        local MS = Instance.new("UIStroke"); MS.Color = strokeColor; MS.Thickness = 2; MS.Parent = MainFrame
    end
    local mainScale = Instance.new("UIScale"); mainScale.Scale = 0.85; mainScale.Parent = MainFrame
    TweenService:Create(mainScale, TweenInfo.new(0.45, Enum.EasingStyle.Back, Enum.EasingDirection.Out), {Scale = 1}):Play()

    local savedBlurState = enableBlur
    local savedParticleState = enableParticles

    local blurEffect = nil
    if enableBlur then
        blurEffect = Instance.new("BlurEffect")
        blurEffect.Size = 0
        blurEffect.Parent = Lighting
        TweenService:Create(blurEffect, TweenInfo.new(0.5, Enum.EasingStyle.Quart), {Size = blurSize}):Play()
    end

    local particleContainer = nil
    local particleList = {}
    local particleConn = nil

    local function createParticles()
        particleContainer = Instance.new("Frame")
        particleContainer.Name = "ParticleContainer"
        particleContainer.Size = UDim2.new(1, 0, 1, 0)
        particleContainer.BackgroundTransparency = 1
        particleContainer.ZIndex = 0
        particleContainer.Parent = ScreenGui
        particleList = {}
        for i = 1, particleCount do
            local p = Instance.new("Frame")
            local size = math.random(3, 8)
            p.Size = UDim2.new(0, size, 0, size)
            p.Position = UDim2.new(math.random(), 0, math.random(), 0)
            p.BackgroundColor3 = particleColor
            p.BorderSizePixel = 0
            p.BackgroundTransparency = math.random(40, 80) / 100
            p.ZIndex = 0
            p.Parent = particleContainer
            local pc = Instance.new("UICorner"); pc.CornerRadius = UDim.new(1, 0); pc.Parent = p
            table.insert(particleList, {
                frame = p,
                speed = math.random(15, 50) / 1000,
                drift = (math.random() - 0.5) * 0.0008,
                wobble = math.random() * math.pi * 2,
            })
        end
        particleConn = RunService.RenderStepped:Connect(function(dt)
            for _, data in ipairs(particleList) do
                local p = data.frame
                if p and p.Parent then
                    data.wobble = data.wobble + dt * 2
                    local newY = p.Position.Y.Scale - data.speed * dt
                    local newX = p.Position.X.Scale + data.drift + math.sin(data.wobble) * 0.0005
                    if newY < -0.05 then newY = 1.05; newX = math.random() end
                    if newX < -0.05 then newX = 1.05 elseif newX > 1.05 then newX = -0.05 end
                    p.Position = UDim2.new(newX, 0, newY, 0)
                end
            end
        end)
        Track(particleConn)
    end

    if enableParticles then createParticles() end

    -- ============ TOPBAR ============
    local Topbar = Instance.new("Frame")
    Topbar.Size = UDim2.new(1, 0, 0, 46)
    Topbar.BackgroundColor3 = Color3.fromRGB(20, 23, 32)
    Topbar.BorderSizePixel = 0
    Topbar.Parent = MainFrame
    local TC = Instance.new("UICorner"); TC.CornerRadius = UDim.new(0, cornerRadius); TC.Parent = Topbar

    local topGradient = Instance.new("UIGradient")
    topGradient.Color = ColorSequence.new({
        ColorSequenceKeypoint.new(0, Color3.fromRGB(20, 23, 32)),
        ColorSequenceKeypoint.new(0.5, Color3.fromRGB(30, 34, 48)),
        ColorSequenceKeypoint.new(1, Color3.fromRGB(20, 23, 32)),
    })
    topGradient.Parent = Topbar
    Track(RunService.RenderStepped:Connect(function()
        topGradient.Offset = Vector2.new(math.sin(tick() * 0.25) * 0.4, 0)
    end))

    local TitleLabel = Instance.new("TextLabel")
    TitleLabel.Size = UDim2.new(0.28, 0, 1, 0); TitleLabel.Position = UDim2.new(0, 18, 0, 0)
    TitleLabel.BackgroundTransparency = 1
    TitleLabel.RichText = true
    TitleLabel.Text = hubTitle .. (hubVersion ~= "" and ("  <font color=\"" .. hexStringOf(themeColor) .. "\">" .. hubVersion .. "</font>") or "")
    TitleLabel.TextColor3 = Color3.fromRGB(255, 255, 255)
    TitleLabel.TextXAlignment = Enum.TextXAlignment.Left
    TitleLabel.Font = Enum.Font.GothamBold; TitleLabel.TextSize = 15
    TitleLabel.Parent = Topbar

    local StatsLabel = Instance.new("TextLabel")
    StatsLabel.Size = UDim2.new(0.20, 0, 1, 0); StatsLabel.Position = UDim2.new(0.28, 10, 0, 0)
    StatsLabel.BackgroundTransparency = 1; StatsLabel.Text = "⚡ 60 FPS | 0 ms"
    StatsLabel.TextColor3 = Color3.fromRGB(150, 155, 170)
    StatsLabel.TextXAlignment = Enum.TextXAlignment.Left
    StatsLabel.Font = Enum.Font.Gotham; StatsLabel.TextSize = 12
    StatsLabel.Parent = Topbar

    local frameCount, lastTime = 0, tick()
    Track(RunService.RenderStepped:Connect(function()
        frameCount = frameCount + 1
        local currentTime = tick()
        if currentTime - lastTime >= 1 then
            local fps = math.floor(frameCount / (currentTime - lastTime))
            local ok, ping = pcall(function() return math.floor(Stats.Network.ServerStatsItem["Data Ping"]:GetValue()) end)
            StatsLabel.Text = string.format("⚡ %d FPS | %d ms", fps, ok and ping or 0)
            frameCount = 0; lastTime = currentTime
        end
    end))

    -- Layout từ phải sang trái: ✕ | 📜 | 🔍 | 🔒 | ⌨ | ? | 📋 | 📥
    local function makeTopBtn(text, xOffset, color, size)
        local b = Instance.new("TextButton")
        b.Size = UDim2.new(0, 34, 0, 34); b.Position = UDim2.new(1, xOffset, 0, 6)
        b.BackgroundTransparency = 1; b.Text = text
        b.TextColor3 = color or Color3.fromRGB(180, 185, 200)
        b.TextSize = size or 16
        b.Font = Enum.Font.GothamBold; b.Parent = Topbar
        return b
    end

    local CloseBtn        = makeTopBtn("✕", -42,  Color3.fromRGB(245, 70, 90), 17)
    local HistoryBtn      = makeTopBtn("📜", -78)
    local SearchToggleBtn = makeTopBtn("🔍", -114)
    local LockBtn         = makeTopBtn("🔓", -150)
    local KeybindBtn      = makeTopBtn("⌨", -186)
    local HelpBtn         = makeTopBtn("?", -222)
    local CopyCfgBtn      = makeTopBtn("📋", -258)
    local PasteCfgBtn     = makeTopBtn("📥", -294)
    local isLocked = false

    -- ============ SEARCH BOX ============
    local SearchBox = Instance.new("TextBox")
    SearchBox.Name = "SearchBox"
    SearchBox.AnchorPoint = Vector2.new(1, 0)
    SearchBox.Size = UDim2.new(0, 0, 0, 28)
    SearchBox.Position = UDim2.new(1, -298, 0, 9)
    SearchBox.BackgroundColor3 = Color3.fromRGB(12, 14, 19)
    SearchBox.Text = ""
    SearchBox.PlaceholderText = "🔍 Tìm kiếm..."
    SearchBox.TextColor3 = Color3.fromRGB(240, 240, 245)
    SearchBox.PlaceholderColor3 = Color3.fromRGB(110, 115, 130)
    SearchBox.Font = Enum.Font.Gotham; SearchBox.TextSize = 12
    SearchBox.ClearTextOnFocus = false
    SearchBox.Visible = false
    SearchBox.Parent = Topbar
    local SrC = Instance.new("UICorner"); SrC.CornerRadius = UDim.new(0, 8); SrC.Parent = SearchBox
    if enableStroke then
        local SrS = Instance.new("UIStroke"); SrS.Color = strokeColor; SrS.Thickness = 1; SrS.Parent = SearchBox
    end

    local isSearchOpen = false
    local function setSearchOpen(open)
        isSearchOpen = open
        if open then
            SearchBox.Visible = true
            SearchBox.Size = UDim2.new(0, 0, 0, 28)
            TweenService:Create(SearchBox, TweenInfo.new(0.25, Enum.EasingStyle.Quart), {Size = UDim2.new(0, 160, 0, 28)}):Play()
            task.wait(0.15)
            SearchBox:CaptureFocus()
        else
            SearchBox.Text = ""
            TweenService:Create(SearchBox, TweenInfo.new(0.25, Enum.EasingStyle.Quart), {Size = UDim2.new(0, 0, 0, 28)}):Play()
            task.wait(0.25)
            if not isSearchOpen then SearchBox.Visible = false end
        end
    end

    Track(SearchToggleBtn.MouseButton1Click:Connect(function()
        setSearchOpen(not isSearchOpen)
    end))
    Track(SearchBox.FocusLost:Connect(function()
        TweenService:Create(SearchBox, TweenInfo.new(0.2), {BackgroundColor3 = Color3.fromRGB(12, 14, 19)}):Play()
        if SearchBox.Text == "" and isSearchOpen then
            setSearchOpen(false)
        end
    end))
    Track(SearchBox.Focused:Connect(function()
        TweenService:Create(SearchBox, TweenInfo.new(0.2), {BackgroundColor3 = Color3.fromRGB(20, 24, 34)}):Play()
    end))

    -- Open button
    local OpenBtnFrame = Instance.new("TextButton")
    OpenBtnFrame.Size = UDim2.new(0, 120, 0, 32); OpenBtnFrame.Position = UDim2.new(0.5, -60, 0, 8)
    OpenBtnFrame.BackgroundColor3 = Color3.fromRGB(22, 25, 34); OpenBtnFrame.BackgroundTransparency = 0.35
    OpenBtnFrame.Text = "⚡ Mở GUI"; OpenBtnFrame.TextColor3 = themeColor
    OpenBtnFrame.Font = Enum.Font.GothamBold; OpenBtnFrame.TextSize = 13
    OpenBtnFrame.Visible = false; OpenBtnFrame.Parent = ScreenGui
    local OBC = Instance.new("UICorner"); OBC.CornerRadius = UDim.new(0, 16); OBC.Parent = OpenBtnFrame
    if enableStroke then
        local OBS = Instance.new("UIStroke"); OBS.Color = themeColor; OBS.Transparency = 0.4; OBS.Thickness = 1; OBS.Parent = OpenBtnFrame
    end
    registerTheme(OpenBtnFrame, "TextColor3")

    task.spawn(function()
        while OpenBtnFrame.Parent do
            TweenService:Create(OpenBtnFrame, TweenInfo.new(0.7, Enum.EasingStyle.Sine, Enum.EasingDirection.InOut), {BackgroundTransparency = 0.55}):Play()
            task.wait(0.7)
            if not OpenBtnFrame.Parent then break end
            TweenService:Create(OpenBtnFrame, TweenInfo.new(0.7, Enum.EasingStyle.Sine, Enum.EasingDirection.InOut), {BackgroundTransparency = 0.25}):Play()
            task.wait(0.7)
        end
    end)

    Track(CloseBtn.MouseButton1Click:Connect(function()
        savedBlurState = Window:IsBlurEnabled()
        savedParticleState = Window:IsParticlesEnabled()
        Window:SetBlur(false)
        Window:SetParticles(false)
        TweenService:Create(mainScale, TweenInfo.new(0.25, Enum.EasingStyle.Back, Enum.EasingDirection.In), {Scale = 0}):Play()
        task.wait(0.25)
        MainFrame.Visible = false
        mainScale.Scale = 1
        OpenBtnFrame.Visible = true
    end))

    Track(OpenBtnFrame.MouseButton1Click:Connect(function()
        MainFrame.Visible = true
        OpenBtnFrame.Visible = false
        mainScale.Scale = 0.85
        TweenService:Create(mainScale, TweenInfo.new(0.4, Enum.EasingStyle.Back, Enum.EasingDirection.Out), {Scale = 1}):Play()
        if savedBlurState then Window:SetBlur(true) end
        if savedParticleState then Window:SetParticles(true) end
    end))

    local currentToggleKey = config.ToggleKey or Enum.KeyCode.RightControl
    Track(HelpBtn.MouseButton1Click:Connect(function()
        Window:Notify({
            Title = "Hướng Dẫn",
            Text = "• Kéo GUI bằng thanh trên cùng.\n• 🔒 Khóa GUI.\n• 🔍 Mở khung tìm kiếm.\n• 📜 Xem lịch sử thông báo.\n• RightControl ẩn/hiện.",
            Duration = 5,
        })
    end))
    Track(KeybindBtn.MouseButton1Click:Connect(function()
        Window:Notify({ Title = "Phím Tắt", Text = "Ẩn/hiện GUI: " .. tostring(currentToggleKey.Name), Duration = 4 })
    end))
    Track(LockBtn.MouseButton1Click:Connect(function()
        isLocked = not isLocked
        LockBtn.Text = isLocked and "🔒" or "🔓"
        LockBtn.TextColor3 = isLocked and themeColor or Color3.fromRGB(180, 185, 200)
        Window:Notify({
            Title = isLocked and "🔒 Đã Khóa" or "🔓 Đã Mở",
            Text = isLocked and "GUI sẽ không kéo được nữa." or "Có thể kéo GUI bình thường.",
            Duration = 1.5,
            Color = isLocked and "red" or "green",
        })
    end))

    Track(UserInputService.InputBegan:Connect(function(input, gpe)
        if not gpe and input.KeyCode == currentToggleKey then
            if MainFrame.Visible then
                savedBlurState = Window:IsBlurEnabled()
                savedParticleState = Window:IsParticlesEnabled()
                Window:SetBlur(false)
                Window:SetParticles(false)
                MainFrame.Visible = false
                OpenBtnFrame.Visible = true
            else
                MainFrame.Visible = true
                OpenBtnFrame.Visible = false
                if savedBlurState then Window:SetBlur(true) end
                if savedParticleState then Window:SetParticles(true) end
            end
        end
    end))

    -- ============ DRAG LOGIC (kéo bằng Topbar) ============
    local dragging, dragStart, startPos
    Track(Topbar.InputBegan:Connect(function(input)
        if isLocked then return end
        if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
            dragging = true
            dragStart = input.Position
            startPos = MainFrame.Position
        end
    end))
    Track(UserInputService.InputChanged:Connect(function(input)
        if isLocked then dragging = false; return end
        if dragging and (input.UserInputType == Enum.UserInputType.MouseMovement or input.UserInputType == Enum.UserInputType.Touch) then
            local delta = input.Position - dragStart
            MainFrame.Position = UDim2.new(startPos.X.Scale, startPos.X.Offset + delta.X, startPos.Y.Scale, startPos.Y.Offset + delta.Y)
        end
    end))
    Track(UserInputService.InputEnded:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then dragging = false end
    end))

    -- ============ SIDEBAR ============
    local Sidebar = Instance.new("ScrollingFrame")
    Sidebar.Size = UDim2.new(0, 160, 1, showPlaytime and -86 or -58)
    Sidebar.Position = UDim2.new(0, 10, 0, 54)
    Sidebar.BackgroundColor3 = Color3.fromRGB(19, 21, 28)
    Sidebar.BorderSizePixel = 0
    Sidebar.ScrollBarThickness = 2; Sidebar.ScrollBarImageColor3 = themeColor
    Sidebar.CanvasSize = UDim2.new(0, 0, 0, 0)
    Sidebar.Parent = MainFrame
    local SbC = Instance.new("UICorner"); SbC.CornerRadius = UDim.new(0, cornerRadius); SbC.Parent = Sidebar
    if enableStroke then
        local SbS = Instance.new("UIStroke"); SbS.Color = strokeColor; SbS.Thickness = 1; SbS.Parent = Sidebar
    end
    local SbPad = Instance.new("UIPadding")
    SbPad.PaddingTop = UDim.new(0, 6); SbPad.PaddingLeft = UDim.new(0, 3); SbPad.Parent = Sidebar

    local TabListLayout = Instance.new("UIListLayout")
    TabListLayout.Padding = UDim.new(0, 6); TabListLayout.SortOrder = Enum.SortOrder.LayoutOrder
    TabListLayout.Parent = Sidebar
    Track(TabListLayout:GetPropertyChangedSignal("AbsoluteContentSize"):Connect(function()
        Sidebar.CanvasSize = UDim2.new(0, 0, 0, TabListLayout.AbsoluteContentSize.Y + 12)
    end))

    if showPlaytime then
        local PlaytimeFrame = Instance.new("Frame")
        PlaytimeFrame.Size = UDim2.new(0, 160, 0, 24); PlaytimeFrame.Position = UDim2.new(0, 10, 1, -28)
        PlaytimeFrame.BackgroundColor3 = Color3.fromRGB(19, 21, 28); PlaytimeFrame.BorderSizePixel = 0
        PlaytimeFrame.Parent = MainFrame
        local PTC = Instance.new("UICorner"); PTC.CornerRadius = UDim.new(0, cornerRadius); PTC.Parent = PlaytimeFrame
        if enableStroke then
            local PTS = Instance.new("UIStroke"); PTS.Color = strokeColor; PTS.Thickness = 1; PTS.Parent = PlaytimeFrame
        end
        local PlaytimeLabel = Instance.new("TextLabel")
        PlaytimeLabel.Size = UDim2.new(1, 0, 1, 0); PlaytimeLabel.BackgroundTransparency = 1
        PlaytimeLabel.Text = "⏱️ 00:00:00"
        PlaytimeLabel.TextColor3 = Color3.fromRGB(150, 155, 170)
        PlaytimeLabel.Font = Enum.Font.Gotham; PlaytimeLabel.TextSize = 12
        PlaytimeLabel.Parent = PlaytimeFrame
        local startTime = os.time()
        task.spawn(function()
            while task.wait(1) do
                if not ScreenGui.Parent then break end
                local elapsed = os.time() - startTime
                local hrs = math.floor(elapsed / 3600)
                local mins = math.floor((elapsed % 3600) / 60)
                local secs = elapsed % 60
                PlaytimeLabel.Text = string.format("⏱️ %02d:%02d:%02d", hrs, mins, secs)
            end
        end)
    end

    local ContentArea = Instance.new("Frame")
    ContentArea.Size = UDim2.new(1, -188, 1, -62)
    ContentArea.Position = UDim2.new(0, 178, 0, 54)
    ContentArea.BackgroundTransparency = 1
    ContentArea.ClipsDescendants = true
    ContentArea.Parent = MainFrame

    local FirstTab = true

    Track(SearchBox:GetPropertyChangedSignal("Text"):Connect(function()
        local query = string.lower(SearchBox.Text)
        for _, container in pairs(ContentArea:GetChildren()) do
            if container:IsA("ScrollingFrame") and container.Visible then
                for _, elem in pairs(container:GetDescendants()) do
                    local searchText = elem:GetAttribute("SearchText")
                    if searchText then
                        elem.Visible = (query == "" or string.find(searchText, query, 1, true) ~= nil)
                    end
                end
            end
        end
    end))

    -- ============ NOTIFICATION SYSTEM (X + History) ============
    local notifHistory = {}
    local MAX_HISTORY = 50

    local NotifContainer = Instance.new("Frame")
    NotifContainer.Name = "NotifContainer"
    NotifContainer.Size = UDim2.new(0, 280, 1, -20)
    NotifContainer.Position = UDim2.new(1, -290, 0, 10)
    NotifContainer.BackgroundTransparency = 1
    NotifContainer.Parent = ScreenGui
    local NotifLayout = Instance.new("UIListLayout")
    NotifLayout.VerticalAlignment = Enum.VerticalAlignment.Bottom
    NotifLayout.Padding = UDim.new(0, 6)
    NotifLayout.SortOrder = Enum.SortOrder.LayoutOrder
    NotifLayout.Parent = NotifContainer

    local HistoryPanel = Instance.new("Frame")
    HistoryPanel.Name = "HistoryPanel"
    HistoryPanel.Size = UDim2.new(0, 320, 0, 420)
    HistoryPanel.Position = UDim2.new(0.5, -160, 0.5, -210)
    HistoryPanel.BackgroundColor3 = Color3.fromRGB(15, 17, 23)
    HistoryPanel.BorderSizePixel = 0
    HistoryPanel.Visible = false
    HistoryPanel.ZIndex = 50
    HistoryPanel.Parent = ScreenGui
    local HPC = Instance.new("UICorner"); HPC.CornerRadius = UDim.new(0, cornerRadius); HPC.Parent = HistoryPanel
    if enableStroke then
        local HPS = Instance.new("UIStroke"); HPS.Color = strokeColor; HPS.Thickness = 2; HPS.Parent = HistoryPanel
    end

    local HPTitle = Instance.new("TextLabel")
    HPTitle.Size = UDim2.new(1, -80, 0, 36)
    HPTitle.Position = UDim2.new(0, 10, 0, 6)
    HPTitle.BackgroundTransparency = 1
    HPTitle.Text = "📜 LỊCH SỬ THÔNG BÁO"
    HPTitle.TextColor3 = themeColor
    HPTitle.Font = Enum.Font.GothamBold
    HPTitle.TextSize = 14
    HPTitle.TextXAlignment = Enum.TextXAlignment.Left
    HPTitle.Parent = HistoryPanel
    registerTheme(HPTitle, "TextColor3")

    local HPClose = Instance.new("TextButton")
    HPClose.Size = UDim2.new(0, 30, 0, 30)
    HPClose.Position = UDim2.new(1, -38, 0, 9)
    HPClose.BackgroundTransparency = 1
    HPClose.Text = "✕"
    HPClose.TextColor3 = Color3.fromRGB(245, 70, 90)
    HPClose.TextSize = 16
    HPClose.Font = Enum.Font.GothamBold
    HPClose.Parent = HistoryPanel

    local HPScroll = Instance.new("ScrollingFrame")
    HPScroll.Size = UDim2.new(1, -20, 1, -56)
    HPScroll.Position = UDim2.new(0, 10, 0, 48)
    HPScroll.BackgroundTransparency = 1
    HPScroll.BorderSizePixel = 0
    HPScroll.ScrollBarThickness = 3
    HPScroll.ScrollBarImageColor3 = themeColor
    HPScroll.CanvasSize = UDim2.new(0, 0, 0, 0)
    HPScroll.Parent = HistoryPanel

    local HPScrollLayout = Instance.new("UIListLayout")
    HPScrollLayout.Padding = UDim.new(0, 6)
    HPScrollLayout.SortOrder = Enum.SortOrder.LayoutOrder
    HPScrollLayout.Parent = HPScroll
    Track(HPScrollLayout:GetPropertyChangedSignal("AbsoluteContentSize"):Connect(function()
        HPScroll.CanvasSize = UDim2.new(0, 0, 0, HPScrollLayout.AbsoluteContentSize.Y + 10)
    end))

    Track(HPClose.MouseButton1Click:Connect(function()
        HistoryPanel.Visible = false
    end))

    local function addToHistory(title, text, color)
        table.insert(notifHistory, 1, { title = title, text = text, color = color, time = os.date("%H:%M:%S") })
        while #notifHistory > MAX_HISTORY do
            table.remove(notifHistory)
        end

        for _, child in ipairs(HPScroll:GetChildren()) do
            if child:IsA("Frame") then child.LayoutOrder = child.LayoutOrder + 1 end
        end

        local Item = Instance.new("Frame")
        Item.Size = UDim2.new(1, -4, 0, 50)
        Item.BackgroundColor3 = Color3.fromRGB(22, 25, 34)
        Item.BorderSizePixel = 0
        Item.LayoutOrder = 0
        Item.Parent = HPScroll
        local IC = Instance.new("UICorner"); IC.CornerRadius = UDim.new(0, 8); IC.Parent = Item

        local Bar = Instance.new("Frame")
        Bar.Size = UDim2.new(0, 3, 1, 0)
        Bar.BackgroundColor3 = color
        Bar.BorderSizePixel = 0
        Bar.Parent = Item
        local BRC = Instance.new("UICorner"); BRC.CornerRadius = UDim.new(0, 8); BRC.Parent = Bar

        local T = Instance.new("TextLabel")
        T.Size = UDim2.new(1, -70, 0, 18)
        T.Position = UDim2.new(0, 10, 0, 4)
        T.BackgroundTransparency = 1
        T.Text = title
        T.TextColor3 = Color3.fromRGB(255, 255, 255)
        T.Font = Enum.Font.GothamBold
        T.TextSize = 12
        T.TextXAlignment = Enum.TextXAlignment.Left
        T.Parent = Item

        local Tm = Instance.new("TextLabel")
        Tm.Size = UDim2.new(0, 55, 0, 18)
        Tm.Position = UDim2.new(1, -58, 0, 4)
        Tm.BackgroundTransparency = 1
        Tm.Text = os.date("%H:%M:%S")
        Tm.TextColor3 = Color3.fromRGB(120, 125, 140)
        Tm.Font = Enum.Font.Gotham
        Tm.TextSize = 10
        Tm.TextXAlignment = Enum.TextXAlignment.Right
        Tm.Parent = Item

        local Tx = Instance.new("TextLabel")
        Tx.Size = UDim2.new(1, -20, 0, 24)
        Tx.Position = UDim2.new(0, 10, 0, 22)
        Tx.BackgroundTransparency = 1
        Tx.Text = (tostring(text):gsub("\n", " "))
        Tx.TextColor3 = Color3.fromRGB(180, 185, 200)
        Tx.Font = Enum.Font.Gotham
        Tx.TextSize = 11
        Tx.TextXAlignment = Enum.TextXAlignment.Left
        Tx.TextTruncate = Enum.TextTruncate.AtEnd
        Tx.Parent = Item
    end

    function Window:Notify(notifConfig)
        notifConfig = notifConfig or {}
        local title    = notifConfig.Title or "Thông Báo"
        local text     = notifConfig.Text or ""
        local duration = notifConfig.Duration or 3
        local nColor   = ParseColor(notifConfig.Color, Window.ThemeColor)
        local pos      = notifConfig.Position
        local fs       = notifConfig.FontSize or notifyFontSize
        local tfs      = notifConfig.TitleSize or notifyTitleSize

        addToHistory(title, text, nColor)

        local textSize = TextService:GetTextSize(text, fs, Enum.Font.Gotham, Vector2.new(240, 1000))
        local itemH = math.max(54, 30 + textSize.Y)

        local Item = Instance.new("Frame")
        Item.Size = UDim2.new(1, 0, 0, itemH)
        Item.BackgroundColor3 = Color3.fromRGB(22, 25, 34)
        Item.BorderSizePixel = 0
        Item.Parent = NotifContainer
        local ICC = Instance.new("UICorner"); ICC.CornerRadius = UDim.new(0, cornerRadius); ICC.Parent = Item
        if enableStroke then
            local IS = Instance.new("UIStroke"); IS.Color = strokeColor; IS.Thickness = 1; IS.Parent = Item
        end
        local nScale = Instance.new("UIScale"); nScale.Scale = 0.7; nScale.Parent = Item
        TweenService:Create(nScale, TweenInfo.new(0.35, Enum.EasingStyle.Back, Enum.EasingDirection.Out), {Scale = 1}):Play()

        local Bar = Instance.new("Frame")
        Bar.Size = UDim2.new(0, 4, 1, 0)
        Bar.BackgroundColor3 = nColor
        Bar.BorderSizePixel = 0
        Bar.Parent = Item
        local BC2 = Instance.new("UICorner"); BC2.CornerRadius = UDim.new(0, cornerRadius); BC2.Parent = Bar

        local NTitle = Instance.new("TextLabel")
        NTitle.Size = UDim2.new(1, -50, 0, 20)
        NTitle.Position = UDim2.new(0, 12, 0, 4)
        NTitle.BackgroundTransparency = 1
        NTitle.Text = title
        NTitle.TextColor3 = Color3.fromRGB(255, 255, 255)
        NTitle.Font = Enum.Font.GothamBold
        NTitle.TextSize = tfs
        NTitle.TextXAlignment = Enum.TextXAlignment.Left
        NTitle.Parent = Item

        local NText = Instance.new("TextLabel")
        NText.Size = UDim2.new(1, -30, 0, textSize.Y)
        NText.Position = UDim2.new(0, 12, 0, 26)
        NText.BackgroundTransparency = 1
        NText.Text = text
        NText.TextColor3 = Color3.fromRGB(160, 165, 180)
        NText.Font = Enum.Font.Gotham
        NText.TextSize = fs
        NText.TextWrapped = true
        NText.TextXAlignment = Enum.TextXAlignment.Left
        NText.TextYAlignment = Enum.TextYAlignment.Top
        NText.Parent = Item

        local CloseNotif = Instance.new("TextButton")
        CloseNotif.Size = UDim2.new(0, 24, 0, 24)
        CloseNotif.Position = UDim2.new(1, -30, 0, 4)
        CloseNotif.BackgroundTransparency = 1
        CloseNotif.Text = "✕"
        CloseNotif.TextColor3 = Color3.fromRGB(180, 185, 200)
        CloseNotif.TextSize = 13
        CloseNotif.Font = Enum.Font.GothamBold
        CloseNotif.Parent = Item

        local destroyed = false
        local function destroyNotif()
            if destroyed then return end
            destroyed = true
            TweenService:Create(nScale, TweenInfo.new(0.3, Enum.EasingStyle.Back, Enum.EasingDirection.In), {Scale = 0}):Play()
            TweenService:Create(Item, TweenInfo.new(0.3), {BackgroundTransparency = 1}):Play()
            TweenService:Create(NTitle, TweenInfo.new(0.3), {TextTransparency = 1}):Play()
            TweenService:Create(NText, TweenInfo.new(0.3), {TextTransparency = 1}):Play()
            TweenService:Create(Bar, TweenInfo.new(0.3), {BackgroundTransparency = 1}):Play()
            task.wait(0.3)
            Item:Destroy()
        end

        Track(CloseNotif.MouseButton1Click:Connect(destroyNotif))
        Track(CloseNotif.MouseEnter:Connect(function()
            CloseNotif.TextColor3 = Color3.fromRGB(245, 70, 90)
        end))
        Track(CloseNotif.MouseLeave:Connect(function()
            CloseNotif.TextColor3 = Color3.fromRGB(180, 185, 200)
        end))

        if pos == "left" then NotifContainer.Position = UDim2.new(0, 10, 0, 10)
        elseif pos == "center" then NotifContainer.Position = UDim2.new(0.5, -140, 0, 10)
        else NotifContainer.Position = UDim2.new(1, -290, 0, 10) end

        task.spawn(function()
            task.wait(duration)
            destroyNotif()
        end)
    end

    Track(HistoryBtn.MouseButton1Click:Connect(function()
        HistoryPanel.Visible = not HistoryPanel.Visible
        if HistoryPanel.Visible then HistoryPanel.ZIndex = 100 end
    end))

    -- ============ COPY/PASTE CONFIG ============
    local PastePanel = Instance.new("Frame")
    PastePanel.Size = UDim2.new(0, 400, 0, 200)
    PastePanel.Position = UDim2.new(0.5, -200, 0.5, -100)
    PastePanel.BackgroundColor3 = Color3.fromRGB(15, 17, 23)
    PastePanel.BorderSizePixel = 0
    PastePanel.Visible = false
    PastePanel.ZIndex = 60
    PastePanel.Parent = ScreenGui
    local PPC = Instance.new("UICorner"); PPC.CornerRadius = UDim.new(0, cornerRadius); PPC.Parent = PastePanel
    if enableStroke then
        local PPS = Instance.new("UIStroke"); PPS.Color = strokeColor; PPS.Thickness = 2; PPS.Parent = PastePanel
    end

    local PPTitle = Instance.new("TextLabel")
    PPTitle.Size = UDim2.new(1, -50, 0, 30)
    PPTitle.Position = UDim2.new(0, 14, 0, 8)
    PPTitle.BackgroundTransparency = 1
    PPTitle.Text = "📥 DÁN CONFIG"
    PPTitle.TextColor3 = themeColor
    PPTitle.Font = Enum.Font.GothamBold
    PPTitle.TextSize = 14
    PPTitle.TextXAlignment = Enum.TextXAlignment.Left
    PPTitle.Parent = PastePanel
    registerTheme(PPTitle, "TextColor3")

    local PPClose = Instance.new("TextButton")
    PPClose.Size = UDim2.new(0, 30, 0, 30)
    PPClose.Position = UDim2.new(1, -38, 0, 8)
    PPClose.BackgroundTransparency = 1
    PPClose.Text = "✕"
    PPClose.TextColor3 = Color3.fromRGB(245, 70, 90)
    PPClose.TextSize = 16
    PPClose.Font = Enum.Font.GothamBold
    PPClose.Parent = PastePanel

    local PPInput = Instance.new("TextBox")
    PPInput.Size = UDim2.new(1, -28, 0, 100)
    PPInput.Position = UDim2.new(0, 14, 0, 42)
    PPInput.BackgroundColor3 = Color3.fromRGB(24, 27, 36)
    PPInput.Text = ""
    PPInput.PlaceholderText = "Dán JSON config vào đây..."
    PPInput.TextColor3 = Color3.fromRGB(240, 240, 245)
    PPInput.PlaceholderColor3 = Color3.fromRGB(110, 115, 130)
    PPInput.Font = Enum.Font.Code
    PPInput.TextSize = 11
    PPInput.TextWrapped = true
    PPInput.TextXAlignment = Enum.TextXAlignment.Left
    PPInput.TextYAlignment = Enum.TextYAlignment.Top
    PPInput.ClearTextOnFocus = false
    PPInput.Parent = PastePanel
    local PPIC = Instance.new("UICorner"); PPIC.CornerRadius = UDim.new(0, 6); PPIC.Parent = PPInput

    local PPApply = Instance.new("TextButton")
    PPApply.Size = UDim2.new(1, -28, 0, 32)
    PPApply.Position = UDim2.new(0, 14, 0, 152)
    PPApply.BackgroundColor3 = themeColor
    PPApply.Text = "✓ Áp Dụng Config"
    PPApply.TextColor3 = Color3.fromRGB(255, 255, 255)
    PPApply.Font = Enum.Font.GothamBold
    PPApply.TextSize = 13
    PPApply.Parent = PastePanel
    local PPAC = Instance.new("UICorner"); PPAC.CornerRadius = UDim.new(0, 6); PPAC.Parent = PPApply
    registerTheme(PPApply, "BackgroundColor3")

    Track(PPClose.MouseButton1Click:Connect(function()
        PastePanel.Visible = false
    end))

    Track(PPApply.MouseButton1Click:Connect(function()
        local text = PPInput.Text
        if text == "" then
            Window:Notify({ Title = "Lỗi", Text = "Chưa dán config!", Color = "red" })
            return
        end

        local ok, data = pcall(function() return HttpService:JSONDecode(text) end)
        if not ok or type(data) ~= "table" then
            Window:Notify({ Title = "Lỗi", Text = "Config không hợp lệ (JSON sai)!", Color = "red" })
            return
        end

        local applied = 0
        for k, v in pairs(data) do
            if k ~= "_Presets" and k ~= "_SavedKey" then
                savedData[k] = v
                applied = applied + 1
            end
        end

        if applied > 0 then
            SaveCurrentConfig()
            for _, fn in ipairs(presetAppliers) do pcall(fn) end
            Window:Notify({
                Title = "✅ Thành Công",
                Text = "Đã áp dụng " .. applied .. " cài đặt!",
                Color = "green",
            })
            PastePanel.Visible = false
            PPInput.Text = ""
        else
            Window:Notify({ Title = "Lỗi", Text = "Config trống!", Color = "red" })
        end
    end))

    Track(CopyCfgBtn.MouseButton1Click:Connect(function()
        local snapshot = {}
        for k, v in pairs(savedData) do
            if k ~= "_SavedKey" then snapshot[k] = v end
        end
        local json = HttpService:JSONEncode(snapshot)
        CopyToClipboard(json)
        Window:Notify({
            Title = "📋 Đã Copy",
            Text = "Config đã được copy vào clipboard!",
            Color = "green",
        })
    end))

    Track(PasteCfgBtn.MouseButton1Click:Connect(function()
        PastePanel.Visible = true
        PastePanel.ZIndex = 100
        local cb = GetFromClipboard()
        if cb ~= "" and cb:sub(1,1) == "{" then
            PPInput.Text = cb
        end
    end))

    -- ============ TOOLTIP ============
    local TooltipFrame = Instance.new("Frame")
    TooltipFrame.Size = UDim2.new(0, 200, 0, 28)
    TooltipFrame.BackgroundColor3 = Color3.fromRGB(30, 34, 46)
    TooltipFrame.BorderSizePixel = 0
    TooltipFrame.Visible = false
    TooltipFrame.ZIndex = 100
    TooltipFrame.Parent = ScreenGui
    local TTC = Instance.new("UICorner"); TTC.CornerRadius = UDim.new(0, 6); TTC.Parent = TooltipFrame
    if enableStroke then
        local TTS = Instance.new("UIStroke"); TTS.Color = themeColor; TTS.Thickness = 1; TTS.Parent = TooltipFrame
        registerTheme(TTS, "Color")
    end
    local TooltipLabel = Instance.new("TextLabel")
    TooltipLabel.Size = UDim2.new(1, -12, 1, 0); TooltipLabel.Position = UDim2.new(0, 6, 0, 0)
    TooltipLabel.BackgroundTransparency = 1
    TooltipLabel.TextColor3 = Color3.fromRGB(240, 240, 245)
    TooltipLabel.Font = Enum.Font.Gotham; TooltipLabel.TextSize = 12
    TooltipLabel.TextXAlignment = Enum.TextXAlignment.Left
    TooltipLabel.ZIndex = 101
    TooltipLabel.Parent = TooltipFrame

    local function AddTooltip(widget, text)
        if not text or text == "" then return end
        Track(widget.MouseEnter:Connect(function()
            TooltipLabel.Text = text
            local w = math.clamp(#text * 7 + 20, 100, 260)
            TooltipFrame.Size = UDim2.new(0, w, 0, 28)
            TooltipFrame.Position = UDim2.new(0, widget.AbsolutePosition.X + widget.AbsoluteSize.X + 10, 0, widget.AbsolutePosition.Y + widget.AbsoluteSize.Y/2 - 14)
            TooltipFrame.Visible = true
        end))
        Track(widget.MouseLeave:Connect(function()
            TooltipFrame.Visible = false
        end))
    end

    -- ============ PRESET METHODS ============
    function Window:SavePreset(name)
        if not name or name == "" then return false end
        if not savedData["_Presets"] then savedData["_Presets"] = {} end
        local snapshot = {}
        for k, v in pairs(savedData) do
            if k ~= "_Presets" and k ~= "_SavedKey" then snapshot[k] = v end
        end
        savedData["_Presets"][name] = snapshot
        SaveCurrentConfig()
        return true
    end

    function Window:LoadPreset(name)
        if not savedData["_Presets"] or not savedData["_Presets"][name] then return false end
        local snapshot = savedData["_Presets"][name]
        for k, v in pairs(snapshot) do savedData[k] = v end
        SaveCurrentConfig()
        for _, fn in ipairs(presetAppliers) do pcall(fn) end
        return true
    end

    function Window:GetPresets()
        local list = {}
        if savedData["_Presets"] then
            for name, _ in pairs(savedData["_Presets"]) do table.insert(list, name) end
        end
        return list
    end

    function Window:DeletePreset(name)
        if savedData["_Presets"] and savedData["_Presets"][name] then
            savedData["_Presets"][name] = nil
            SaveCurrentConfig()
            return true
        end
        return false
    end

    -- ============ THEME SWITCHER ============
    function Window:SetTheme(newColor)
        local c = ParseColor(newColor, ColorPresets.blue)
        themeColor = c
        Window.ThemeColor = c
        Sidebar.ScrollBarImageColor3 = c
        for _, entry in ipairs(themeTargets) do
            pcall(function()
                TweenService:Create(entry.obj, TweenInfo.new(0.35, Enum.EasingStyle.Quart), {[entry.prop] = c}):Play()
            end)
        end
        TitleLabel.Text = hubTitle .. (hubVersion ~= "" and ("  <font color=\"" .. hexStringOf(c) .. "\">" .. hubVersion .. "</font>") or "")
        savedData["_SavedTheme"] = hexStringOf(c)
        SaveCurrentConfig()
    end

    function Window:GetTheme() return themeColor end

    -- ============ WIDGET BUILDER ============
    -- Mọi widget nhận tham số fontSize cuối cùng (8-40)
    local function buildWidgets(container)
        local W = {}

        local function getFontSize(param, default)
            if type(param) == "number" then
                return math.clamp(param, 8, 40)
            end
            return default
        end

        function W:CreateLabel(text, tooltip, fontSize)
            local fs = getFontSize(fontSize, 12)
            local L = Instance.new("TextLabel")
            L.Size = UDim2.new(1, -10, 0, math.max(28, fs + 16))
            L.BackgroundColor3 = Color3.fromRGB(22, 25, 34)
            L.Text = "  " .. text
            L.TextColor3 = Color3.fromRGB(170, 175, 190)
            L.TextXAlignment = Enum.TextXAlignment.Left
            L.Font = Enum.Font.Gotham
            L.TextSize = fs
            L.Parent = container
            local LC = Instance.new("UICorner"); LC.CornerRadius = UDim.new(0, cornerRadius); LC.Parent = L
            if enableStroke then
                local LS = Instance.new("UIStroke"); LS.Color = strokeColor; LS.Thickness = 1; LS.Parent = L
            end
            L:SetAttribute("SearchText", string.lower(text))
            AddTooltip(L, tooltip or text)
            return L
        end

        function W:CreateParagraph(text, fontSize)
            local fs = getFontSize(fontSize, 13)
            local Wrap = Instance.new("Frame")
            Wrap.Size = UDim2.new(1, -10, 0, 0)
            Wrap.AutomaticSize = Enum.AutomaticSize.Y
            Wrap.BackgroundColor3 = Color3.fromRGB(22, 25, 34)
            Wrap.BorderSizePixel = 0
            Wrap.Parent = container
            local WC = Instance.new("UICorner"); WC.CornerRadius = UDim.new(0, cornerRadius); WC.Parent = Wrap
            if enableStroke then
                local WS = Instance.new("UIStroke"); WS.Color = strokeColor; WS.Thickness = 1; WS.Parent = Wrap
            end
            local Pad = Instance.new("UIPadding")
            Pad.PaddingTop = UDim.new(0, 8); Pad.PaddingBottom = UDim.new(0, 8)
            Pad.PaddingLeft = UDim.new(0, 12); Pad.PaddingRight = UDim.new(0, 12)
            Pad.Parent = Wrap
            local TL = Instance.new("TextLabel")
            TL.Size = UDim2.new(1, 0, 0, 0)
            TL.AutomaticSize = Enum.AutomaticSize.Y
            TL.BackgroundTransparency = 1
            TL.Text = text
            TL.TextColor3 = Color3.fromRGB(190, 195, 210)
            TL.Font = Enum.Font.Gotham
            TL.TextSize = fs
            TL.TextXAlignment = Enum.TextXAlignment.Left
            TL.TextYAlignment = Enum.TextYAlignment.Top
            TL.TextWrapped = true
            TL.Parent = Wrap
            Wrap:SetAttribute("SearchText", string.lower(text))
            return Wrap
        end

        function W:CreateDivider()
            local Div = Instance.new("Frame")
            Div.Size = UDim2.new(1, -20, 0, 1)
            Div.BackgroundColor3 = Color3.fromRGB(45, 50, 65)
            Div.BorderSizePixel = 0
            Div.Parent = container
            return Div
        end

        function W:CreateSection(text, fontSize)
            local fs = getFontSize(fontSize, 12)
            local SF = Instance.new("Frame")
            SF.Size = UDim2.new(1, -10, 0, math.max(24, fs + 12))
            SF.BackgroundTransparency = 1
            SF.Parent = container
            local ST = Instance.new("TextLabel")
            ST.Size = UDim2.new(1, 0, 1, 0); ST.BackgroundTransparency = 1
            ST.Text = "───  " .. string.upper(text or "SECTION") .. "  ───"
            ST.TextColor3 = themeColor
            ST.Font = Enum.Font.GothamBold
            ST.TextSize = fs
            ST.TextTransparency = 1
            ST.Parent = SF
            registerTheme(ST, "TextColor3")
            TweenService:Create(ST, TweenInfo.new(0.5, Enum.EasingStyle.Quart), {TextTransparency = 0}):Play()
            SF:SetAttribute("SearchText", string.lower(text or ""))
            return SF
        end

        -- CreateButton(text, callback, tooltip, fontSize)
        function W:CreateButton(btnText, callback, tooltip, fontSize)
            callback = callback or function() end
            local fs = getFontSize(fontSize, 13)
            local B = Instance.new("TextButton")
            B.Size = UDim2.new(1, -10, 0, math.max(38, fs + 22))
            B.BackgroundColor3 = Color3.fromRGB(24, 28, 38)
            B.Text = btnText; B.TextColor3 = Color3.fromRGB(240, 240, 245)
            B.Font = Enum.Font.Gotham
            B.TextSize = fs
            B.AutoButtonColor = false
            B.Parent = container
            local BC3 = Instance.new("UICorner"); BC3.CornerRadius = UDim.new(0, cornerRadius); BC3.Parent = B
            if enableStroke then
                local BS = Instance.new("UIStroke"); BS.Color = strokeColor; BS.Thickness = 1; BS.Parent = B
            end
            B:SetAttribute("SearchText", string.lower(btnText))
            AttachButtonEffects(B, Track, Color3.fromRGB(34, 40, 56))
            AddTooltip(B, tooltip)
            Track(B.MouseButton1Click:Connect(function()
                CreateRipple(B)
                TweenService:Create(B, TweenInfo.new(0.1), {BackgroundColor3 = Window.ThemeColor}):Play()
                task.wait(0.1)
                TweenService:Create(B, TweenInfo.new(0.15), {BackgroundColor3 = Color3.fromRGB(24, 28, 38)}):Play()
                callback()
            end))
            return B
        end

        -- CreateToggle(text, flag, default, callback, tooltip, fontSize)
        function W:CreateToggle(toggleText, flagName, defaultState, callback, tooltip, fontSize)
            callback = callback or function() end
            local fs = getFontSize(fontSize, 13)
            if flagName and savedData[flagName] ~= nil then defaultState = savedData[flagName] end
            local state = defaultState or false
            local T = Instance.new("TextButton")
            T.Size = UDim2.new(1, -10, 0, math.max(38, fs + 22))
            T.BackgroundColor3 = Color3.fromRGB(24, 28, 38)
            T.Text = ""
            T.AutoButtonColor = false
            T.Parent = container
            local TFC = Instance.new("UICorner"); TFC.CornerRadius = UDim.new(0, cornerRadius); TFC.Parent = T
            if enableStroke then
                local TFS = Instance.new("UIStroke"); TFS.Color = strokeColor; TFS.Thickness = 1; TFS.Parent = T
            end
            T:SetAttribute("SearchText", string.lower(toggleText))
            AddTooltip(T, tooltip or toggleText)
            local Lbl = Instance.new("TextLabel")
            Lbl.Size = UDim2.new(1, -55, 1, 0); Lbl.Position = UDim2.new(0, 14, 0, 0)
            Lbl.BackgroundTransparency = 1; Lbl.Text = toggleText
            Lbl.TextColor3 = Color3.fromRGB(240, 240, 245)
            Lbl.TextXAlignment = Enum.TextXAlignment.Left
            Lbl.Font = Enum.Font.Gotham
            Lbl.TextSize = fs
            Lbl.Parent = T
            local Tr = Instance.new("Frame")
            Tr.Size = UDim2.new(0, 42, 0, 22); Tr.Position = UDim2.new(1, -52, 0.5, -11)
            Tr.BackgroundColor3 = state and themeColor or Color3.fromRGB(50, 56, 72)
            Tr.BorderSizePixel = 0; Tr.Parent = T
            local TFC2 = Instance.new("UICorner"); TFC2.CornerRadius = UDim.new(1, 0); TFC2.Parent = Tr
            local K = Instance.new("Frame")
            K.Size = UDim2.new(0, 18, 0, 18)
            K.Position = state and UDim2.new(1, -20, 0.5, -9) or UDim2.new(0, 2, 0.5, -9)
            K.BackgroundColor3 = Color3.fromRGB(255, 255, 255)
            K.BorderSizePixel = 0; K.Parent = Tr
            local KC2 = Instance.new("UICorner"); KC2.CornerRadius = UDim.new(1, 0); KC2.Parent = K
            local function applyToggle(ns)
                state = ns
                TweenService:Create(Tr, TweenInfo.new(0.3, Enum.EasingStyle.Back, Enum.EasingDirection.Out), {BackgroundColor3 = state and Window.ThemeColor or Color3.fromRGB(50, 56, 72)}):Play()
                TweenService:Create(K, TweenInfo.new(0.3, Enum.EasingStyle.Back, Enum.EasingDirection.Out), {Position = state and UDim2.new(1, -20, 0.5, -9) or UDim2.new(0, 2, 0.5, -9)}):Play()
            end
            Track(T.MouseButton1Click:Connect(function()
                applyToggle(not state)
                if flagName then savedData[flagName] = state; SaveCurrentConfig() end
                callback(state)
            end))
            if flagName then
                table.insert(presetAppliers, function()
                    if savedData[flagName] ~= nil and savedData[flagName] ~= state then
                        applyToggle(savedData[flagName]); callback(savedData[flagName])
                    end
                end)
            end
            if defaultState then callback(true) end
            local obj = {}
            function obj:Set(v) applyToggle(v and true or false); if flagName then savedData[flagName] = state; SaveCurrentConfig() end; callback(state) end
            function obj:Get() return state end
            return obj
        end

        -- CreateSlider(text, flag, min, max, default, callback, showInput, tooltip, fontSize)
        function W:CreateSlider(sliderText, flagName, minVal, maxVal, defaultVal, callback, showInput, tooltip, fontSize)
            callback = callback or function() end
            local fs = getFontSize(fontSize, 13)
            minVal = minVal or 0; maxVal = maxVal or 1000
            if flagName and savedData[flagName] ~= nil then defaultVal = savedData[flagName] end
            defaultVal = math.clamp(defaultVal or minVal, minVal, maxVal)
            showInput = showInput or false
            local range = math.max(maxVal - minVal, 1)
            local SF = Instance.new("Frame")
            SF.Size = UDim2.new(1, -10, 0, math.max(48, fs + 35))
            SF.BackgroundColor3 = Color3.fromRGB(24, 28, 38)
            SF.Parent = container
            local SFC = Instance.new("UICorner"); SFC.CornerRadius = UDim.new(0, cornerRadius); SFC.Parent = SF
            if enableStroke then
                local SFS = Instance.new("UIStroke"); SFS.Color = strokeColor; SFS.Thickness = 1; SFS.Parent = SF
            end
            SF:SetAttribute("SearchText", string.lower(sliderText))
            AddTooltip(SF, tooltip or sliderText)
            local Lbl = Instance.new("TextLabel")
            Lbl.Size = UDim2.new(1, -80, 0, 22); Lbl.Position = UDim2.new(0, 14, 0, 3)
            Lbl.BackgroundTransparency = 1; Lbl.Text = sliderText
            Lbl.TextColor3 = Color3.fromRGB(240, 240, 245)
            Lbl.TextXAlignment = Enum.TextXAlignment.Left
            Lbl.Font = Enum.Font.Gotham
            Lbl.TextSize = fs
            Lbl.Parent = SF
            local VB = Instance.new("TextBox")
            VB.Size = UDim2.new(0, 60, 0, 22); VB.Position = UDim2.new(1, -68, 0, 3)
            VB.BackgroundColor3 = showInput and Color3.fromRGB(15, 17, 23) or Color3.fromRGB(24, 28, 38)
            VB.BackgroundTransparency = showInput and 0 or 1
            VB.Text = tostring(defaultVal)
            VB.TextColor3 = themeColor
            VB.Font = Enum.Font.GothamBold
            VB.TextSize = fs - 1
            VB.TextEditable = showInput
            VB.ClearTextOnFocus = false
            VB.Name = "InputBox"
            VB.Parent = SF
            local VBC = Instance.new("UICorner"); VBC.CornerRadius = UDim.new(0, 6); VBC.Parent = VB
            registerTheme(VB, "TextColor3")
            local VStr = nil
            if showInput and enableStroke then
                VStr = Instance.new("UIStroke"); VStr.Color = strokeColor; VStr.Thickness = 1; VStr.Parent = VB
            end
            local Bar = Instance.new("TextButton")
            Bar.Size = UDim2.new(1, -28, 0, 7); Bar.Position = UDim2.new(0, 14, 0, fs + 18)
            Bar.BackgroundColor3 = Color3.fromRGB(50, 56, 72)
            Bar.Text = ""; Bar.AutoButtonColor = false
            Bar.Name = "SliderBar"
            Bar.Parent = SF
            local SBC = Instance.new("UICorner"); SBC.CornerRadius = UDim.new(1, 0); SBC.Parent = Bar
            local Fill = Instance.new("Frame")
            local initPct = (defaultVal - minVal) / range
            Fill.Size = UDim2.new(initPct, 0, 1, 0)
            Fill.BackgroundColor3 = themeColor
            Fill.BorderSizePixel = 0; Fill.Parent = Bar
            local FBC = Instance.new("UICorner"); FBC.CornerRadius = UDim.new(1, 0); FBC.Parent = Fill
            registerTheme(Fill, "BackgroundColor3")
            local Thumb = Instance.new("Frame")
            Thumb.Size = UDim2.new(0, 16, 0, 16)
            Thumb.Position = UDim2.new(initPct, -8, 0.5, -8)
            Thumb.BackgroundColor3 = Color3.fromRGB(255, 255, 255)
            Thumb.BorderSizePixel = 0; Thumb.ZIndex = 3
            Thumb.Parent = Bar
            local THC = Instance.new("UICorner"); THC.CornerRadius = UDim.new(1, 0); THC.Parent = Thumb
            local ThStr = Instance.new("UIStroke"); ThStr.Color = themeColor; ThStr.Thickness = 2; ThStr.Parent = Thumb
            registerTheme(ThStr, "Color")
            local currentValue = defaultVal
            local sliding = false
            local isInputVisible = showInput
            local function applyValue(val)
                currentValue = val
                local pct = (val - minVal) / range
                Fill.Size = UDim2.new(pct, 0, 1, 0)
                Thumb.Position = UDim2.new(pct, -8, 0.5, -8)
                VB.Text = tostring(val)
            end
            local function updateSlider(input)
                local pct = math.clamp((input.Position.X - Bar.AbsolutePosition.X) / Bar.AbsoluteSize.X, 0, 1)
                local val = math.floor(minVal + (maxVal - minVal) * pct + 0.5)
                applyValue(val)
                if flagName then savedData[flagName] = val; SaveCurrentConfig() end
                callback(val)
            end
            Track(Bar.InputBegan:Connect(function(input)
                if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
                    sliding = true; updateSlider(input)
                end
            end))
            Track(UserInputService.InputChanged:Connect(function(input)
                if sliding and (input.UserInputType == Enum.UserInputType.MouseMovement or input.UserInputType == Enum.UserInputType.Touch) then
                    updateSlider(input)
                end
            end))
            Track(UserInputService.InputEnded:Connect(function(input)
                if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then sliding = false end
            end))
            Track(VB.FocusLost:Connect(function()
                if isInputVisible then
                    local num = tonumber(VB.Text)
                    if num == nil then VB.Text = tostring(currentValue); return end
                    num = math.clamp(math.floor(num), minVal, maxVal)
                    applyValue(num)
                    if flagName then savedData[flagName] = num; SaveCurrentConfig() end
                    callback(num)
                end
            end))
            if flagName then
                table.insert(presetAppliers, function()
                    if savedData[flagName] ~= nil then applyValue(savedData[flagName]); callback(savedData[flagName]) end
                end)
            end
            local obj = {}
            function obj:Set(val)
                val = math.clamp(math.floor(val), minVal, maxVal)
                applyValue(val)
                if flagName then savedData[flagName] = val; SaveCurrentConfig() end
                callback(val)
            end
            function obj:Get() return currentValue end
            function obj:SetInputVisible(b)
                isInputVisible = b and true or false
                VB.TextEditable = isInputVisible
                VB.BackgroundTransparency = isInputVisible and 0 or 1
                if isInputVisible then
                    VB.BackgroundColor3 = Color3.fromRGB(15, 17, 23)
                    if not VStr and enableStroke then
                        VStr = Instance.new("UIStroke"); VStr.Color = strokeColor; VStr.Thickness = 1; VStr.Parent = VB
                    elseif VStr then VStr.Enabled = true end
                else
                    if VStr then VStr.Enabled = false end
                end
            end
            function obj:IsInputVisible() return isInputVisible end
            function obj:GetMin() return minVal end
            function obj:GetMax() return maxVal end
            return obj
        end

        -- CreateDropdown(text, options, default, callback, tooltip, fontSize)
        function W:CreateDropdown(dropText, options, defaultOption, callback, tooltip, fontSize)
            callback = callback or function() end
            local fs = getFontSize(fontSize, 13)
            options = options or {}
            local currentChoice = defaultOption or options[1] or "Chưa chọn"
            local MAX_HEIGHT = 150
            local headerH = math.max(38, fs + 22)
            local DF = Instance.new("Frame")
            DF.Size = UDim2.new(1, -10, 0, headerH)
            DF.BackgroundColor3 = Color3.fromRGB(24, 28, 38)
            DF.ClipsDescendants = true
            DF.Parent = container
            local DFC = Instance.new("UICorner"); DFC.CornerRadius = UDim.new(0, cornerRadius); DFC.Parent = DF
            if enableStroke then
                local DFS = Instance.new("UIStroke"); DFS.Color = strokeColor; DFS.Thickness = 1; DFS.Parent = DF
            end
            DF:SetAttribute("SearchText", string.lower(dropText))
            AddTooltip(DF, tooltip or dropText)
            local HB = Instance.new("TextButton")
            HB.Size = UDim2.new(1, 0, 0, headerH); HB.BackgroundTransparency = 1
            HB.Text = ""; HB.Parent = DF
            local Lbl = Instance.new("TextLabel")
            Lbl.Size = UDim2.new(0.5, 0, 1, 0); Lbl.Position = UDim2.new(0, 14, 0, 0)
            Lbl.BackgroundTransparency = 1; Lbl.Text = dropText
            Lbl.TextColor3 = Color3.fromRGB(240, 240, 245)
            Lbl.TextXAlignment = Enum.TextXAlignment.Left
            Lbl.Font = Enum.Font.Gotham
            Lbl.TextSize = fs
            Lbl.Parent = HB
            local SL = Instance.new("TextLabel")
            SL.Size = UDim2.new(0.5, -20, 1, 0); SL.Position = UDim2.new(0.5, -5, 0, 0)
            SL.BackgroundTransparency = 1
            SL.Text = currentChoice .. " ▼"
            SL.TextColor3 = themeColor
            SL.TextXAlignment = Enum.TextXAlignment.Right
            SL.Font = Enum.Font.GothamBold
            SL.TextSize = fs - 1
            SL.Parent = HB
            registerTheme(SL, "TextColor3")
            local listH = math.min(#options * (fs + 18), MAX_HEIGHT)
            local OC = Instance.new("ScrollingFrame")
            OC.Size = UDim2.new(1, -16, 0, listH)
            OC.Position = UDim2.new(0, 8, 0, headerH + 4)
            OC.BackgroundTransparency = 1
            OC.BorderSizePixel = 0
            OC.ScrollBarThickness = 2; OC.ScrollBarImageColor3 = themeColor
            OC.CanvasSize = UDim2.new(0, 0, 0, #options * (fs + 18))
            OC.Parent = DF
            local OL = Instance.new("UIListLayout"); OL.Padding = UDim.new(0, 4); OL.Parent = OC
            local isExp = false
            Track(HB.MouseButton1Click:Connect(function()
                isExp = not isExp
                SL.Text = currentChoice .. (isExp and " ▲" or " ▼")
                local h = headerH + 8 + listH
                DF:TweenSize(UDim2.new(1, -10, 0, isExp and h or headerH), Enum.EasingDirection.Out, Enum.EasingStyle.Quart, 0.25, true)
            end))
            for _, opt in ipairs(options) do
                local OB = Instance.new("TextButton")
                OB.Size = UDim2.new(1, 0, 0, fs + 14)
                OB.BackgroundColor3 = Color3.fromRGB(18, 20, 26)
                OB.Text = opt
                OB.TextColor3 = Color3.fromRGB(200, 205, 220)
                OB.Font = Enum.Font.Gotham
                OB.TextSize = fs - 1
                OB.Name = "DropdownOption"
                OB.Parent = OC
                local OBC = Instance.new("UICorner"); OBC.CornerRadius = UDim.new(0, cornerRadius - 2); OBC.Parent = OB
                Track(OB.MouseEnter:Connect(function()
                    TweenService:Create(OB, TweenInfo.new(0.15), {BackgroundColor3 = Color3.fromRGB(28, 32, 42)}):Play()
                end))
                Track(OB.MouseLeave:Connect(function()
                    TweenService:Create(OB, TweenInfo.new(0.15), {BackgroundColor3 = Color3.fromRGB(18, 20, 26)}):Play()
                end))
                Track(OB.MouseButton1Click:Connect(function()
                    currentChoice = opt
                    SL.Text = currentChoice .. " ▼"
                    isExp = false
                    DF:TweenSize(UDim2.new(1, -10, 0, headerH), Enum.EasingDirection.Out, Enum.EasingStyle.Quart, 0.2, true)
                    callback(opt)
                end))
            end
            local obj = {}
            function obj:Get() return currentChoice end
            function obj:Set(v) currentChoice = v; SL.Text = v .. " ▼"; callback(v) end
            return obj
        end

        -- CreateMultiDropdown(text, options, defaultSelected, callback, tooltip, fontSize)
        function W:CreateMultiDropdown(dropText, options, defaultSelected, callback, tooltip, fontSize)
            callback = callback or function() end
            local fs = getFontSize(fontSize, 13)
            options = options or {}
            defaultSelected = defaultSelected or {}
            local MAX_HEIGHT = 150
            local selected = {}
            for _, o in ipairs(defaultSelected) do selected[o] = true end
            local function getList()
                local l = {}
                for _, o in ipairs(options) do if selected[o] then table.insert(l, o) end end
                return l
            end
            local function summary()
                local l = getList()
                if #l == 0 then return "Chưa chọn ▼" end
                if #l <= 2 then return table.concat(l, ", ") .. " ▼" end
                return l[1] .. " +" .. (#l - 1) .. " ▼"
            end
            local headerH = math.max(38, fs + 22)
            local DF = Instance.new("Frame")
            DF.Size = UDim2.new(1, -10, 0, headerH)
            DF.BackgroundColor3 = Color3.fromRGB(24, 28, 38)
            DF.ClipsDescendants = true
            DF.Parent = container
            local DFC = Instance.new("UICorner"); DFC.CornerRadius = UDim.new(0, cornerRadius); DFC.Parent = DF
            if enableStroke then
                local DFS = Instance.new("UIStroke"); DFS.Color = strokeColor; DFS.Thickness = 1; DFS.Parent = DF
            end
            DF:SetAttribute("SearchText", string.lower(dropText))
            AddTooltip(DF, tooltip or dropText)
            local HB = Instance.new("TextButton")
            HB.Size = UDim2.new(1, 0, 0, headerH); HB.BackgroundTransparency = 1
            HB.Text = ""; HB.Parent = DF
            local Lbl = Instance.new("TextLabel")
            Lbl.Size = UDim2.new(0.4, 0, 1, 0); Lbl.Position = UDim2.new(0, 14, 0, 0)
            Lbl.BackgroundTransparency = 1; Lbl.Text = dropText
            Lbl.TextColor3 = Color3.fromRGB(240, 240, 245)
            Lbl.TextXAlignment = Enum.TextXAlignment.Left
            Lbl.Font = Enum.Font.Gotham
            Lbl.TextSize = fs
            Lbl.Parent = HB
            local SL = Instance.new("TextLabel")
            SL.Size = UDim2.new(0.6, -20, 1, 0); SL.Position = UDim2.new(0.4, -5, 0, 0)
            SL.BackgroundTransparency = 1
            SL.Text = summary()
            SL.TextColor3 = themeColor
            SL.TextXAlignment = Enum.TextXAlignment.Right
            SL.Font = Enum.Font.GothamBold
            SL.TextSize = fs - 1
            SL.Parent = HB
            registerTheme(SL, "TextColor3")
            local listH = math.min(#options * (fs + 16), MAX_HEIGHT)
            local OC = Instance.new("ScrollingFrame")
            OC.Size = UDim2.new(1, -16, 0, listH)
            OC.Position = UDim2.new(0, 8, 0, headerH + 4)
            OC.BackgroundTransparency = 1
            OC.BorderSizePixel = 0
            OC.ScrollBarThickness = 2; OC.ScrollBarImageColor3 = themeColor
            OC.CanvasSize = UDim2.new(0, 0, 0, #options * (fs + 16) + 10)
            OC.Parent = DF
            local OL = Instance.new("UIListLayout"); OL.Padding = UDim.new(0, 3); OL.Parent = OC
            local isExp = false
            Track(HB.MouseButton1Click:Connect(function()
                isExp = not isExp
                SL.Text = summary()
                local h = headerH + 8 + listH + 10
                DF:TweenSize(UDim2.new(1, -10, 0, isExp and h or headerH), Enum.EasingDirection.Out, Enum.EasingStyle.Quart, 0.25, true)
            end))
            local rows = {}
            for _, opt in ipairs(options) do
                local OB = Instance.new("TextButton")
                OB.Size = UDim2.new(1, 0, 0, fs + 12)
                OB.BackgroundColor3 = selected[opt] and Color3.fromRGB(30, 36, 48) or Color3.fromRGB(18, 20, 26)
                OB.Text = ""; OB.AutoButtonColor = false
                OB.Name = "MultiDropdownOption"
                OB.Parent = OC
                local OBC = Instance.new("UICorner"); OBC.CornerRadius = UDim.new(0, cornerRadius - 2); OBC.Parent = OB
                local CB = Instance.new("Frame")
                CB.Size = UDim2.new(0, 16, 0, 16); CB.Position = UDim2.new(0, 8, 0.5, -8)
                CB.BackgroundColor3 = selected[opt] and themeColor or Color3.fromRGB(50, 56, 72)
                CB.BorderSizePixel = 0; CB.Parent = OB
                local CBC = Instance.new("UICorner"); CBC.CornerRadius = UDim.new(0, 4); CBC.Parent = CB
                local CM = Instance.new("TextLabel")
                CM.Size = UDim2.new(1, 0, 1, 0); CM.BackgroundTransparency = 1
                CM.Text = selected[opt] and "✓" or ""
                CM.TextColor3 = Color3.fromRGB(255, 255, 255)
                CM.Font = Enum.Font.GothamBold; CM.TextSize = fs - 2
                CM.Parent = CB
                local OL2 = Instance.new("TextLabel")
                OL2.Size = UDim2.new(1, -40, 1, 0); OL2.Position = UDim2.new(0, 30, 0, 0)
                OL2.BackgroundTransparency = 1; OL2.Text = opt
                OL2.TextColor3 = Color3.fromRGB(200, 205, 220)
                OL2.TextXAlignment = Enum.TextXAlignment.Left
                OL2.Font = Enum.Font.Gotham
                OL2.TextSize = fs - 1
                OL2.Parent = OB
                rows[opt] = function()
                    local isOn = selected[opt]
                    OB.BackgroundColor3 = isOn and Color3.fromRGB(30, 36, 48) or Color3.fromRGB(18, 20, 26)
                    CB.BackgroundColor3 = isOn and Window.ThemeColor or Color3.fromRGB(50, 56, 72)
                    CM.Text = isOn and "✓" or ""
                end
                Track(OB.MouseEnter:Connect(function()
                    TweenService:Create(OB, TweenInfo.new(0.15), {BackgroundColor3 = Color3.fromRGB(28, 32, 42)}):Play()
                end))
                Track(OB.MouseLeave:Connect(function()
                    TweenService:Create(OB, TweenInfo.new(0.15), {BackgroundColor3 = selected[opt] and Color3.fromRGB(30, 36, 48) or Color3.fromRGB(18, 20, 26)}):Play()
                end))
                Track(OB.MouseButton1Click:Connect(function()
                    selected[opt] = not selected[opt]
                    rows[opt]()
                    SL.Text = summary()
                    callback(getList())
                end))
            end
            local obj = {}
            function obj:Get() return getList() end
            function obj:Set(list)
                selected = {}
                for _, o in ipairs(list or {}) do selected[o] = true end
                for _, fn in pairs(rows) do fn() end
                SL.Text = summary(); callback(getList())
            end
            function obj:Clear()
                selected = {}
                for _, fn in pairs(rows) do fn() end
                SL.Text = summary(); callback(getList())
            end
            return obj
        end

        -- CreateColorPicker(text, flag, defaultColor, callback, tooltip, fontSize)
        function W:CreateColorPicker(pickerText, flagName, defaultColor, callback, tooltip, fontSize)
            callback = callback or function() end
            local fs = getFontSize(fontSize, 13)
            local currentColor = defaultColor or Color3.fromRGB(255, 255, 255)
            local function loadSaved()
                local t = flagName and savedData[flagName]
                if type(t) == "table" and tonumber(t[1]) and tonumber(t[2]) and tonumber(t[3]) then
                    return Color3.new(math.clamp(t[1], 0, 1), math.clamp(t[2], 0, 1), math.clamp(t[3], 0, 1))
                end
                return nil
            end
            currentColor = loadSaved() or currentColor
            local rowH = math.max(38, fs + 22)
            local PF = Instance.new("Frame")
            PF.Size = UDim2.new(1, -10, 0, rowH)
            PF.BackgroundColor3 = Color3.fromRGB(24, 28, 38)
            PF.Parent = container
            local PFC = Instance.new("UICorner"); PFC.CornerRadius = UDim.new(0, cornerRadius); PFC.Parent = PF
            if enableStroke then
                local PFS = Instance.new("UIStroke"); PFS.Color = strokeColor; PFS.Thickness = 1; PFS.Parent = PF
            end
            PF:SetAttribute("SearchText", string.lower(pickerText))
            AddTooltip(PF, tooltip or pickerText)
            local Lbl = Instance.new("TextLabel")
            Lbl.Size = UDim2.new(0.5, -10, 1, 0); Lbl.Position = UDim2.new(0, 14, 0, 0)
            Lbl.BackgroundTransparency = 1; Lbl.Text = pickerText
            Lbl.TextColor3 = Color3.fromRGB(240, 240, 245)
            Lbl.TextXAlignment = Enum.TextXAlignment.Left
            Lbl.Font = Enum.Font.Gotham
            Lbl.TextSize = fs
            Lbl.Parent = PF
            local btnH = math.min(24, rowH - 8)
            local CBtn = Instance.new("TextButton")
            CBtn.Size = UDim2.new(0, 60, 0, btnH); CBtn.Position = UDim2.new(1, -68, 0.5, -(btnH / 2))
            CBtn.BackgroundColor3 = currentColor
            CBtn.Text = ""; CBtn.Parent = PF
            local CBC = Instance.new("UICorner"); CBC.CornerRadius = UDim.new(0, 6); CBC.Parent = CBtn
            if enableStroke then
                local CBS = Instance.new("UIStroke"); CBS.Color = strokeColor; CBS.Thickness = 1; CBS.Parent = CBtn
            end
            local Panel = Instance.new("Frame")
            Panel.Size = UDim2.new(1, -10, 0, 0)
            Panel.BackgroundColor3 = Color3.fromRGB(20, 23, 32)
            Panel.BorderSizePixel = 0
            Panel.ClipsDescendants = true
            Panel.Visible = false
            Panel.Parent = container
            local PC2 = Instance.new("UICorner"); PC2.CornerRadius = UDim.new(0, cornerRadius); PC2.Parent = Panel
            if enableStroke then
                local PS = Instance.new("UIStroke"); PS.Color = strokeColor; PS.Thickness = 1; PS.Parent = Panel
            end
            local Preview = Instance.new("Frame")
            Preview.Size = UDim2.new(1, -20, 0, 30); Preview.Position = UDim2.new(0, 10, 0, 10)
            Preview.BackgroundColor3 = currentColor; Preview.BorderSizePixel = 0
            Preview.Parent = Panel
            local PVC = Instance.new("UICorner"); PVC.CornerRadius = UDim.new(0, 6); PVC.Parent = Preview
            local rVal, gVal, bVal = currentColor.R, currentColor.G, currentColor.B
            local Hex
            local sliderUpdaters = {}
            local function refresh()
                currentColor = Color3.new(rVal, gVal, bVal)
                Preview.BackgroundColor3 = currentColor
                CBtn.BackgroundColor3 = currentColor
                if Hex then Hex.Text = hexStringOf(currentColor) end
                if flagName then savedData[flagName] = {currentColor.R, currentColor.G, currentColor.B}; SaveCurrentConfig() end
                callback(currentColor)
            end
            local function mkSlider(label, yOff, getV, setV)
                local Row = Instance.new("Frame")
                Row.Size = UDim2.new(1, -20, 0, 26); Row.Position = UDim2.new(0, 10, 0, yOff)
                Row.BackgroundTransparency = 1; Row.Parent = Panel
                local Lb = Instance.new("TextLabel")
                Lb.Size = UDim2.new(0, 40, 1, 0); Lb.BackgroundTransparency = 1
                Lb.Text = label; Lb.TextColor3 = Color3.fromRGB(200, 205, 220)
                Lb.TextXAlignment = Enum.TextXAlignment.Left
                Lb.Font = Enum.Font.Gotham; Lb.TextSize = fs - 1
                Lb.Parent = Row
                local Bar = Instance.new("TextButton")
                Bar.Size = UDim2.new(1, -90, 0, 8); Bar.Position = UDim2.new(0, 45, 0.5, -4)
                Bar.BackgroundColor3 = Color3.fromRGB(50, 56, 72)
                Bar.Text = ""; Bar.AutoButtonColor = false
                Bar.Name = "ColorPickerBar"
                Bar.Parent = Row
                local BC3 = Instance.new("UICorner"); BC3.CornerRadius = UDim.new(1, 0); BC3.Parent = Bar
                local Fill = Instance.new("Frame")
                Fill.Size = UDim2.new(getV(), 0, 1, 0)
                Fill.BackgroundColor3 = Color3.fromRGB(255, 255, 255)
                Fill.BackgroundTransparency = 0.4
                Fill.BorderSizePixel = 0; Fill.Parent = Bar
                local FC2 = Instance.new("UICorner"); FC2.CornerRadius = UDim.new(1, 0); FC2.Parent = Fill
                local VL = Instance.new("TextLabel")
                VL.Size = UDim2.new(0, 40, 1, 0); VL.Position = UDim2.new(1, -40, 0, 0)
                VL.BackgroundTransparency = 1
                VL.Text = tostring(math.floor(getV() * 255 + 0.5))
                VL.TextColor3 = themeColor
                VL.TextXAlignment = Enum.TextXAlignment.Right
                VL.Font = Enum.Font.GothamBold; VL.TextSize = fs - 2
                VL.Parent = Row
                registerTheme(VL, "TextColor3")
                table.insert(sliderUpdaters, function()
                    Fill.Size = UDim2.new(getV(), 0, 1, 0)
                    VL.Text = tostring(math.floor(getV() * 255 + 0.5))
                end)
                local dr = false
                local function upd(input)
                    local pct = math.clamp((input.Position.X - Bar.AbsolutePosition.X) / Bar.AbsoluteSize.X, 0, 1)
                    Fill.Size = UDim2.new(pct, 0, 1, 0)
                    VL.Text = tostring(math.floor(pct * 255 + 0.5))
                    setV(pct)
                end
                Track(Bar.InputBegan:Connect(function(input)
                    if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then dr = true; upd(input) end
                end))
                Track(UserInputService.InputChanged:Connect(function(input)
                    if dr and (input.UserInputType == Enum.UserInputType.MouseMovement or input.UserInputType == Enum.UserInputType.Touch) then upd(input) end
                end))
                Track(UserInputService.InputEnded:Connect(function(input)
                    if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then dr = false end
                end))
            end
            mkSlider("R", 48, function() return rVal end, function(v) rVal = v; refresh() end)
            mkSlider("G", 78, function() return gVal end, function(v) gVal = v; refresh() end)
            mkSlider("B", 108, function() return bVal end, function(v) bVal = v; refresh() end)
            Hex = Instance.new("TextBox")
            Hex.Size = UDim2.new(1, -20, 0, 26); Hex.Position = UDim2.new(0, 10, 0, 138)
            Hex.BackgroundColor3 = Color3.fromRGB(15, 17, 23)
            Hex.Text = hexStringOf(currentColor)
            Hex.TextColor3 = Color3.fromRGB(240, 240, 245)
            Hex.PlaceholderText = "#RRGGBB"
            Hex.Font = Enum.Font.Gotham; Hex.TextSize = fs - 1
            Hex.Name = "InputBox"
            Hex.Parent = Panel
            local HBC = Instance.new("UICorner"); HBC.CornerRadius = UDim.new(0, 6); HBC.Parent = Hex
            local function setAll(c)
                rVal, gVal, bVal = c.R, c.G, c.B
                for _, fn in ipairs(sliderUpdaters) do fn() end
                refresh()
            end
            Track(Hex.FocusLost:Connect(function()
                local h = Hex.Text
                if h:sub(1,1) ~= "#" then h = "#" .. h end
                local ok, c = pcall(function() return Color3.fromHex(h) end)
                if ok and c then
                    setAll(c)
                else
                    Hex.Text = hexStringOf(currentColor)
                end
            end))
            local isOpen = false
            Track(CBtn.MouseButton1Click:Connect(function()
                isOpen = not isOpen
                if isOpen then Panel.Visible = true end
                local h = isOpen and 176 or 0
                Panel:TweenSize(UDim2.new(1, -10, 0, h), Enum.EasingDirection.Out, Enum.EasingStyle.Quart, 0.25, true)
                if not isOpen then task.delay(0.25, function() if not isOpen then Panel.Visible = false end end) end
            end))
            if flagName then
                table.insert(presetAppliers, function()
                    local c = loadSaved()
                    if c then
                        currentColor = c
                        rVal, gVal, bVal = c.R, c.G, c.B
                        Preview.BackgroundColor3 = c
                        CBtn.BackgroundColor3 = c
                        Hex.Text = hexStringOf(c)
                        for _, fn in ipairs(sliderUpdaters) do fn() end
                        callback(c)
                    end
                end)
            end
            local obj = {}
            function obj:Get() return currentColor end
            function obj:Set(c) setAll(c) end
            return obj
        end

        -- CreateTextbox(text, flag, maxChars, callback, tooltip, fontSize)
        function W:CreateTextbox(boxText, flagName, maxChars, callback, tooltip, fontSize)
            callback = callback or function() end
            local fs = getFontSize(fontSize, 13)
            local defaultVal = ""
            if flagName and savedData[flagName] ~= nil then defaultVal = tostring(savedData[flagName]) end
            local rowH = math.max(38, fs + 22)
            local BF = Instance.new("Frame")
            BF.Size = UDim2.new(1, -10, 0, rowH)
            BF.BackgroundColor3 = Color3.fromRGB(24, 28, 38)
            BF.Parent = container
            local BFC = Instance.new("UICorner"); BFC.CornerRadius = UDim.new(0, cornerRadius); BFC.Parent = BF
            if enableStroke then
                local BFS = Instance.new("UIStroke"); BFS.Color = strokeColor; BFS.Thickness = 1; BFS.Parent = BF
            end
            BF:SetAttribute("SearchText", string.lower(boxText))
            AddTooltip(BF, tooltip or boxText)
            local Lbl = Instance.new("TextLabel")
            Lbl.Size = UDim2.new(0.5, -10, 1, 0); Lbl.Position = UDim2.new(0, 14, 0, 0)
            Lbl.BackgroundTransparency = 1; Lbl.Text = boxText
            Lbl.TextColor3 = Color3.fromRGB(240, 240, 245)
            Lbl.TextXAlignment = Enum.TextXAlignment.Left
            Lbl.Font = Enum.Font.Gotham
            Lbl.TextSize = fs
            Lbl.Parent = BF
            local inputH = math.min(26, rowH - 10)
            local IB = Instance.new("TextBox")
            IB.Size = UDim2.new(0.5, -10, 0, inputH); IB.Position = UDim2.new(0.5, 0, 0.5, -(inputH/2))
            IB.BackgroundColor3 = Color3.fromRGB(15, 17, 23)
            IB.Text = defaultVal
            IB.PlaceholderText = maxChars and ("Max " .. maxChars .. " chữ") or "Nhập..."
            IB.TextColor3 = Color3.fromRGB(255, 255, 255)
            IB.PlaceholderColor3 = Color3.fromRGB(120, 125, 140)
            IB.Font = Enum.Font.Gotham
            IB.TextSize = fs - 1
            IB.ClearTextOnFocus = false
            IB.Name = "InputBox"
            IB.Parent = BF
            local IBC = Instance.new("UICorner"); IBC.CornerRadius = UDim.new(0, cornerRadius - 2); IBC.Parent = IB
            Track(IB.Focused:Connect(function()
                TweenService:Create(IB, TweenInfo.new(0.2), {BackgroundColor3 = Color3.fromRGB(22, 26, 36)}):Play()
            end))
            Track(IB.FocusLost:Connect(function(enterPressed)
                TweenService:Create(IB, TweenInfo.new(0.2), {BackgroundColor3 = Color3.fromRGB(15, 17, 23)}):Play()
                if flagName then savedData[flagName] = IB.Text; SaveCurrentConfig() end
                callback(IB.Text, enterPressed)
            end))
            if maxChars then
                Track(IB:GetPropertyChangedSignal("Text"):Connect(function()
                    if #IB.Text > maxChars then IB.Text = string.sub(IB.Text, 1, maxChars) end
                end))
            end
            if flagName then
                table.insert(presetAppliers, function()
                    if savedData[flagName] ~= nil then IB.Text = tostring(savedData[flagName]); callback(IB.Text, false) end
                end)
            end
            local obj = {}
            function obj:Get() return IB.Text end
            function obj:Set(t) IB.Text = tostring(t) end
            return obj
        end

        -- CreateKeybind(text, flag, defaultKey, callback, tooltip, fontSize)
        function W:CreateKeybind(labelText, flagName, defaultKey, callback, tooltip, fontSize)
            callback = callback or function() end
            local fs = getFontSize(fontSize, 13)
            local currentKey = defaultKey or Enum.KeyCode.Unknown
            if flagName and savedData[flagName] then
                pcall(function() currentKey = Enum.KeyCode[savedData[flagName]] or currentKey end)
            end
            local listening = false
            local rowH = math.max(38, fs + 22)
            local BF = Instance.new("Frame")
            BF.Size = UDim2.new(1, -10, 0, rowH)
            BF.BackgroundColor3 = Color3.fromRGB(24, 28, 38)
            BF.Parent = container
            local BKC = Instance.new("UICorner"); BKC.CornerRadius = UDim.new(0, cornerRadius); BKC.Parent = BF
            if enableStroke then
                local BKS = Instance.new("UIStroke"); BKS.Color = strokeColor; BKS.Thickness = 1; BKS.Parent = BF
            end
            BF:SetAttribute("SearchText", string.lower(labelText))
            AddTooltip(BF, tooltip or labelText)
            local Lbl = Instance.new("TextLabel")
            Lbl.Size = UDim2.new(0.5, -10, 1, 0); Lbl.Position = UDim2.new(0, 14, 0, 0)
            Lbl.BackgroundTransparency = 1; Lbl.Text = labelText
            Lbl.TextColor3 = Color3.fromRGB(240, 240, 245)
            Lbl.TextXAlignment = Enum.TextXAlignment.Left
            Lbl.Font = Enum.Font.Gotham
            Lbl.TextSize = fs
            Lbl.Parent = BF
            local kbH = math.min(26, rowH - 10)
            local KB = Instance.new("TextButton")
            KB.Size = UDim2.new(0, 100, 0, kbH); KB.Position = UDim2.new(1, -110, 0.5, -(kbH/2))
            KB.BackgroundColor3 = Color3.fromRGB(15, 17, 23)
            KB.Text = currentKey.Name
            KB.TextColor3 = themeColor
            KB.Font = Enum.Font.GothamBold; KB.TextSize = fs - 1
            KB.Parent = BF
            local KBC = Instance.new("UICorner"); KBC.CornerRadius = UDim.new(0, cornerRadius - 2); KBC.Parent = KB
            registerTheme(KB, "TextColor3")
            Track(KB.MouseButton1Click:Connect(function()
                listening = true
                KB.Text = "Nhấn phím..."
                KB.TextColor3 = Color3.fromRGB(255, 180, 80)
            end))
            Track(UserInputService.InputBegan:Connect(function(input, gpe)
                if not listening then return end
                if input.UserInputType == Enum.UserInputType.Keyboard then
                    currentKey = input.KeyCode
                    listening = false
                    KB.Text = currentKey.Name
                    KB.TextColor3 = Window.ThemeColor
                    if flagName then savedData[flagName] = currentKey.Name; SaveCurrentConfig() end
                    callback(currentKey)
                end
            end))
            if flagName then
                table.insert(presetAppliers, function()
                    if savedData[flagName] then
                        pcall(function()
                            currentKey = Enum.KeyCode[savedData[flagName]]
                            KB.Text = currentKey.Name
                            callback(currentKey)
                        end)
                    end
                end)
            end
            return {
                GetKey = function() return currentKey end,
                SetKey = function(k)
                    currentKey = k
                    KB.Text = k.Name
                    if flagName then savedData[flagName] = k.Name; SaveCurrentConfig() end
                    callback(k)
                end,
            }
        end

        -- CreateProgressBar(text, initialPercent, tooltip, fontSize)
        function W:CreateProgressBar(labelText, initialPercent, tooltip, fontSize)
            local fs = getFontSize(fontSize, 13)
            local pct = math.clamp(initialPercent or 0, 0, 100)
            local rowH = math.max(46, fs + 33)
            local PF = Instance.new("Frame")
            PF.Size = UDim2.new(1, -10, 0, rowH)
            PF.BackgroundColor3 = Color3.fromRGB(24, 28, 38)
            PF.Parent = container
            local PFC = Instance.new("UICorner"); PFC.CornerRadius = UDim.new(0, cornerRadius); PFC.Parent = PF
            if enableStroke then
                local PFS = Instance.new("UIStroke"); PFS.Color = strokeColor; PFS.Thickness = 1; PFS.Parent = PF
            end
            PF:SetAttribute("SearchText", string.lower(labelText))
            AddTooltip(PF, tooltip or labelText)
            local Lbl = Instance.new("TextLabel")
            Lbl.Size = UDim2.new(0.7, 0, 0, 22); Lbl.Position = UDim2.new(0, 14, 0, 3)
            Lbl.BackgroundTransparency = 1; Lbl.Text = labelText
            Lbl.TextColor3 = Color3.fromRGB(240, 240, 245)
            Lbl.TextXAlignment = Enum.TextXAlignment.Left
            Lbl.Font = Enum.Font.Gotham
            Lbl.TextSize = fs
            Lbl.Parent = PF
            local VL = Instance.new("TextLabel")
            VL.Size = UDim2.new(0.3, -14, 0, 22); VL.Position = UDim2.new(0.7, 0, 0, 3)
            VL.BackgroundTransparency = 1; VL.Text = math.floor(pct) .. "%"
            VL.TextColor3 = themeColor
            VL.TextXAlignment = Enum.TextXAlignment.Right
            VL.Font = Enum.Font.GothamBold; VL.TextSize = fs
            VL.Parent = PF
            registerTheme(VL, "TextColor3")
            local barY = math.max(30, fs + 17)
            local BG = Instance.new("Frame")
            BG.Size = UDim2.new(1, -28, 0, 8); BG.Position = UDim2.new(0, 14, 0, barY)
            BG.BackgroundColor3 = Color3.fromRGB(50, 56, 72)
            BG.BorderSizePixel = 0; BG.Parent = PF
            local BBC = Instance.new("UICorner"); BBC.CornerRadius = UDim.new(1, 0); BBC.Parent = BG
            local Fill = Instance.new("Frame")
            Fill.Size = UDim2.new(pct/100, 0, 1, 0)
            Fill.BackgroundColor3 = themeColor
            Fill.BorderSizePixel = 0; Fill.Parent = BG
            local BFC = Instance.new("UICorner"); BFC.CornerRadius = UDim.new(1, 0); BFC.Parent = Fill
            registerTheme(Fill, "BackgroundColor3")
            local obj = {}
            function obj:Set(p)
                pct = math.clamp(p, 0, 100)
                TweenService:Create(Fill, TweenInfo.new(0.35, Enum.EasingStyle.Quart), {Size = UDim2.new(pct/100, 0, 1, 0)}):Play()
                VL.Text = math.floor(pct) .. "%"
            end
            function obj:Get() return pct end
            return obj
        end

        -- CreateImage(imageId, height)
        function W:CreateImage(imageId, height)
            if not imageId then return nil end
            local h = height or 100
            local IF = Instance.new("Frame")
            IF.Size = UDim2.new(1, -10, 0, h)
            IF.BackgroundColor3 = Color3.fromRGB(24, 28, 38)
            IF.BorderSizePixel = 0
            IF.ClipsDescendants = true
            IF.Parent = container
            local IFC = Instance.new("UICorner"); IFC.CornerRadius = UDim.new(0, cornerRadius); IFC.Parent = IF
            if enableStroke then
                local IFS = Instance.new("UIStroke"); IFS.Color = strokeColor; IFS.Thickness = 1; IFS.Parent = IF
            end
            local Img = Instance.new("ImageLabel")
            Img.Size = UDim2.new(1, -16, 1, -16)
            Img.Position = UDim2.new(0, 8, 0, 8)
            Img.BackgroundTransparency = 1
            Img.Image = typeof(imageId) == "number" and ("rbxassetid://" .. imageId) or imageId
            Img.ScaleType = Enum.ScaleType.Fit
            Img.Parent = IF
            IF:SetAttribute("SearchText", "image ảnh logo")
            return IF
        end

        return W
    end

    -- ============ CREATE TAB ============
    function Window:CreateTab(tabName, iconEmoji)
        local displayText = iconEmoji and (iconEmoji .. "  " .. tabName) or ("  " .. tabName)
        local TB = Instance.new("TextButton")
        TB.Size = UDim2.new(1, -6, 0, 36)
        TB.BackgroundColor3 = FirstTab and Color3.fromRGB(28, 33, 46) or Color3.fromRGB(19, 21, 28)
        TB.Text = displayText
        TB.TextColor3 = FirstTab and Color3.fromRGB(255, 255, 255) or Color3.fromRGB(140, 145, 160)
        TB.TextXAlignment = Enum.TextXAlignment.Left
        TB.Font = Enum.Font.GothamBold; TB.TextSize = 13
        TB.AutoButtonColor = false
        TB.Parent = Sidebar
        local TBPad = Instance.new("UIPadding"); TBPad.PaddingLeft = UDim.new(0, 8); TBPad.Parent = TB
        local TBC = Instance.new("UICorner"); TBC.CornerRadius = UDim.new(0, cornerRadius); TBC.Parent = TB
        if enableStroke then
            local TBS = Instance.new("UIStroke"); TBS.Color = strokeColor; TBS.Thickness = 1; TBS.Parent = TB
        end
        local TabC = Instance.new("ScrollingFrame")
        TabC.Size = UDim2.new(1, 0, 1, 0)
        TabC.BackgroundTransparency = 1
        TabC.BorderSizePixel = 0
        TabC.ScrollBarThickness = 3; TabC.ScrollBarImageColor3 = themeColor
        TabC.CanvasSize = UDim2.new(0, 0, 0, 0)
        TabC.Visible = FirstTab
        TabC.Parent = ContentArea
        local CL = Instance.new("UIListLayout")
        CL.Padding = UDim.new(0, 10); CL.SortOrder = Enum.SortOrder.LayoutOrder
        CL.Parent = TabC
        Track(CL:GetPropertyChangedSignal("AbsoluteContentSize"):Connect(function()
            TabC.CanvasSize = UDim2.new(0, 0, 0, CL.AbsoluteContentSize.Y + 15)
        end))
        Track(TB.MouseEnter:Connect(function()
            if not TabC.Visible then
                TweenService:Create(TB, TweenInfo.new(0.15), {BackgroundColor3 = Color3.fromRGB(24, 28, 40)}):Play()
            end
        end))
        Track(TB.MouseLeave:Connect(function()
            if not TabC.Visible then
                TweenService:Create(TB, TweenInfo.new(0.15), {BackgroundColor3 = Color3.fromRGB(19, 21, 28)}):Play()
            end
        end))
        Track(TB.MouseButton1Click:Connect(function()
            for _, child in pairs(ContentArea:GetChildren()) do
                if child:IsA("ScrollingFrame") then child.Visible = false end
            end
            for _, btn in pairs(Sidebar:GetChildren()) do
                if btn:IsA("TextButton") then
                    btn.BackgroundColor3 = Color3.fromRGB(19, 21, 28)
                    btn.TextColor3 = Color3.fromRGB(140, 145, 160)
                end
            end
            TabC.Visible = true
            TB.BackgroundColor3 = Color3.fromRGB(28, 33, 46)
            TB.TextColor3 = Color3.fromRGB(255, 255, 255)
            TabC.Position = UDim2.new(0.06, 0, 0, 0)
            TweenService:Create(TabC, TweenInfo.new(0.25, Enum.EasingStyle.Quart, Enum.EasingDirection.Out), {Position = UDim2.new(0, 0, 0, 0)}):Play()
        end))
        FirstTab = false
        local Tab = buildWidgets(TabC)
        function Tab:CreateSubTab(subName)
            local SB = Instance.new("Frame")
            SB.Size = UDim2.new(1, -10, 0, 32)
            SB.BackgroundColor3 = Color3.fromRGB(19, 21, 28)
            SB.BorderSizePixel = 0
            SB.Parent = TabC
            local SBC = Instance.new("UICorner"); SBC.CornerRadius = UDim.new(0, cornerRadius); SBC.Parent = SB
            if enableStroke then
                local SBS = Instance.new("UIStroke"); SBS.Color = strokeColor; SBS.Thickness = 1; SBS.Parent = SB
            end
            local SBL = Instance.new("UIListLayout")
            SBL.FillDirection = Enum.FillDirection.Horizontal
            SBL.Padding = UDim.new(0, 6)
            SBL.VerticalAlignment = Enum.VerticalAlignment.Center
            SBL.SortOrder = Enum.SortOrder.LayoutOrder
            SBL.Parent = SB
            local Pad = Instance.new("UIPadding")
            Pad.PaddingLeft = UDim.new(0, 6); Pad.PaddingRight = UDim.new(0, 6)
            Pad.Parent = SB
            local SubC = Instance.new("Frame")
            SubC.Size = UDim2.new(1, -10, 0, 0)
            SubC.AutomaticSize = Enum.AutomaticSize.Y
            SubC.BackgroundTransparency = 1
            SubC.Parent = TabC
            local SL = Instance.new("UIListLayout")
            SL.Padding = UDim.new(0, 10); SL.SortOrder = Enum.SortOrder.LayoutOrder
            SL.Parent = SubC
            local SBtn = Instance.new("TextButton")
            SBtn.Size = UDim2.new(0, 100, 0, 24)
            SBtn.BackgroundColor3 = themeColor
            SBtn.Text = subName
            SBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
            SBtn.Font = Enum.Font.GothamBold; SBtn.TextSize = 12
            SBtn.Parent = SB
            local SBtnC = Instance.new("UICorner"); SBtnC.CornerRadius = UDim.new(0, 6); SBtnC.Parent = SBtn
            registerTheme(SBtn, "BackgroundColor3")
            local widgets = buildWidgets(SubC)
            return widgets, SBtn, SubC
        end
        return Tab
    end

    -- ============ MINI WINDOW ============
    function Window:CreateMiniWindow(mwConfig)
        mwConfig = mwConfig or {}
        local mwX = mwConfig.X or 20
        local mwY = mwConfig.Y or 100
        local mwWidth = mwConfig.Width or 150
        local mwHeight = mwConfig.Height or 220
        local mwResizable = (mwConfig.Resizable == nil) and true or mwConfig.Resizable
        local mwDraggable = (mwConfig.Draggable == nil) and true or mwConfig.Draggable
        local mwMinW = mwConfig.MinWidth or 120
        local mwMaxW = mwConfig.MaxWidth or 400
        local mwMinH = mwConfig.MinHeight or 80
        local mwMaxH = mwConfig.MaxHeight or 500
        local mwShowDragBar = (mwConfig.ShowDragBar == nil) and true or mwConfig.ShowDragBar

        local MiniFrame = Instance.new("Frame")
        MiniFrame.Name = "MiniWindow"
        MiniFrame.Size = UDim2.new(0, mwWidth, 0, mwHeight)
        MiniFrame.Position = UDim2.new(0, mwX, 0, mwY)
        MiniFrame.BackgroundTransparency = 1
        MiniFrame.BorderSizePixel = 0
        MiniFrame.ClipsDescendants = false
        MiniFrame.Parent = ScreenGui

        local Container = Instance.new("ScrollingFrame")
        Container.Size = UDim2.new(1, 0, 1, mwShowDragBar and -20 or 0)
        Container.Position = UDim2.new(0, 0, 0, mwShowDragBar and 20 or 0)
        Container.BackgroundTransparency = 1
        Container.BorderSizePixel = 0
        Container.ScrollBarThickness = 2
        Container.ScrollBarImageColor3 = themeColor
        Container.CanvasSize = UDim2.new(0, 0, 0, 0)
        Container.Parent = MiniFrame

        local Layout = Instance.new("UIListLayout")
        Layout.Padding = UDim.new(0, 4)
        Layout.SortOrder = Enum.SortOrder.LayoutOrder
        Layout.Parent = Container
        Track(Layout:GetPropertyChangedSignal("AbsoluteContentSize"):Connect(function()
            Container.CanvasSize = UDim2.new(0, 0, 0, Layout.AbsoluteContentSize.Y + 10)
        end))

        local DragBar = Instance.new("TextButton")
        DragBar.Size = UDim2.new(1, 0, 0, 18)
        DragBar.Position = UDim2.new(0, 0, 0, 0)
        DragBar.BackgroundColor3 = Color3.fromRGB(25, 28, 38)
        DragBar.BackgroundTransparency = 0.25
        DragBar.Text = "✥  Kéo"
        DragBar.TextColor3 = Color3.fromRGB(180, 185, 200)
        DragBar.TextSize = 11
        DragBar.Font = Enum.Font.Gotham
        DragBar.AutoButtonColor = false
        DragBar.Visible = mwShowDragBar
        DragBar.Parent = MiniFrame
        local DBC = Instance.new("UICorner"); DBC.CornerRadius = UDim.new(0, 6); DBC.Parent = DragBar

        local ResizeHandle = Instance.new("TextButton")
        ResizeHandle.Size = UDim2.new(0, 16, 0, 16)
        ResizeHandle.Position = UDim2.new(1, -16, 1, -16)
        ResizeHandle.BackgroundColor3 = themeColor
        ResizeHandle.BackgroundTransparency = 0.3
        ResizeHandle.Text = "◢"
        ResizeHandle.TextColor3 = Color3.fromRGB(255, 255, 255)
        ResizeHandle.TextSize = 12
        ResizeHandle.Font = Enum.Font.GothamBold
        ResizeHandle.AutoButtonColor = false
        ResizeHandle.Visible = mwResizable
        ResizeHandle.ZIndex = 5
        ResizeHandle.Parent = MiniFrame
        local RHC = Instance.new("UICorner"); RHC.CornerRadius = UDim.new(0, 4); RHC.Parent = ResizeHandle
        registerTheme(ResizeHandle, "BackgroundColor3")

        local dg, ds, sp
        Track(DragBar.InputBegan:Connect(function(input)
            if not mwDraggable then return end
            if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
                dg = true; ds = input.Position; sp = MiniFrame.Position
            end
        end))
        Track(UserInputService.InputChanged:Connect(function(input)
            if dg and (input.UserInputType == Enum.UserInputType.MouseMovement or input.UserInputType == Enum.UserInputType.Touch) then
                local delta = input.Position - ds
                MiniFrame.Position = UDim2.new(0, sp.X.Offset + delta.X, 0, sp.Y.Offset + delta.Y)
            end
        end))
        Track(UserInputService.InputEnded:Connect(function(input)
            if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then dg = false end
        end))

        local rg, rs, rsize
        Track(ResizeHandle.InputBegan:Connect(function(input)
            if not mwResizable then return end
            if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
                rg = true; rs = input.Position; rsize = MiniFrame.AbsoluteSize
            end
        end))
        Track(UserInputService.InputChanged:Connect(function(input)
            if rg and (input.UserInputType == Enum.UserInputType.MouseMovement or input.UserInputType == Enum.UserInputType.Touch) then
                local delta = input.Position - rs
                local nW = math.clamp(rsize.X + delta.X, mwMinW, mwMaxW)
                local nH = math.clamp(rsize.Y + delta.Y, mwMinH, mwMaxH)
                MiniFrame.Size = UDim2.new(0, nW, 0, nH)
            end
        end))
        Track(UserInputService.InputEnded:Connect(function(input)
            if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then rg = false end
        end))

        local MW = {}

        function MW:CreateLabel(text, tooltip, fontSize)
            local fs = fontSize or 12
            local L = Instance.new("TextLabel")
            L.Size = UDim2.new(1, -4, 0, math.max(22, fs + 12))
            L.BackgroundColor3 = Color3.fromRGB(24, 28, 38)
            L.BackgroundTransparency = 0.15
            L.Text = text
            L.TextColor3 = Color3.fromRGB(240, 240, 245)
            L.Font = Enum.Font.Gotham
            L.TextSize = fs
            L.Parent = Container
            local LC2 = Instance.new("UICorner"); LC2.CornerRadius = UDim.new(0, 6); LC2.Parent = L
            AddTooltip(L, tooltip or text)
            return L
        end

        function MW:CreateButton(text, callback, tooltip, fontSize)
            callback = callback or function() end
            local fs = fontSize or 12
            local B = Instance.new("TextButton")
            B.Size = UDim2.new(1, -4, 0, math.max(26, fs + 14))
            B.BackgroundColor3 = themeColor
            B.BackgroundTransparency = 0.1
            B.Text = text
            B.TextColor3 = Color3.fromRGB(255, 255, 255)
            B.Font = Enum.Font.GothamBold
            B.TextSize = fs
            B.Parent = Container
            local BC2 = Instance.new("UICorner"); BC2.CornerRadius = UDim.new(0, 6); BC2.Parent = B
            AddTooltip(B, tooltip)
            registerTheme(B, "BackgroundColor3")
            Track(B.MouseButton1Click:Connect(function() callback() end))
            return B
        end

        function MW:CreateToggle(text, flagName, defaultState, callback, tooltip, fontSize)
            callback = callback or function() end
            local fs = fontSize or 11
            if flagName and savedData[flagName] ~= nil then defaultState = savedData[flagName] end
            local state = defaultState or false
            local T = Instance.new("TextButton")
            T.Size = UDim2.new(1, -4, 0, math.max(26, fs + 14))
            T.BackgroundColor3 = Color3.fromRGB(24, 28, 38)
            T.BackgroundTransparency = 0.15
            T.Text = ""
            T.AutoButtonColor = false
            T.Parent = Container
            local TCc = Instance.new("UICorner"); TCc.CornerRadius = UDim.new(0, 6); TCc.Parent = T
            AddTooltip(T, tooltip or text)
            local Lbl = Instance.new("TextLabel")
            Lbl.Size = UDim2.new(1, -40, 1, 0)
            Lbl.Position = UDim2.new(0, 8, 0, 0)
            Lbl.BackgroundTransparency = 1
            Lbl.Text = text
            Lbl.TextColor3 = Color3.fromRGB(240, 240, 245)
            Lbl.TextXAlignment = Enum.TextXAlignment.Left
            Lbl.Font = Enum.Font.Gotham
            Lbl.TextSize = fs
            Lbl.Parent = T
            local Tr = Instance.new("Frame")
            Tr.Size = UDim2.new(0, 30, 0, 16)
            Tr.Position = UDim2.new(1, -36, 0.5, -8)
            Tr.BackgroundColor3 = state and themeColor or Color3.fromRGB(50, 56, 72)
            Tr.BorderSizePixel = 0
            Tr.Parent = T
            local TrC = Instance.new("UICorner"); TrC.CornerRadius = UDim.new(1, 0); TrC.Parent = Tr
            local K = Instance.new("Frame")
            K.Size = UDim2.new(0, 12, 0, 12)
            K.Position = state and UDim2.new(1, -14, 0.5, -6) or UDim2.new(0, 2, 0.5, -6)
            K.BackgroundColor3 = Color3.fromRGB(255, 255, 255)
            K.BorderSizePixel = 0
            K.Parent = Tr
            local KC = Instance.new("UICorner"); KC.CornerRadius = UDim.new(1, 0); KC.Parent = K
            Track(T.MouseButton1Click:Connect(function()
                state = not state
                TweenService:Create(Tr, TweenInfo.new(0.2), {BackgroundColor3 = state and Window.ThemeColor or Color3.fromRGB(50, 56, 72)}):Play()
                TweenService:Create(K, TweenInfo.new(0.2, Enum.EasingStyle.Back), {Position = state and UDim2.new(1, -14, 0.5, -6) or UDim2.new(0, 2, 0.5, -6)}):Play()
                if flagName then savedData[flagName] = state; SaveCurrentConfig() end
                callback(state)
            end))
            return T
        end

        function MW:CreateSlider(text, flagName, minV, maxV, defaultV, callback, tooltip, fontSize)
            callback = callback or function() end
            local fs = fontSize or 10
            minV = minV or 0; maxV = maxV or 100
            local range = math.max(maxV - minV, 1)
            if flagName and savedData[flagName] ~= nil then defaultV = savedData[flagName] end
            defaultV = math.clamp(defaultV or minV, minV, maxV)
            local S = Instance.new("Frame")
            S.Size = UDim2.new(1, -4, 0, math.max(34, fs + 24))
            S.BackgroundColor3 = Color3.fromRGB(24, 28, 38)
            S.BackgroundTransparency = 0.15
            S.Parent = Container
            local SCc = Instance.new("UICorner"); SCc.CornerRadius = UDim.new(0, 6); SCc.Parent = S
            AddTooltip(S, tooltip or text)
            local Lbl = Instance.new("TextLabel")
            Lbl.Size = UDim2.new(1, -40, 0, 16)
            Lbl.Position = UDim2.new(0, 8, 0, 2)
            Lbl.BackgroundTransparency = 1
            Lbl.Text = text
            Lbl.TextColor3 = Color3.fromRGB(240, 240, 245)
            Lbl.TextXAlignment = Enum.TextXAlignment.Left
            Lbl.Font = Enum.Font.Gotham
            Lbl.TextSize = fs
            Lbl.Parent = S
            local Val = Instance.new("TextLabel")
            Val.Size = UDim2.new(0, 40, 0, 16)
            Val.Position = UDim2.new(1, -40, 0, 2)
            Val.BackgroundTransparency = 1
            Val.Text = tostring(defaultV)
            Val.TextColor3 = themeColor
            Val.TextXAlignment = Enum.TextXAlignment.Right
            Val.Font = Enum.Font.GothamBold
            Val.TextSize = fs
            Val.Parent = S
            registerTheme(Val, "TextColor3")
            local barY = math.max(22, fs + 12)
            local Bar = Instance.new("TextButton")
            Bar.Size = UDim2.new(1, -16, 0, 5)
            Bar.Position = UDim2.new(0, 8, 0, barY)
            Bar.BackgroundColor3 = Color3.fromRGB(50, 56, 72)
            Bar.Text = ""
            Bar.AutoButtonColor = false
            Bar.Name = "MiniSliderBar"
            Bar.Parent = S
            local BCc = Instance.new("UICorner"); BCc.CornerRadius = UDim.new(1, 0); BCc.Parent = Bar
            local Fill = Instance.new("Frame")
            local pct = (defaultV - minV) / range
            Fill.Size = UDim2.new(pct, 0, 1, 0)
            Fill.BackgroundColor3 = themeColor
            Fill.BorderSizePixel = 0
            Fill.Parent = Bar
            local FCc = Instance.new("UICorner"); FCc.CornerRadius = UDim.new(1, 0); FCc.Parent = Fill
            registerTheme(Fill, "BackgroundColor3")
            local sliding = false
            local function upd(input)
                local p = math.clamp((input.Position.X - Bar.AbsolutePosition.X) / Bar.AbsoluteSize.X, 0, 1)
                local v = math.floor(minV + (maxV - minV) * p + 0.5)
                Fill.Size = UDim2.new(p, 0, 1, 0)
                Val.Text = tostring(v)
                if flagName then savedData[flagName] = v; SaveCurrentConfig() end
                callback(v)
            end
            Track(Bar.InputBegan:Connect(function(input)
                if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
                    sliding = true; upd(input)
                end
            end))
            Track(UserInputService.InputChanged:Connect(function(input)
                if sliding and (input.UserInputType == Enum.UserInputType.MouseMovement or input.UserInputType == Enum.UserInputType.Touch) then
                    upd(input)
                end
            end))
            Track(UserInputService.InputEnded:Connect(function(input)
                if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then sliding = false end
            end))
            return S
        end

        function MW:CreateImage(imageId, height)
            if not imageId then return nil end
            local h = height or 60
            local F = Instance.new("Frame")
            F.Size = UDim2.new(1, -4, 0, h)
            F.BackgroundColor3 = Color3.fromRGB(24, 28, 38)
            F.BackgroundTransparency = 0.15
            F.ClipsDescendants = true
            F.Parent = Container
            local FCc = Instance.new("UICorner"); FCc.CornerRadius = UDim.new(0, 6); FCc.Parent = F
            local Img = Instance.new("ImageLabel")
            Img.Size = UDim2.new(1, -8, 1, -8)
            Img.Position = UDim2.new(0, 4, 0, 4)
            Img.BackgroundTransparency = 1
            Img.Image = typeof(imageId) == "number" and ("rbxassetid://" .. imageId) or imageId
            Img.ScaleType = Enum.ScaleType.Fit
            Img.Parent = F
            return F
        end

        function MW:SetPosition(x, y) MiniFrame.Position = UDim2.new(0, x, 0, y) end
        function MW:SetSize(w, h)
            MiniFrame.Size = UDim2.new(0, math.clamp(w, mwMinW, mwMaxW), 0, math.clamp(h, mwMinH, mwMaxH))
        end
        function MW:GetSize() return MiniFrame.AbsoluteSize end
        function MW:SetDraggable(bool) mwDraggable = bool end
        function MW:SetResizable(bool) mwResizable = bool; ResizeHandle.Visible = bool end
        function MW:SetDragBarVisible(bool)
            mwShowDragBar = bool
            DragBar.Visible = bool
            Container.Position = UDim2.new(0, 0, 0, bool and 20 or 0)
            Container.Size = UDim2.new(1, 0, 1, bool and -20 or 0)
        end
        function MW:Show() MiniFrame.Visible = true end
        function MW:Hide() MiniFrame.Visible = false end
        function MW:IsVisible() return MiniFrame.Visible end
        function MW:ToggleVisible() MiniFrame.Visible = not MiniFrame.Visible end
        function MW:Destroy() MiniFrame:Destroy() end
        function MW:GetFrame() return MiniFrame end

        return MW
    end

    -- ============ VISUAL EFFECTS METHODS ============
    function Window:SetBlur(enabled)
        if enabled then
            if not blurEffect or not blurEffect.Parent then
                blurEffect = Instance.new("BlurEffect")
                blurEffect.Size = 0
                blurEffect.Parent = Lighting
            end
            TweenService:Create(blurEffect, TweenInfo.new(0.4, Enum.EasingStyle.Quart), {Size = blurSize}):Play()
        else
            if blurEffect and blurEffect.Parent then
                TweenService:Create(blurEffect, TweenInfo.new(0.4, Enum.EasingStyle.Quart), {Size = 0}):Play()
            end
        end
    end

    function Window:SetParticles(enabled)
        if enabled then
            if not particleContainer or not particleContainer.Parent then createParticles() end
        else
            if particleConn then particleConn:Disconnect(); particleConn = nil end
            if particleContainer and particleContainer.Parent then
                particleContainer:Destroy(); particleContainer = nil
            end
            particleList = {}
        end
    end

    function Window:IsBlurEnabled()
        return blurEffect ~= nil and blurEffect.Parent ~= nil and blurEffect.Size > 0
    end

    function Window:IsParticlesEnabled()
        return particleContainer ~= nil and particleContainer.Parent ~= nil
    end

    -- ============ DESTROY ============
    function Window:Destroy()
        TweenService:Create(mainScale, TweenInfo.new(0.3, Enum.EasingStyle.Back, Enum.EasingDirection.In), {Scale = 0}):Play()
        task.wait(0.3)
        for _, conn in ipairs(connections) do
            pcall(function() conn:Disconnect() end)
        end
        connections = {}
        if blurEffect and blurEffect.Parent then blurEffect:Destroy() end
        if particleContainer and particleContainer.Parent then particleContainer:Destroy() end
        if ScreenGui then ScreenGui:Destroy() end
    end

    return Window
end

return AlatferaLib
