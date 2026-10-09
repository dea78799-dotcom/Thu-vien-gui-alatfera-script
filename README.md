--[[
    ============================================
    ALATFERA LIB V12 - VISUAL EDITION
    Thêm: Blur Background, Particle Effects
    API 100% tương thích V11
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
            pcall(function() default = Color3.fromHex(val) end); return default
        elseif ColorPresets[val:lower()] then return ColorPresets[val:lower()] end
    elseif typeof(val) == "Color3" then return val end
    return default or ColorPresets.blue
end

local function CopyToClipboard(st)
    if setclipboard then setclipboard(st)
    elseif toclipboard then toclipboard(st)
    elseif Syn and Syn.set_thread_identity then setclipboard(st) end
end

local function CreateRipple(button, input)
    button.ClipsDescendants = true
    local xPos = input.Position.X - button.AbsolutePosition.X
    local yPos = input.Position.Y - button.AbsolutePosition.Y
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
    local hubVersion     = config.Version or "v12.0"
    local showPlaytime   = (config.ShowPlaytime == nil) and true or config.ShowPlaytime
    local scriptNameText = config.LoadingScriptName or "Alatfera Script"
    local statusText     = config.LoadingStatus or "Đang tải giao diện..."

    local cornerRadius = config.CornerRadius or 12
    local themeColor   = ParseColor(config.ThemeColor, ColorPresets.blue)
    local enableStroke = (config.Stroke == nil) and true or config.Stroke
    local strokeColor  = ParseColor(config.StrokeColor, Color3.fromRGB(45, 50, 65))

    local useKeySystem = config.KeySystem or false
    local correctKey   = config.Key or "AlatferaKey123"
    local discordLink  = config.DiscordLink or "https://discord.gg/alatfera"
    local maxAttempts  = config.MaxAttempts or 10
    local remainingAttempts = maxAttempts

    -- === VISUAL EFFECTS (V12) ===
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
    ScreenGui.ZIndexBehavior = Enum.ZIndexBehavior.Sibling
    ScreenGui.Parent = ParentContainer

    local connections = {}
    local function Track(c) table.insert(connections, c); return c end
    local presetAppliers = {}

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

    -- ═══════════════════════════════════════════
    -- BLUR BACKGROUND (V12)
    -- ═══════════════════════════════════════════
    local blurEffect = nil
    if enableBlur then
        blurEffect = Instance.new("BlurEffect")
        blurEffect.Size = 0
        blurEffect.Parent = Lighting
        TweenService:Create(blurEffect, TweenInfo.new(0.5, Enum.EasingStyle.Quart), {Size = blurSize}):Play()
    end

    -- ═══════════════════════════════════════════
    -- PARTICLE EFFECTS (V12)
    -- ═══════════════════════════════════════════
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

            local pc = Instance.new("UICorner")
            pc.CornerRadius = UDim.new(1, 0)
            pc.Parent = p

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
                    local newY = p.Position.Y.Scale - data.speed * dt * 60
                    local newX = p.Position.X.Scale + data.drift + math.sin(data.wobble) * 0.0005
                    if newY < -0.05 then
                        newY = 1.05
                        newX = math.random()
                    end
                    if newX < -0.05 then newX = 1.05
                    elseif newX > 1.05 then newX = -0.05 end
                    p.Position = UDim2.new(newX, 0, newY, 0)
                end
            end
        end)
        Track(particleConn)
    end

    if enableParticles then
        createParticles()
    end

    -- Topbar
    local Topbar = Instance.new("Frame")
    Topbar.Size = UDim2.new(1, 0, 0, 46)
    Topbar.BackgroundColor3 = Color3.fromRGB(20, 23, 32); Topbar.BorderSizePixel = 0
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

    local hexString = string.format("#%02X%02X%02X", themeColor.R*255, themeColor.G*255, themeColor.B*255)
    local TitleLabel = Instance.new("TextLabel")
    TitleLabel.Size = UDim2.new(0.28, 0, 1, 0); TitleLabel.Position = UDim2.new(0, 18, 0, 0)
    TitleLabel.BackgroundTransparency = 1
    TitleLabel.Text = hubTitle .. (hubVersion ~= "" and ("  <font color=\"" .. hexString .. "\">" .. hubVersion .. "</font>") or "")
    TitleLabel.RichText = true
    TitleLabel.TextColor3 = Color3.fromRGB(255, 255, 255)
    TitleLabel.TextXAlignment = Enum.TextXAlignment.Left
    TitleLabel.Font = Enum.Font.GothamBold; TitleLabel.TextSize = 15
    TitleLabel.Parent = Topbar

    local StatsLabel = Instance.new("TextLabel")
    StatsLabel.Size = UDim2.new(0.22, 0, 1, 0); StatsLabel.Position = UDim2.new(0.28, 10, 0, 0)
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

    -- Close
    local CloseBtn = Instance.new("TextButton")
    CloseBtn.Size = UDim2.new(0, 34, 0, 34); CloseBtn.Position = UDim2.new(1, -42, 0, 6)
    CloseBtn.BackgroundTransparency = 1; CloseBtn.Text = "✕"
    CloseBtn.TextColor3 = Color3.fromRGB(245, 70, 90); CloseBtn.TextSize = 17
    CloseBtn.Font = Enum.Font.GothamBold; CloseBtn.Parent = Topbar

    -- Search
    local SearchBox = Instance.new("TextBox")
    SearchBox.Size = UDim2.new(0, 160, 0, 28); SearchBox.Position = UDim2.new(1, -350, 0, 9)
    SearchBox.BackgroundColor3 = Color3.fromRGB(12, 14, 19); SearchBox.Text = ""
    SearchBox.PlaceholderText = "🔍 Tìm kiếm..."
    SearchBox.TextColor3 = Color3.fromRGB(240, 240, 245)
    SearchBox.PlaceholderColor3 = Color3.fromRGB(110, 115, 130)
    SearchBox.Font = Enum.Font.Gotham; SearchBox.TextSize = 12
    SearchBox.Parent = Topbar
    local SrC = Instance.new("UICorner"); SrC.CornerRadius = UDim.new(0, 8); SrC.Parent = SearchBox
    if enableStroke then
        local SrS = Instance.new("UIStroke"); SrS.Color = strokeColor; SrS.Thickness = 1; SrS.Parent = SearchBox
    end
    Track(SearchBox.Focused:Connect(function()
        TweenService:Create(SearchBox, TweenInfo.new(0.2), {BackgroundColor3 = Color3.fromRGB(20, 24, 34)}):Play()
    end))
    Track(SearchBox.FocusLost:Connect(function()
        TweenService:Create(SearchBox, TweenInfo.new(0.2), {BackgroundColor3 = Color3.fromRGB(12, 14, 19)}):Play()
    end))

    -- === LOCK BUTTON ===
    local isLocked = false
    local LockBtn = Instance.new("TextButton")
    LockBtn.Size = UDim2.new(0, 34, 0, 34); LockBtn.Position = UDim2.new(1, -186, 0, 6)
    LockBtn.BackgroundTransparency = 1
    LockBtn.Text = "🔓"
    LockBtn.TextColor3 = Color3.fromRGB(180, 185, 200)
    LockBtn.TextSize = 16; LockBtn.Font = Enum.Font.GothamBold
    LockBtn.Parent = Topbar
    LockBtn:SetAttribute("SearchText", "lock khóa gui")

    Track(LockBtn.MouseButton1Click:Connect(function()
        isLocked = not isLocked
        LockBtn.Text = isLocked and "🔒" or "🔓"
        LockBtn.TextColor3 = isLocked and themeColor or Color3.fromRGB(180, 185, 200)
        TweenService:Create(LockBtn, TweenInfo.new(0.15, Enum.EasingStyle.Back), {Size = UDim2.new(0, 40, 0, 40)}):Play()
        task.wait(0.15)
        TweenService:Create(LockBtn, TweenInfo.new(0.15, Enum.EasingStyle.Back), {Size = UDim2.new(0, 34, 0, 34)}):Play()
        Window:Notify({
            Title = isLocked and "🔒 Đã Khóa" or "🔓 Đã Mở",
            Text = isLocked and "GUI sẽ không kéo được nữa." or "Có thể kéo GUI bình thường.",
            Duration = 1.5,
            Color = isLocked and "red" or "green",
        })
    end))

    -- Minimize
    local MinimizeBtn = Instance.new("TextButton")
    MinimizeBtn.Size = UDim2.new(0, 34, 0, 34); MinimizeBtn.Position = UDim2.new(1, -78, 0, 6)
    MinimizeBtn.BackgroundTransparency = 1; MinimizeBtn.Text = "↑"
    MinimizeBtn.TextColor3 = Color3.fromRGB(180, 185, 200); MinimizeBtn.TextSize = 18
    MinimizeBtn.Font = Enum.Font.GothamBold; MinimizeBtn.Parent = Topbar

    -- Help
    local HelpBtn = Instance.new("TextButton")
    HelpBtn.Size = UDim2.new(0, 34, 0, 34); HelpBtn.Position = UDim2.new(1, -114, 0, 6)
    HelpBtn.BackgroundTransparency = 1; HelpBtn.Text = "?"
    HelpBtn.TextColor3 = Color3.fromRGB(180, 185, 200); HelpBtn.TextSize = 17
    HelpBtn.Font = Enum.Font.GothamBold; HelpBtn.Parent = Topbar

    -- Keybind info
    local KeybindBtn = Instance.new("TextButton")
    KeybindBtn.Size = UDim2.new(0, 34, 0, 34); KeybindBtn.Position = UDim2.new(1, -150, 0, 6)
    KeybindBtn.BackgroundTransparency = 1; KeybindBtn.Text = "⌨"
    KeybindBtn.TextColor3 = Color3.fromRGB(180, 185, 200); KeybindBtn.TextSize = 17
    KeybindBtn.Font = Enum.Font.GothamBold; KeybindBtn.Parent = Topbar

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
    end))

    local isMinimized = false
    Track(MinimizeBtn.MouseButton1Click:Connect(function()
        isMinimized = not isMinimized
        if isMinimized then
            MinimizeBtn.Text = "↓"
            MainFrame:TweenSize(UDim2.new(0.70, 0, 0, 46), Enum.EasingDirection.Out, Enum.EasingStyle.Back, 0.35, true)
        else
            MinimizeBtn.Text = "↑"
            MainFrame:TweenSize(UDim2.new(0.70, 0, 0.88, 0), Enum.EasingDirection.Out, Enum.EasingStyle.Back, 0.35, true)
        end
    end))

    local currentToggleKey = config.ToggleKey or Enum.KeyCode.RightControl
    Track(HelpBtn.MouseButton1Click:Connect(function()
        Window:Notify({
            Title = "Hướng Dẫn",
            Text = "• Kéo topbar di chuyển GUI.\n• 🔒 Khóa GUI (không kéo được).\n• RightControl ẩn/hiện.",
            Duration = 5,
        })
    end))
    Track(KeybindBtn.MouseButton1Click:Connect(function()
        Window:Notify({ Title = "Phím Tắt", Text = "Ẩn/hiện GUI: " .. tostring(currentToggleKey.Name), Duration = 4 })
    end))

    Track(UserInputService.InputBegan:Connect(function(input, gpe)
        if not gpe and input.KeyCode == currentToggleKey then
            MainFrame.Visible = not MainFrame.Visible
            OpenBtnFrame.Visible = not MainFrame.Visible
        end
    end))

    -- Drag
    local dragging, dragStart, startPos
    Track(Topbar.InputBegan:Connect(function(input)
        if isLocked then return end
        if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
            dragging = true; dragStart = input.Position; startPos = MainFrame.Position
        end
    end))
    Track(UserInputService.InputChanged:Connect(function(input)
        if isLocked then dragging = false; return end
        if dragging and (input.UserInputType == Enum.UserInputType.MouseMovement or input.UserInputType == Enum.UserInputType.Touch) then
            local delta = input.Position - dragStart
            MainFrame.Position = UDim2.new(startPos.X.Scale, startPos.X.Offset + delta.X, startPos.Y.Scale, startPos.Y.Offset + delta.Y)
        end
    end))
    Track(Topbar.InputEnded:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then dragging = false end
    end))

    -- ============ SIDEBAR ============
    local Sidebar = Instance.new("ScrollingFrame")
    Sidebar.Size = UDim2.new(0, 160, 1, showPlaytime and -86 or -58)
    Sidebar.Position = UDim2.new(0, 10, 0, 54)
    Sidebar.BackgroundColor3 = Color3.fromRGB(19, 21, 28)
    Sidebar.BorderSizePixel = 0
    Sidebar.ScrollBarThickness = 2; Sidebar.ScrollBarImageColor3 = themeColor
    Sidebar.Parent = MainFrame
    local SbC = Instance.new("UICorner"); SbC.CornerRadius = UDim.new(0, cornerRadius); SbC.Parent = Sidebar
    if enableStroke then
        local SbS = Instance.new("UIStroke"); SbS.Color = strokeColor; SbS.Thickness = 1; SbS.Parent = Sidebar
    end

    local TabListLayout = Instance.new("UIListLayout")
    TabListLayout.Padding = UDim.new(0, 6); TabListLayout.SortOrder = Enum.SortOrder.LayoutOrder
    TabListLayout.Parent = Sidebar
    Track(TabListLayout:GetPropertyChangedSignal("AbsoluteContentSize"):Connect(function()
        Sidebar.CanvasSize = UDim2.new(0, 0, 0, TabListLayout.AbsoluteContentSize.Y + 10)
    end))

    local ActiveIndicator = Instance.new("Frame")
    ActiveIndicator.Size = UDim2.new(0, 3, 0, 20)
    ActiveIndicator.Position = UDim2.new(0, 0, 0, 8)
    ActiveIndicator.BackgroundColor3 = themeColor
    ActiveIndicator.BorderSizePixel = 0
    ActiveIndicator.ZIndex = 3
    ActiveIndicator.Parent = Sidebar
    local AIC = Instance.new("UICorner"); AIC.CornerRadius = UDim.new(1, 0); AIC.Parent = ActiveIndicator

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
                for _, elem in pairs(container:GetChildren()) do
                    local searchText = elem:GetAttribute("SearchText")
                    if searchText then
                        elem.Visible = (query == "" or string.find(searchText, query, 1, true) ~= nil)
                    end
                end
            end
        end
    end))

    -- ============ NOTIFY ============
    local NotifContainer = Instance.new("Frame")
    NotifContainer.Size = UDim2.new(0, 250, 1, -20)
    NotifContainer.Position = UDim2.new(1, -260, 0, 10)
    NotifContainer.BackgroundTransparency = 1
    NotifContainer.Parent = ScreenGui
    local NotifLayout = Instance.new("UIListLayout")
    NotifLayout.VerticalAlignment = Enum.VerticalAlignment.Bottom
    NotifLayout.Padding = UDim.new(0, 6); NotifLayout.SortOrder = Enum.SortOrder.LayoutOrder
    NotifLayout.Parent = NotifContainer

    function Window:Notify(notifConfig)
        notifConfig = notifConfig or {}
        local title    = notifConfig.Title or "Thông Báo"
        local text     = notifConfig.Text or ""
        local duration = notifConfig.Duration or 3
        local nColor   = ParseColor(notifConfig.Color, themeColor)
        local pos      = notifConfig.Position

        local Item = Instance.new("Frame")
        Item.Size = UDim2.new(1, 0, 0, 54)
        Item.BackgroundColor3 = Color3.fromRGB(22, 25, 34)
        Item.BorderSizePixel = 0; Item.Parent = NotifContainer
        local ICC = Instance.new("UICorner"); ICC.CornerRadius = UDim.new(0, cornerRadius); ICC.Parent = Item
        if enableStroke then
            local IS = Instance.new("UIStroke"); IS.Color = strokeColor; IS.Thickness = 1; IS.Parent = Item
        end
        local nScale = Instance.new("UIScale"); nScale.Scale = 0.7; nScale.Parent = Item
        TweenService:Create(nScale, TweenInfo.new(0.35, Enum.EasingStyle.Back, Enum.EasingDirection.Out), {Scale = 1}):Play()

        local Bar = Instance.new("Frame")
        Bar.Size = UDim2.new(0, 4, 1, 0); Bar.BackgroundColor3 = nColor
        Bar.BorderSizePixel = 0; Bar.Parent = Item
        local BC2 = Instance.new("UICorner"); BC2.CornerRadius = UDim.new(0, cornerRadius); BC2.Parent = Bar

        local NTitle = Instance.new("TextLabel")
        NTitle.Size = UDim2.new(1, -12, 0, 20); NTitle.Position = UDim2.new(0, 12, 0, 4)
        NTitle.BackgroundTransparency = 1; NTitle.Text = title
        NTitle.TextColor3 = Color3.fromRGB(255, 255, 255)
        NTitle.Font = Enum.Font.GothamBold; NTitle.TextSize = 13
        NTitle.TextXAlignment = Enum.TextXAlignment.Left
        NTitle.Parent = Item

        local NText = Instance.new("TextLabel")
        NText.Size = UDim2.new(1, -12, 0, 22); NText.Position = UDim2.new(0, 12, 0, 24)
        NText.BackgroundTransparency = 1; NText.Text = text
        NText.TextColor3 = Color3.fromRGB(160, 165, 180)
        NText.Font = Enum.Font.Gotham; NText.TextSize = 12
        NText.TextXAlignment = Enum.TextXAlignment.Left
        NText.Parent = Item

        if pos == "left" then NotifContainer.Position = UDim2.new(0, 10, 0, 10)
        elseif pos == "center" then NotifContainer.Position = UDim2.new(0.5, -125, 0, 10)
        else NotifContainer.Position = UDim2.new(1, -260, 0, 10) end

        task.spawn(function()
            task.wait(duration)
            TweenService:Create(nScale, TweenInfo.new(0.3, Enum.EasingStyle.Back, Enum.EasingDirection.In), {Scale = 0}):Play()
            TweenService:Create(Item, TweenInfo.new(0.3), {BackgroundTransparency = 1}):Play()
            TweenService:Create(NTitle, TweenInfo.new(0.3), {TextTransparency = 1}):Play()
            TweenService:Create(NText, TweenInfo.new(0.3), {TextTransparency = 1}):Play()
            TweenService:Create(Bar, TweenInfo.new(0.3), {BackgroundTransparency = 1}):Play()
            task.wait(0.3)
            Item:Destroy()
        end)
    end

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
    end
    local TooltipLabel = Instance.new("TextLabel")
    TooltipLabel.Size = UDim2.new(1, -12, 1, 0); TooltipLabel.Position = UDim2.new(0, 6, 0, 0)
    TooltipLabel.BackgroundTransparency = 1
    TooltipLabel.TextColor3 = Color3.fromRGB(240, 240, 245)
    TooltipLabel.Font = Enum.Font.Gotham; TooltipLabel.TextSize = 12
    TooltipLabel.TextXAlignment = Enum.TextXAlignment.Left
    TooltipLabel.Parent = TooltipFrame

    local function AddTooltip(widget, text)
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

    -- ============ CREATE TAB ============
    function Window:CreateTab(tabName, iconEmoji)
        local Tab = {}
        local displayText = iconEmoji and (iconEmoji .. "  " .. tabName) or ("  " .. tabName)

        local TabButton = Instance.new("TextButton")
        TabButton.Size = UDim2.new(1, -6, 0, 36)
        TabButton.BackgroundColor3 = FirstTab and Color3.fromRGB(28, 33, 46) or Color3.fromRGB(19, 21, 28)
        TabButton.Text = displayText
        TabButton.TextColor3 = FirstTab and Color3.fromRGB(255, 255, 255) or Color3.fromRGB(140, 145, 160)
        TabButton.TextXAlignment = Enum.TextXAlignment.Left
        TabButton.Font = Enum.Font.GothamBold; TabButton.TextSize = 13
        TabButton.Parent = Sidebar
        local TBC = Instance.new("UICorner"); TBC.CornerRadius = UDim.new(0, cornerRadius); TBC.Parent = TabButton
        if enableStroke then
            local TBS = Instance.new("UIStroke"); TBS.Color = strokeColor; TBS.Thickness = 1; TBS.Parent = TabButton
        end

        local TabContainer = Instance.new("ScrollingFrame")
        TabContainer.Size = UDim2.new(1, 0, 1, 0)
        TabContainer.BackgroundTransparency = 1
        TabContainer.BorderSizePixel = 0
        TabContainer.ScrollBarThickness = 3
        TabContainer.ScrollBarImageColor3 = themeColor
        TabContainer.Visible = FirstTab
        TabContainer.Parent = ContentArea

        local ContainerLayout = Instance.new("UIListLayout")
        ContainerLayout.Padding = UDim.new(0, 10); ContainerLayout.SortOrder = Enum.SortOrder.LayoutOrder
        ContainerLayout.Parent = TabContainer
        Track(ContainerLayout:GetPropertyChangedSignal("AbsoluteContentSize"):Connect(function()
            TabContainer.CanvasSize = UDim2.new(0, 0, 0, ContainerLayout.AbsoluteContentSize.Y + 15)
        end))

        Track(TabButton.MouseEnter:Connect(function()
            if not TabContainer.Visible then
                TweenService:Create(TabButton, TweenInfo.new(0.15), {BackgroundColor3 = Color3.fromRGB(24, 28, 40)}):Play()
            end
        end))
        Track(TabButton.MouseLeave:Connect(function()
            if not TabContainer.Visible then
                TweenService:Create(TabButton, TweenInfo.new(0.15), {BackgroundColor3 = Color3.fromRGB(19, 21, 28)}):Play()
            end
        end))

        Track(TabButton.MouseButton1Click:Connect(function()
            for _, child in pairs(ContentArea:GetChildren()) do
                if child:IsA("ScrollingFrame") then child.Visible = false end
            end
            for _, btn in pairs(Sidebar:GetChildren()) do
                if btn:IsA("TextButton") then
                    btn.BackgroundColor3 = Color3.fromRGB(19, 21, 28)
                    btn.TextColor3 = Color3.fromRGB(140, 145, 160)
                end
            end
            TabContainer.Visible = true
            TabButton.BackgroundColor3 = Color3.fromRGB(28, 33, 46)
            TabButton.TextColor3 = Color3.fromRGB(255, 255, 255)
            TabContainer.Position = UDim2.new(0.06, 0, 0, 0)
            TweenService:Create(TabContainer, TweenInfo.new(0.25, Enum.EasingStyle.Quart, Enum.EasingDirection.Out), {
                Position = UDim2.new(0, 0, 0, 0),
            }):Play()
            TweenService:Create(ActiveIndicator, TweenInfo.new(0.25, Enum.EasingStyle.Quart), {
                Position = UDim2.new(0, 0, 0, TabButton.AbsolutePosition.Y - Sidebar.AbsolutePosition.Y + Sidebar.CanvasPosition.Y + 8),
            }):Play()
        end))

        FirstTab = false

        -- ===== WIDGET: LABEL =====
        function Tab:CreateLabel(text)
            local Label = Instance.new("TextLabel")
            Label.Size = UDim2.new(1, -10, 0, 28)
            Label.BackgroundColor3 = Color3.fromRGB(22, 25, 34)
            Label.Text = "  " .. text
            Label.TextColor3 = Color3.fromRGB(170, 175, 190)
            Label.TextXAlignment = Enum.TextXAlignment.Left
            Label.Font = Enum.Font.Gotham; Label.TextSize = 12
            Label.Parent = TabContainer
            local LC = Instance.new("UICorner"); LC.CornerRadius = UDim.new(0, cornerRadius); LC.Parent = Label
            if enableStroke then
                local LS = Instance.new("UIStroke"); LS.Color = strokeColor; LS.Thickness = 1; LS.Parent = Label
            end
            Label:SetAttribute("SearchText", string.lower(text))
            AddTooltip(Label, text)
            return Label
        end

        -- ===== WIDGET: PARAGRAPH =====
        function Tab:CreateParagraph(text)
            local Wrap = Instance.new("Frame")
            Wrap.Size = UDim2.new(1, -10, 0, 0)
            Wrap.AutomaticSize = Enum.AutomaticSize.Y
            Wrap.BackgroundColor3 = Color3.fromRGB(22, 25, 34)
            Wrap.BorderSizePixel = 0
            Wrap.Parent = TabContainer
            local WC = Instance.new("UICorner"); WC.CornerRadius = UDim.new(0, cornerRadius); WC.Parent = Wrap
            if enableStroke then
                local WS = Instance.new("UIStroke"); WS.Color = strokeColor; WS.Thickness = 1; WS.Parent = Wrap
            end
            local Padding = Instance.new("UIPadding")
            Padding.PaddingTop = UDim.new(0, 8); Padding.PaddingBottom = UDim.new(0, 8)
            Padding.PaddingLeft = UDim.new(0, 12); Padding.PaddingRight = UDim.new(0, 12)
            Padding.Parent = Wrap

            local TextLbl = Instance.new("TextLabel")
            TextLbl.Size = UDim2.new(1, 0, 0, 0)
            TextLbl.AutomaticSize = Enum.AutomaticSize.Y
            TextLbl.BackgroundTransparency = 1
            TextLbl.Text = text
            TextLbl.TextColor3 = Color3.fromRGB(190, 195, 210)
            TextLbl.Font = Enum.Font.Gotham; TextLbl.TextSize = 13
            TextLbl.TextXAlignment = Enum.TextXAlignment.Left
            TextLbl.TextYAlignment = Enum.TextYAlignment.Top
            TextLbl.TextWrapped = true
            TextLbl.Parent = Wrap
            Wrap:SetAttribute("SearchText", string.lower(text))
            return Wrap
        end

        -- ===== WIDGET: DIVIDER =====
        function Tab:CreateDivider()
            local Div = Instance.new("Frame")
            Div.Size = UDim2.new(1, -20, 0, 1)
            Div.Position = UDim2.new(0, 10, 0, 0)
            Div.BackgroundColor3 = Color3.fromRGB(45, 50, 65)
            Div.BorderSizePixel = 0
            Div.Parent = TabContainer
            return Div
        end

        -- ===== WIDGET: SECTION =====
        function Tab:CreateSection(text)
            local SectionFrame = Instance.new("Frame")
            SectionFrame.Size = UDim2.new(1, -10, 0, 24)
            SectionFrame.BackgroundTransparency = 1
            SectionFrame.Parent = TabContainer
            local SecText = Instance.new("TextLabel")
            SecText.Size = UDim2.new(1, 0, 1, 0); SecText.BackgroundTransparency = 1
            SecText.Text = "───  " .. string.upper(text or "SECTION") .. "  ───"
            SecText.TextColor3 = themeColor
            SecText.Font = Enum.Font.GothamBold; SecText.TextSize = 12
            SecText.TextTransparency = 1
            SecText.Parent = SectionFrame
            task.spawn(function()
                TweenService:Create(SecText, TweenInfo.new(0.5, Enum.EasingStyle.Quart), {TextTransparency = 0}):Play()
            end)
            SectionFrame:SetAttribute("SearchText", string.lower(text or ""))
        end

        -- ===== WIDGET: BUTTON =====
        function Tab:CreateButton(btnText, callback)
            callback = callback or function() end
            local Button = Instance.new("TextButton")
            Button.Size = UDim2.new(1, -10, 0, 38)
            Button.BackgroundColor3 = Color3.fromRGB(24, 28, 38)
            Button.Text = btnText; Button.TextColor3 = Color3.fromRGB(240, 240, 245)
            Button.Font = Enum.Font.Gotham; Button.TextSize = 13
            Button.Parent = TabContainer
            local BC3 = Instance.new("UICorner"); BC3.CornerRadius = UDim.new(0, cornerRadius); BC3.Parent = Button
            if enableStroke then
                local BS = Instance.new("UIStroke"); BS.Color = strokeColor; BS.Thickness = 1; BS.Parent = Button
            end
            Button:SetAttribute("SearchText", string.lower(btnText))
            AttachButtonEffects(Button, Track, Color3.fromRGB(34, 40, 56))

            Track(Button.MouseButton1Click:Connect(function(input)
                CreateRipple(Button, input)
                TweenService:Create(Button, TweenInfo.new(0.1), {BackgroundColor3 = themeColor}):Play()
                task.wait(0.1)
                TweenService:Create(Button, TweenInfo.new(0.15), {BackgroundColor3 = Color3.fromRGB(24, 28, 38)}):Play()
                callback()
            end))
            return Button
        end

        -- ===== WIDGET: TOGGLE =====
        function Tab:CreateToggle(toggleText, flagName, defaultState, callback)
            callback = callback or function() end
            if flagName and savedData[flagName] ~= nil then defaultState = savedData[flagName] end
            local state = defaultState or false

            local ToggleFrame = Instance.new("TextButton")
            ToggleFrame.Size = UDim2.new(1, -10, 0, 38)
            ToggleFrame.BackgroundColor3 = Color3.fromRGB(24, 28, 38)
            ToggleFrame.Text = ""
            ToggleFrame.Parent = TabContainer
            local TFC = Instance.new("UICorner"); TFC.CornerRadius = UDim.new(0, cornerRadius); TFC.Parent = ToggleFrame
            if enableStroke then
                local TFS = Instance.new("UIStroke"); TFS.Color = strokeColor; TFS.Thickness = 1; TFS.Parent = ToggleFrame
            end
            ToggleFrame:SetAttribute("SearchText", string.lower(toggleText))

            local Label = Instance.new("TextLabel")
            Label.Size = UDim2.new(1, -55, 1, 0); Label.Position = UDim2.new(0, 14, 0, 0)
            Label.BackgroundTransparency = 1; Label.Text = toggleText
            Label.TextColor3 = Color3.fromRGB(240, 240, 245)
            Label.TextXAlignment = Enum.TextXAlignment.Left
            Label.Font = Enum.Font.Gotham; Label.TextSize = 13
            Label.Parent = ToggleFrame

            local TrackFrame = Instance.new("Frame")
            TrackFrame.Size = UDim2.new(0, 42, 0, 22); TrackFrame.Position = UDim2.new(1, -52, 0.5, -11)
            TrackFrame.BackgroundColor3 = state and themeColor or Color3.fromRGB(50, 56, 72)
            TrackFrame.BorderSizePixel = 0; TrackFrame.Parent = ToggleFrame
            local TFC2 = Instance.new("UICorner"); TFC2.CornerRadius = UDim.new(1, 0); TFC2.Parent = TrackFrame

            local Knob = Instance.new("Frame")
            Knob.Size = UDim2.new(0, 18, 0, 18)
            Knob.Position = state and UDim2.new(1, -20, 0.5, -9) or UDim2.new(0, 2, 0.5, -9)
            Knob.BackgroundColor3 = Color3.fromRGB(255, 255, 255)
            Knob.BorderSizePixel = 0; Knob.Parent = TrackFrame
            local KC2 = Instance.new("UICorner"); KC2.CornerRadius = UDim.new(1, 0); KC2.Parent = Knob

            local function applyToggle(newState)
                state = newState
                TweenService:Create(TrackFrame, TweenInfo.new(0.3, Enum.EasingStyle.Back, Enum.EasingDirection.Out), {
                    BackgroundColor3 = state and themeColor or Color3.fromRGB(50, 56, 72),
                }):Play()
                TweenService:Create(Knob, TweenInfo.new(0.3, Enum.EasingStyle.Back, Enum.EasingDirection.Out), {
                    Position = state and UDim2.new(1, -20, 0.5, -9) or UDim2.new(0, 2, 0.5, -9),
                }):Play()
            end

            Track(ToggleFrame.MouseButton1Click:Connect(function()
                state = not state
                applyToggle(state)
                if flagName then savedData[flagName] = state; SaveCurrentConfig() end
                callback(state)
            end))

            if flagName then
                table.insert(presetAppliers, function()
                    if savedData[flagName] ~= nil and savedData[flagName] ~= state then
                        applyToggle(savedData[flagName])
                        callback(savedData[flagName])
                    end
                end)
            end

            if defaultState then callback(true) end
            return ToggleFrame
        end

        -- ===== WIDGET: SLIDER =====
        function Tab:CreateSlider(sliderText, flagName, minVal, maxVal, defaultVal, callback)
            callback = callback or function() end
            minVal = minVal or 0; maxVal = maxVal or 1000
            if flagName and savedData[flagName] ~= nil then defaultVal = savedData[flagName] end
            defaultVal = math.clamp(defaultVal or minVal, minVal, maxVal)

            local SliderFrame = Instance.new("Frame")
            SliderFrame.Size = UDim2.new(1, -10, 0, 48)
            SliderFrame.BackgroundColor3 = Color3.fromRGB(24, 28, 38)
            SliderFrame.Parent = TabContainer
            local SFC = Instance.new("UICorner"); SFC.CornerRadius = UDim.new(0, cornerRadius); SFC.Parent = SliderFrame
            if enableStroke then
                local SFS = Instance.new("UIStroke"); SFS.Color = strokeColor; SFS.Thickness = 1; SFS.Parent = SliderFrame
            end
            SliderFrame:SetAttribute("SearchText", string.lower(sliderText))

            local Label = Instance.new("TextLabel")
            Label.Size = UDim2.new(1, -60, 0, 22); Label.Position = UDim2.new(0, 14, 0, 3)
            Label.BackgroundTransparency = 1; Label.Text = sliderText
            Label.TextColor3 = Color3.fromRGB(240, 240, 245)
            Label.TextXAlignment = Enum.TextXAlignment.Left
            Label.Font = Enum.Font.Gotham; Label.TextSize = 13
            Label.Parent = SliderFrame

            local ValueLabel = Instance.new("TextLabel")
            ValueLabel.Size = UDim2.new(0, 50, 0, 22); ValueLabel.Position = UDim2.new(1, -60, 0, 3)
            ValueLabel.BackgroundTransparency = 1; ValueLabel.Text = tostring(defaultVal)
            ValueLabel.TextColor3 = themeColor
            ValueLabel.Font = Enum.Font.GothamBold; ValueLabel.TextSize = 13
            ValueLabel.Parent = SliderFrame

            local SliderBar = Instance.new("TextButton")
            SliderBar.Size = UDim2.new(1, -28, 0, 7); SliderBar.Position = UDim2.new(0, 14, 0, 30)
            SliderBar.BackgroundColor3 = Color3.fromRGB(50, 56, 72)
            SliderBar.Text = ""; SliderBar.AutoButtonColor = false
            SliderBar.Parent = SliderFrame
            local SBC = Instance.new("UICorner"); SBC.CornerRadius = UDim.new(1, 0); SBC.Parent = SliderBar

            local FillBar = Instance.new("Frame")
            local initPercent = (defaultVal - minVal) / (maxVal - minVal)
            FillBar.Size = UDim2.new(initPercent, 0, 1, 0)
            FillBar.BackgroundColor3 = themeColor
            FillBar.BorderSizePixel = 0; FillBar.Parent = SliderBar
            local FBC = Instance.new("UICorner"); FBC.CornerRadius = UDim.new(1, 0); FBC.Parent = FillBar

            local Thumb = Instance.new("Frame")
            Thumb.Size = UDim2.new(0, 16, 0, 16)
            Thumb.Position = UDim2.new(initPercent, -8, 0.5, -8)
            Thumb.BackgroundColor3 = Color3.fromRGB(255, 255, 255)
            Thumb.BorderSizePixel = 0; Thumb.ZIndex = 3
            Thumb.Parent = SliderBar
            local THC = Instance.new("UICorner"); THC.CornerRadius = UDim.new(1, 0); THC.Parent = Thumb
            local ThumbStroke = Instance.new("UIStroke")
            ThumbStroke.Color = themeColor; ThumbStroke.Thickness = 2; ThumbStroke.Parent = Thumb

            local sliding = false
            local function applyValue(val)
                local percent = (val - minVal) / (maxVal - minVal)
                FillBar.Size = UDim2.new(percent, 0, 1, 0)
                Thumb.Position = UDim2.new(percent, -8, 0.5, -8)
                ValueLabel.Text = tostring(val)
            end

            local function updateSlider(input)
                local percent = math.clamp((input.Position.X - SliderBar.AbsolutePosition.X) / SliderBar.AbsoluteSize.X, 0, 1)
                local val = math.floor(minVal + (maxVal - minVal) * percent)
                applyValue(val)
                if flagName then savedData[flagName] = val; SaveCurrentConfig() end
                callback(val)
            end

            Track(SliderBar.InputBegan:Connect(function(input)
                if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
                    sliding = true; updateSlider(input)
                    TweenService:Create(ThumbStroke, TweenInfo.new(0.15), {Thickness = 3}):Play()
                end
            end))
            Track(UserInputService.InputChanged:Connect(function(input)
                if sliding and (input.UserInputType == Enum.UserInputType.MouseMovement or input.UserInputType == Enum.UserInputType.Touch) then
                    updateSlider(input)
                end
            end))
            Track(UserInputService.InputEnded:Connect(function(input)
                if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
                    sliding = false
                    TweenService:Create(ThumbStroke, TweenInfo.new(0.2), {Thickness = 2}):Play()
                end
            end))

            if flagName then
                table.insert(presetAppliers, function()
                    if savedData[flagName] ~= nil then
                        applyValue(savedData[flagName])
                        callback(savedData[flagName])
                    end
                end)
            end
        end

        -- ===== WIDGET: DROPDOWN =====
        function Tab:CreateDropdown(dropText, options, defaultOption, callback)
            callback = callback or function() end
            options = options or {}
            local currentChoice = defaultOption or options[1] or "Chưa chọn"
            local MAX_HEIGHT = 150

            local DropFrame = Instance.new("Frame")
            DropFrame.Size = UDim2.new(1, -10, 0, 38)
            DropFrame.BackgroundColor3 = Color3.fromRGB(24, 28, 38)
            DropFrame.ClipsDescendants = true
            DropFrame.Parent = TabContainer
            local DFC = Instance.new("UICorner"); DFC.CornerRadius = UDim.new(0, cornerRadius); DFC.Parent = DropFrame
            if enableStroke then
                local DFS = Instance.new("UIStroke"); DFS.Color = strokeColor; DFS.Thickness = 1; DFS.Parent = DropFrame
            end
            DropFrame:SetAttribute("SearchText", string.lower(dropText))

            local HeaderBtn = Instance.new("TextButton")
            HeaderBtn.Size = UDim2.new(1, 0, 0, 38); HeaderBtn.BackgroundTransparency = 1
            HeaderBtn.Text = ""; HeaderBtn.Parent = DropFrame

            local Label = Instance.new("TextLabel")
            Label.Size = UDim2.new(0.5, 0, 1, 0); Label.Position = UDim2.new(0, 14, 0, 0)
            Label.BackgroundTransparency = 1; Label.Text = dropText
            Label.TextColor3 = Color3.fromRGB(240, 240, 245)
            Label.TextXAlignment = Enum.TextXAlignment.Left
            Label.Font = Enum.Font.Gotham; Label.TextSize = 13
            Label.Parent = HeaderBtn

            local SelectedLabel = Instance.new("TextLabel")
            SelectedLabel.Size = UDim2.new(0.5, -20, 1, 0); SelectedLabel.Position = UDim2.new(0.5, -5, 0, 0)
            SelectedLabel.BackgroundTransparency = 1
            SelectedLabel.Text = currentChoice .. " ▼"
            SelectedLabel.TextColor3 = themeColor
            SelectedLabel.TextXAlignment = Enum.TextXAlignment.Right
            SelectedLabel.Font = Enum.Font.GothamBold; SelectedLabel.TextSize = 12
            SelectedLabel.Parent = HeaderBtn

            local OptionContainer = Instance.new("ScrollingFrame")
            OptionContainer.Size = UDim2.new(1, -16, 0, math.min(#options * 32, MAX_HEIGHT))
            OptionContainer.Position = UDim2.new(0, 8, 0, 42)
            OptionContainer.BackgroundTransparency = 1
            OptionContainer.BorderSizePixel = 0
            OptionContainer.ScrollBarThickness = 2
            OptionContainer.ScrollBarImageColor3 = themeColor
            OptionContainer.CanvasSize = UDim2.new(0, 0, 0, #options * 32)
            OptionContainer.Parent = DropFrame

            local OptLayout = Instance.new("UIListLayout")
            OptLayout.Padding = UDim.new(0, 4); OptLayout.Parent = OptionContainer

            local isExpanded = false
            Track(HeaderBtn.MouseButton1Click:Connect(function()
                isExpanded = not isExpanded
                SelectedLabel.Text = currentChoice .. (isExpanded and " ▲" or " ▼")
                local expandedHeight = 46 + math.min(#options * 32, MAX_HEIGHT)
                local targetHeight = isExpanded and expandedHeight or 38
                DropFrame:TweenSize(UDim2.new(1, -10, 0, targetHeight), Enum.EasingDirection.Out, Enum.EasingStyle.Quart, 0.25, true)
            end))

            for _, opt in ipairs(options) do
                local OptBtn = Instance.new("TextButton")
                OptBtn.Size = UDim2.new(1, 0, 0, 28)
                OptBtn.BackgroundColor3 = Color3.fromRGB(18, 20, 26)
                OptBtn.Text = opt
                OptBtn.TextColor3 = Color3.fromRGB(200, 205, 220)
                OptBtn.Font = Enum.Font.Gotham; OptBtn.TextSize = 12
                OptBtn.Parent = OptionContainer
                local OBC2 = Instance.new("UICorner"); OBC2.CornerRadius = UDim.new(0, cornerRadius - 2); OBC2.Parent = OptBtn
                Track(OptBtn.MouseEnter:Connect(function()
                    TweenService:Create(OptBtn, TweenInfo.new(0.15), {BackgroundColor3 = Color3.fromRGB(28, 32, 42)}):Play()
                end))
                Track(OptBtn.MouseLeave:Connect(function()
                    TweenService:Create(OptBtn, TweenInfo.new(0.15), {BackgroundColor3 = Color3.fromRGB(18, 20, 26)}):Play()
                end))
                Track(OptBtn.MouseButton1Click:Connect(function()
                    currentChoice = opt
                    SelectedLabel.Text = currentChoice .. " ▼"
                    isExpanded = false
                    DropFrame:TweenSize(UDim2.new(1, -10, 0, 38), Enum.EasingDirection.Out, Enum.EasingStyle.Quart, 0.2, true)
                    callback(opt)
                end))
            end
        end

        -- ===== WIDGET: TEXTBOX =====
        function Tab:CreateTextbox(boxText, flagName, maxChars, callback)
            callback = callback or function() end
            local defaultVal = ""
            if flagName and savedData[flagName] ~= nil then defaultVal = savedData[flagName] end

            local BoxFrame = Instance.new("Frame")
            BoxFrame.Size = UDim2.new(1, -10, 0, 38)
            BoxFrame.BackgroundColor3 = Color3.fromRGB(24, 28, 38)
            BoxFrame.Parent = TabContainer
            local BFC = Instance.new("UICorner"); BFC.CornerRadius = UDim.new(0, cornerRadius); BFC.Parent = BoxFrame
            if enableStroke then
                local BFS = Instance.new("UIStroke"); BFS.Color = strokeColor; BFS.Thickness = 1; BFS.Parent = BoxFrame
            end
            BoxFrame:SetAttribute("SearchText", string.lower(boxText))

            local Label = Instance.new("TextLabel")
            Label.Size = UDim2.new(0.5, -10, 1, 0); Label.Position = UDim2.new(0, 14, 0, 0)
            Label.BackgroundTransparency = 1; Label.Text = boxText
            Label.TextColor3 = Color3.fromRGB(240, 240, 245)
            Label.TextXAlignment = Enum.TextXAlignment.Left
            Label.Font = Enum.Font.Gotham; Label.TextSize = 13
            Label.Parent = BoxFrame

            local InputBox = Instance.new("TextBox")
            InputBox.Size = UDim2.new(0.5, -10, 0, 26); InputBox.Position = UDim2.new(0.5, 0, 0.5, -13)
            InputBox.BackgroundColor3 = Color3.fromRGB(15, 17, 23)
            InputBox.Text = defaultVal
            InputBox.PlaceholderText = maxChars and ("Max " .. maxChars .. " chữ") or "Nhập..."
            InputBox.TextColor3 = Color3.fromRGB(255, 255, 255)
            InputBox.PlaceholderColor3 = Color3.fromRGB(120, 125, 140)
            InputBox.Font = Enum.Font.Gotham; InputBox.TextSize = 12
            InputBox.Parent = BoxFrame
            local IBC = Instance.new("UICorner"); IBC.CornerRadius = UDim.new(0, cornerRadius - 2); IBC.Parent = InputBox

            Track(InputBox.Focused:Connect(function()
                TweenService:Create(InputBox, TweenInfo.new(0.2), {BackgroundColor3 = Color3.fromRGB(22, 26, 36)}):Play()
            end))
            Track(InputBox.FocusLost:Connect(function(enterPressed)
                TweenService:Create(InputBox, TweenInfo.new(0.2), {BackgroundColor3 = Color3.fromRGB(15, 17, 23)}):Play()
                if flagName then savedData[flagName] = InputBox.Text; SaveCurrentConfig() end
                callback(InputBox.Text, enterPressed)
            end))

            if maxChars then
                Track(InputBox:GetPropertyChangedSignal("Text"):Connect(function()
                    if #InputBox.Text > maxChars then InputBox.Text = string.sub(InputBox.Text, 1, maxChars) end
                end))
            end

            if flagName then
                table.insert(presetAppliers, function()
                    if savedData[flagName] ~= nil then
                        InputBox.Text = savedData[flagName]
                        callback(savedData[flagName], false)
                    end
                end)
            end
        end

        -- ===== WIDGET: KEYBIND =====
        function Tab:CreateKeybind(labelText, flagName, defaultKey, callback)
            callback = callback or function() end
            local currentKey = defaultKey or Enum.KeyCode.Unknown
            if flagName and savedData[flagName] then
                pcall(function() currentKey = Enum.KeyCode[savedData[flagName]] end)
            end
            local listening = false

            local BindFrame = Instance.new("Frame")
            BindFrame.Size = UDim2.new(1, -10, 0, 38)
            BindFrame.BackgroundColor3 = Color3.fromRGB(24, 28, 38)
            BindFrame.Parent = TabContainer
            local BKC = Instance.new("UICorner"); BKC.CornerRadius = UDim.new(0, cornerRadius); BKC.Parent = BindFrame
            if enableStroke then
                local BKS = Instance.new("UIStroke"); BKS.Color = strokeColor; BKS.Thickness = 1; BKS.Parent = BindFrame
            end
            BindFrame:SetAttribute("SearchText", string.lower(labelText))

            local Label = Instance.new("TextLabel")
            Label.Size = UDim2.new(0.5, -10, 1, 0); Label.Position = UDim2.new(0, 14, 0, 0)
            Label.BackgroundTransparency = 1; Label.Text = labelText
            Label.TextColor3 = Color3.fromRGB(240, 240, 245)
            Label.TextXAlignment = Enum.TextXAlignment.Left
            Label.Font = Enum.Font.Gotham; Label.TextSize = 13
            Label.Parent = BindFrame

            local KeyBtn = Instance.new("TextButton")
            KeyBtn.Size = UDim2.new(0, 100, 0, 26); KeyBtn.Position = UDim2.new(1, -110, 0.5, -13)
            KeyBtn.BackgroundColor3 = Color3.fromRGB(15, 17, 23)
            KeyBtn.Text = currentKey.Name
            KeyBtn.TextColor3 = themeColor
            KeyBtn.Font = Enum.Font.GothamBold; KeyBtn.TextSize = 12
            KeyBtn.Parent = BindFrame
            local KBC = Instance.new("UICorner"); KBC.CornerRadius = UDim.new(0, cornerRadius - 2); KBC.Parent = KeyBtn

            Track(KeyBtn.MouseButton1Click:Connect(function()
                listening = true
                KeyBtn.Text = "Nhấn phím..."
                KeyBtn.TextColor3 = Color3.fromRGB(255, 180, 80)
            end))

            Track(UserInputService.InputBegan:Connect(function(input, gpe)
                if not listening then return end
                if gpe then return end
                if input.UserInputType == Enum.UserInputType.Keyboard then
                    currentKey = input.KeyCode
                    listening = false
                    KeyBtn.Text = currentKey.Name
                    KeyBtn.TextColor3 = themeColor
                    if flagName then savedData[flagName] = currentKey.Name; SaveCurrentConfig() end
                    callback(currentKey)
                end
            end))

            if flagName then
                table.insert(presetAppliers, function()
                    if savedData[flagName] then
                        pcall(function()
                            currentKey = Enum.KeyCode[savedData[flagName]]
                            KeyBtn.Text = currentKey.Name
                            callback(currentKey)
                        end)
                    end
                end)
            end

            return {
                GetKey = function() return currentKey end,
                SetKey = function(k)
                    currentKey = k
                    KeyBtn.Text = k.Name
                    if flagName then savedData[flagName] = k.Name; SaveCurrentConfig() end
                    callback(k)
                end,
            }
        end

        -- ===== WIDGET: PROGRESS BAR =====
        function Tab:CreateProgressBar(labelText, initialPercent)
            local percent = math.clamp(initialPercent or 0, 0, 100)

            local PFrame = Instance.new("Frame")
            PFrame.Size = UDim2.new(1, -10, 0, 46)
            PFrame.BackgroundColor3 = Color3.fromRGB(24, 28, 38)
            PFrame.Parent = TabContainer
            local PFC = Instance.new("UICorner"); PFC.CornerRadius = UDim.new(0, cornerRadius); PFC.Parent = PFrame
            if enableStroke then
                local PFS = Instance.new("UIStroke"); PFS.Color = strokeColor; PFS.Thickness = 1; PFS.Parent = PFrame
            end
            PFrame:SetAttribute("SearchText", string.lower(labelText))

            local Label = Instance.new("TextLabel")
            Label.Size = UDim2.new(0.7, 0, 0, 22); Label.Position = UDim2.new(0, 14, 0, 3)
            Label.BackgroundTransparency = 1; Label.Text = labelText
            Label.TextColor3 = Color3.fromRGB(240, 240, 245)
            Label.TextXAlignment = Enum.TextXAlignment.Left
            Label.Font = Enum.Font.Gotham; Label.TextSize = 13
            Label.Parent = PFrame

            local ValueLabel = Instance.new("TextLabel")
            ValueLabel.Size = UDim2.new(0.3, -14, 0, 22); ValueLabel.Position = UDim2.new(0.7, 0, 0, 3)
            ValueLabel.BackgroundTransparency = 1; ValueLabel.Text = math.floor(percent) .. "%"
            ValueLabel.TextColor3 = themeColor
            ValueLabel.TextXAlignment = Enum.TextXAlignment.Right
            ValueLabel.Font = Enum.Font.GothamBold; ValueLabel.TextSize = 13
            ValueLabel.Parent = PFrame

            local BarBg = Instance.new("Frame")
            BarBg.Size = UDim2.new(1, -28, 0, 8); BarBg.Position = UDim2.new(0, 14, 0, 30)
            BarBg.BackgroundColor3 = Color3.fromRGB(50, 56, 72)
            BarBg.BorderSizePixel = 0; BarBg.Parent = PFrame
            local BBC = Instance.new("UICorner"); BBC.CornerRadius = UDim.new(1, 0); BBC.Parent = BarBg

            local BarFill = Instance.new("Frame")
            BarFill.Size = UDim2.new(percent/100, 0, 1, 0)
            BarFill.BackgroundColor3 = themeColor
            BarFill.BorderSizePixel = 0; BarFill.Parent = BarBg
            local BFC2 = Instance.new("UICorner"); BFC2.CornerRadius = UDim.new(1, 0); BFC2.Parent = BarFill

            local obj = {}
            function obj:Set(p)
                percent = math.clamp(p, 0, 100)
                TweenService:Create(BarFill, TweenInfo.new(0.35, Enum.EasingStyle.Quart), {Size = UDim2.new(percent/100, 0, 1, 0)}):Play()
                ValueLabel.Text = math.floor(percent) .. "%"
            end
            function obj:Get() return percent end
            return obj
        end

        -- ===== WIDGET: IMAGE =====
        function Tab:CreateImage(imageId, height)
            if not imageId then return nil end
            local h = height or 100

            local ImgFrame = Instance.new("Frame")
            ImgFrame.Size = UDim2.new(1, -10, 0, h)
            ImgFrame.BackgroundColor3 = Color3.fromRGB(24, 28, 38)
            ImgFrame.BorderSizePixel = 0
            ImgFrame.ClipsDescendants = true
            ImgFrame.Parent = TabContainer
            local IFC = Instance.new("UICorner"); IFC.CornerRadius = UDim.new(0, cornerRadius); IFC.Parent = ImgFrame
            if enableStroke then
                local IFS = Instance.new("UIStroke"); IFS.Color = strokeColor; IFS.Thickness = 1; IFS.Parent = ImgFrame
            end

            local Img = Instance.new("ImageLabel")
            Img.Size = UDim2.new(1, -16, 1, -16)
            Img.Position = UDim2.new(0, 8, 0, 8)
            Img.BackgroundTransparency = 1
            Img.Image = typeof(imageId) == "number" and ("rbxassetid://" .. imageId) or imageId
            Img.ScaleType = Enum.ScaleType.Fit
            Img.Parent = ImgFrame

            ImgFrame:SetAttribute("SearchText", "image ảnh logo")
            return ImgFrame
        end

        return Tab
    end

    -- ============ BẬT/TẮT VISUAL EFFECTS (V12) ============
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
            if not particleContainer or not particleContainer.Parent then
                createParticles()
            end
        else
            if particleConn then
                particleConn:Disconnect()
                particleConn = nil
            end
            if particleContainer and particleContainer.Parent then
                particleContainer:Destroy()
                particleContainer = nil
            end
            particleList = {}
        end
    end

    function Window:IsBlurEnabled()
        return blurEffect ~= nil and blurEffect.Parent and blurEffect.Size > 0
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
