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

local ParentContainer = (gethui and gethui()) or CoreGui or LocalPlayer:WaitForChild("PlayerGui")

-- BẢNG MÀU MỞ RỘNG PHONG PHÚ (15+ THEMES & MÃ HEX)
local ColorPresets = {
    blue    = Color3.fromRGB(0, 170, 255),
    red     = Color3.fromRGB(245, 65, 85),
    green   = Color3.fromRGB(45, 200, 110),
    purple  = Color3.fromRGB(160, 90, 245),
    orange  = Color3.fromRGB(255, 130, 40),
    yellow  = Color3.fromRGB(250, 200, 40),
    cyan    = Color3.fromRGB(0, 220, 220),
    pink    = Color3.fromRGB(255, 110, 180),
    rose    = Color3.fromRGB(240, 50, 100),
    emerald = Color3.fromRGB(16, 185, 129),
    indigo  = Color3.fromRGB(99, 102, 241),
    violet  = Color3.fromRGB(139, 92, 246),
    amber   = Color3.fromRGB(245, 158, 11),
    teal    = Color3.fromRGB(20, 184, 166),
    neon    = Color3.fromRGB(57, 255, 20),
    dark    = Color3.fromRGB(15, 17, 23)
}

local function ParseColor(val, default)
    if type(val) == "string" then
        if val:sub(1,1) == "#" then
            pcall(function()
                default = Color3.fromHex(val)
            end)
            return default
        elseif ColorPresets[val:lower()] then
            return ColorPresets[val:lower()]
        end
    elseif typeof(val) == "Color3" then
        return val
    end
    return default or ColorPresets.blue
end

local function CopyToClipboard(st)
    if setclipboard then
        setclipboard(st)
    elseif toclipboard then
        toclipboard(st)
    elseif Syn and Syn.set_thread_identity then
        setclipboard(st)
    end
end

function AlatferaLib.CreateWindow(config)
    config = config or {}
    local hubTitle = config.Title or "Alatfera Script"
    local hubVersion = config.Version or "v9.0"
    local showPlaytime = (config.ShowPlaytime == nil) and true or config.ShowPlaytime
    local scriptNameText = config.LoadingScriptName or "Alatfera Script"
    local statusText = config.LoadingStatus or "Đang tải giao diện..."
    
    local cornerRadius = config.CornerRadius or 12
    local themeColor = ParseColor(config.ThemeColor, ColorPresets.blue)
    local enableStroke = (config.Stroke == nil) and true or config.Stroke
    local strokeColor = ParseColor(config.StrokeColor, Color3.fromRGB(45, 50, 65))

    -- KEY SYSTEM
    local useKeySystem = config.KeySystem or false
    local correctKey = config.Key or "AlatferaKey123"
    local discordLink = config.DiscordLink or "https://discord.gg/alatfera"
    local maxAttempts = config.MaxAttempts or 10
    local remainingAttempts = maxAttempts

    -- CONFIG FILE
    local configFile = config.ConfigFile or "Alatfera_Config.json"
    local savedData = {}
    if readfile and isfile and isfile(configFile) then
        pcall(function()
            savedData = HttpService:JSONDecode(readfile(configFile))
        end)
    end

    local function SaveCurrentConfig()
        if writefile then
            pcall(function()
                writefile(configFile, HttpService:JSONEncode(savedData))
            end)
        end
    end

    if ParentContainer:FindFirstChild("AlatferaGui") then
        ParentContainer["AlatferaGui"]:Destroy()
    end

    local ScreenGui = Instance.new("ScreenGui")
    ScreenGui.Name = "AlatferaGui"
    ScreenGui.ResetOnSpawn = false
    ScreenGui.Parent = ParentContainer

    -- ----------------------------------------------------
    -- 0. XÁC NHẬN KEY SYSTEM
    -- ----------------------------------------------------
    if useKeySystem then
        local keyPassed = false
        if savedData["_SavedKey"] == correctKey then
            keyPassed = true
        end

        if not keyPassed then
            local KeyFrame = Instance.new("Frame")
            KeyFrame.Name = "KeyFrame"
            KeyFrame.Size = UDim2.new(0, 360, 0, 220)
            KeyFrame.Position = UDim2.new(0.5, -180, 0.5, -110)
            KeyFrame.BackgroundColor3 = Color3.fromRGB(15, 17, 23)
            KeyFrame.BorderSizePixel = 0
            KeyFrame.Parent = ScreenGui

            local KeyCorner = Instance.new("UICorner")
            KeyCorner.CornerRadius = UDim.new(0, cornerRadius)
            KeyCorner.Parent = KeyFrame

            if enableStroke then
                local KStroke = Instance.new("UIStroke")
                KStroke.Color = strokeColor
                KStroke.Thickness = 1.5
                KStroke.Parent = KeyFrame
            end

            local KTitle = Instance.new("TextLabel")
            KTitle.Size = UDim2.new(1, 0, 0, 34)
            KTitle.Position = UDim2.new(0, 0, 0, 12)
            KTitle.BackgroundTransparency = 1
            KTitle.Text = "🔑 XÁC NHẬN KEY"
            KTitle.TextColor3 = themeColor
            KTitle.Font = Enum.Font.GothamBold
            KTitle.TextSize = 17
            KTitle.Parent = KeyFrame

            local KSub = Instance.new("TextLabel")
            KSub.Size = UDim2.new(1, -20, 0, 20)
            KSub.Position = UDim2.new(0, 10, 0, 42)
            KSub.BackgroundTransparency = 1
            KSub.Text = "Nhập Key để truy cập " .. hubTitle
            KSub.TextColor3 = Color3.fromRGB(160, 165, 180)
            KSub.Font = Enum.Font.Gotham
            KSub.TextSize = 13
            KSub.Parent = KeyFrame

            local KeyInput = Instance.new("TextBox")
            KeyInput.Size = UDim2.new(1, -30, 0, 36)
            KeyInput.Position = UDim2.new(0, 15, 0, 70)
            KeyInput.BackgroundColor3 = Color3.fromRGB(24, 27, 36)
            KeyInput.Text = ""
            KeyInput.PlaceholderText = "Dán Key tại đây..."
            KeyInput.TextColor3 = Color3.fromRGB(240, 240, 245)
            KeyInput.PlaceholderColor3 = Color3.fromRGB(110, 115, 130)
            KeyInput.Font = Enum.Font.Gotham
            KeyInput.TextSize = 13
            KeyInput.Parent = KeyFrame

            local InputCorner = Instance.new("UICorner")
            InputCorner.CornerRadius = UDim.new(0, cornerRadius - 2)
            InputCorner.Parent = KeyInput

            local StatusLabel = Instance.new("TextLabel")
            StatusLabel.Size = UDim2.new(1, -20, 0, 18)
            StatusLabel.Position = UDim2.new(0, 10, 0, 110)
            StatusLabel.BackgroundTransparency = 1
            StatusLabel.Text = "Còn lại " .. remainingAttempts .. " lần thử"
            StatusLabel.TextColor3 = Color3.fromRGB(140, 145, 160)
            StatusLabel.Font = Enum.Font.Gotham
            StatusLabel.TextSize = 12
            StatusLabel.Parent = KeyFrame

            local SubmitBtn = Instance.new("TextButton")
            SubmitBtn.Size = UDim2.new(0.46, 0, 0, 34)
            SubmitBtn.Position = UDim2.new(0, 15, 0, 136)
            SubmitBtn.BackgroundColor3 = themeColor
            SubmitBtn.Text = "Xác Nhận"
            SubmitBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
            SubmitBtn.Font = Enum.Font.GothamBold
            SubmitBtn.TextSize = 13
            SubmitBtn.Parent = KeyFrame

            local SubCorner = Instance.new("UICorner")
            SubCorner.CornerRadius = UDim.new(0, cornerRadius - 2)
            SubCorner.Parent = SubmitBtn

            local DiscordBtn = Instance.new("TextButton")
            DiscordBtn.Size = UDim2.new(0.46, 0, 0, 34)
            DiscordBtn.Position = UDim2.new(0.54, -5, 0, 136)
            DiscordBtn.BackgroundColor3 = Color3.fromRGB(70, 80, 200)
            DiscordBtn.Text = "📋 Copy Discord"
            DiscordBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
            DiscordBtn.Font = Enum.Font.GothamBold
            DiscordBtn.TextSize = 12
            DiscordBtn.Parent = KeyFrame

            local DiscCorner = Instance.new("UICorner")
            DiscCorner.CornerRadius = UDim.new(0, cornerRadius - 2)
            DiscCorner.Parent = DiscordBtn

            DiscordBtn.MouseButton1Click:Connect(function()
                CopyToClipboard(discordLink)
                DiscordBtn.Text = "✓ Đã Copy!"
                task.wait(1.5)
                DiscordBtn.Text = "📋 Copy Discord"
            end)

            local dragK, startK, posK
            KeyFrame.InputBegan:Connect(function(input)
                if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
                    dragK = true; startK = input.Position; posK = KeyFrame.Position
                end
            end)
            UserInputService.InputChanged:Connect(function(input)
                if dragK and (input.UserInputType == Enum.UserInputType.MouseMovement or input.UserInputType == Enum.UserInputType.Touch) then
                    local delta = input.Position - startK
                    KeyFrame.Position = UDim2.new(posK.X.Scale, posK.X.Offset + delta.X, posK.Y.Scale, posK.Y.Offset + delta.Y)
                end
            end)
            KeyFrame.InputEnded:Connect(function(input)
                if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then dragK = false end
            end)

            local function VerifyKey()
                local userKey = KeyInput.Text
                if userKey == correctKey then
                    StatusLabel.Text = "✓ Key hợp lệ! Đang mở..."
                    StatusLabel.TextColor3 = Color3.fromRGB(45, 200, 110)
                    savedData["_SavedKey"] = correctKey
                    SaveCurrentConfig()
                    task.wait(0.8)
                    KeyFrame:Destroy()
                    keyPassed = true
                else
                    remainingAttempts = remainingAttempts - 1
                    if remainingAttempts <= 0 then
                        StatusLabel.Text = "❌ Sai quá " .. maxAttempts .. " lần!"
                        StatusLabel.TextColor3 = Color3.fromRGB(245, 65, 85)
                        task.wait(1)
                        LocalPlayer:Kick("Alatfera Security: Nhập sai Key quá " .. maxAttempts .. " lần!")
                    else
                        StatusLabel.Text = "❌ Sai Key! Còn " .. remainingAttempts .. "/" .. maxAttempts .. " lần"
                        StatusLabel.TextColor3 = Color3.fromRGB(245, 65, 85)
                    end
                end
            end

            SubmitBtn.MouseButton1Click:Connect(VerifyKey)
            KeyInput.FocusLost:Connect(function(enterPressed)
                if enterPressed then VerifyKey() end
            end)

            repeat task.wait(0.1) until keyPassed
        end
    end

    -- ----------------------------------------------------
    -- 1. KHUNG LOADING
    -- ----------------------------------------------------
    local LoadingFrame = Instance.new("Frame")
    LoadingFrame.Name = "LoadingFrame"
    LoadingFrame.Size = UDim2.new(0, 280, 0, 100)
    LoadingFrame.Position = UDim2.new(0.5, -140, 0.5, -50)
    LoadingFrame.BackgroundColor3 = Color3.fromRGB(15, 17, 23)
    LoadingFrame.BorderSizePixel = 0
    LoadingFrame.Parent = ScreenGui

    local LoadCorner = Instance.new("UICorner")
    LoadCorner.CornerRadius = UDim.new(0, cornerRadius)
    LoadCorner.Parent = LoadingFrame

    if enableStroke then
        local LStroke = Instance.new("UIStroke")
        LStroke.Color = strokeColor
        LStroke.Thickness = 1.5
        LStroke.Parent = LoadingFrame
    end

    local LoadTitle = Instance.new("TextLabel")
    LoadTitle.Size = UDim2.new(1, 0, 0, 30)
    LoadTitle.Position = UDim2.new(0, 0, 0, 12)
    LoadTitle.BackgroundTransparency = 1
    LoadTitle.Text = scriptNameText
    LoadTitle.TextColor3 = themeColor
    LoadTitle.Font = Enum.Font.GothamBold
    LoadTitle.TextSize = 17
    LoadTitle.Parent = LoadingFrame

    local LoadStatus = Instance.new("TextLabel")
    LoadStatus.Size = UDim2.new(1, 0, 0, 20)
    LoadStatus.Position = UDim2.new(0, 0, 0, 40)
    LoadStatus.BackgroundTransparency = 1
    LoadStatus.Text = statusText
    LoadStatus.TextColor3 = Color3.fromRGB(160, 165, 180)
    LoadStatus.Font = Enum.Font.Gotham
    LoadStatus.TextSize = 12
    LoadStatus.Parent = LoadingFrame

    local BarBackground = Instance.new("Frame")
    BarBackground.Size = UDim2.new(0.8, 0, 0, 5)
    BarBackground.Position = UDim2.new(0.1, 0, 0, 72)
    BarBackground.BackgroundColor3 = Color3.fromRGB(30, 34, 46)
    BarBackground.BorderSizePixel = 0
    BarBackground.Parent = LoadingFrame

    local BarCorner = Instance.new("UICorner")
    BarCorner.CornerRadius = UDim.new(1, 0)
    BarCorner.Parent = BarBackground

    local BarFill = Instance.new("Frame")
    BarFill.Size = UDim2.new(0, 0, 1, 0)
    BarFill.BackgroundColor3 = themeColor
    BarFill.BorderSizePixel = 0
    BarFill.Parent = BarBackground

    local FillCorner = Instance.new("UICorner")
    FillCorner.CornerRadius = UDim.new(1, 0)
    FillCorner.Parent = BarFill

    TweenService:Create(BarFill, TweenInfo.new(1.0, Enum.EasingStyle.Quart, Enum.EasingDirection.Out), {Size = UDim2.new(1, 0, 1, 0)}):Play()
    task.wait(1.1)
    LoadingFrame:Destroy()

    -- ----------------------------------------------------
    -- 2. KHUNG CHÍNH (GUI BỰ 75% RỘNG, 80% CAO)
    -- ----------------------------------------------------
    local Window = {}
    Window.ThemeColor = themeColor

    local MainFrame = Instance.new("Frame")
    MainFrame.Name = "MainFrame"
    MainFrame.Size = UDim2.new(0.75, 0, 0.8, 0)
    MainFrame.Position = UDim2.new(0.125, 0, 0.1, 0)
    MainFrame.BackgroundColor3 = Color3.fromRGB(15, 17, 23)
    MainFrame.BorderSizePixel = 0
    MainFrame.ClipsDescendants = true
    MainFrame.Parent = ScreenGui

    local MainCorner = Instance.new("UICorner")
    MainCorner.CornerRadius = UDim.new(0, cornerRadius)
    MainCorner.Parent = MainFrame

    if enableStroke then
        local MStroke = Instance.new("UIStroke")
        MStroke.Color = strokeColor
        MStroke.Thickness = 2
        MStroke.Parent = MainFrame
    end

    -- TOPBAR
    local Topbar = Instance.new("Frame")
    Topbar.Name = "Topbar"
    Topbar.Size = UDim2.new(1, 0, 0, 46)
    Topbar.BackgroundColor3 = Color3.fromRGB(20, 23, 32)
    Topbar.BorderSizePixel = 0
    Topbar.Parent = MainFrame

    local TopbarCorner = Instance.new("UICorner")
    TopbarCorner.CornerRadius = UDim.new(0, cornerRadius)
    TopbarCorner.Parent = Topbar

    local hexString = string.format("#%02X%02X%02X", themeColor.R*255, themeColor.G*255, themeColor.B*255)
    
    -- TIÊU ĐỀ
    local TitleLabel = Instance.new("TextLabel")
    TitleLabel.Size = UDim2.new(0.28, 0, 1, 0)
    TitleLabel.Position = UDim2.new(0, 18, 0, 0)
    TitleLabel.BackgroundTransparency = 1
    TitleLabel.Text = hubTitle .. (hubVersion ~= "" and ("  <font color=\"" .. hexString .. "\">" .. hubVersion .. "</font>") or "")
    TitleLabel.RichText = true
    TitleLabel.TextColor3 = Color3.fromRGB(255, 255, 255)
    TitleLabel.TextXAlignment = Enum.TextXAlignment.Left
    TitleLabel.Font = Enum.Font.GothamBold
    TitleLabel.TextSize = 15
    TitleLabel.Parent = Topbar

    -- THÔNG SỐ FPS & PING (KẾ BÊN TÊN HUB)
    local StatsLabel = Instance.new("TextLabel")
    StatsLabel.Name = "StatsLabel"
    StatsLabel.Size = UDim2.new(0.22, 0, 1, 0)
    StatsLabel.Position = UDim2.new(0.28, 10, 0, 0)
    StatsLabel.BackgroundTransparency = 1
    StatsLabel.Text = "⚡ 60 FPS | 0 ms"
    StatsLabel.TextColor3 = Color3.fromRGB(150, 155, 170)
    StatsLabel.TextXAlignment = Enum.TextXAlignment.Left
    StatsLabel.Font = Enum.Font.Gotham
    StatsLabel.TextSize = 12
    StatsLabel.Parent = Topbar

    -- BỘ ĐO FPS & PING TỰ ĐỘNG
    local frameCount = 0
    local lastTime = tick()
    RunService.RenderStepped:Connect(function()
        frameCount = frameCount + 1
        local currentTime = tick()
        if currentTime - lastTime >= 1 then
            local fps = math.floor(frameCount / (currentTime - lastTime))
            local ping = math.floor(Stats.Network.ServerStatsItem["Data Ping"]:GetValue())
            StatsLabel.Text = string.format("⚡ %d FPS | %d ms", fps, ping)
            
            frameCount = 0
            lastTime = currentTime
        end
    end)

    -- NÚT ✕ (ĐÓNG)
    local CloseBtn = Instance.new("TextButton")
    CloseBtn.Size = UDim2.new(0, 34, 0, 34)
    CloseBtn.Position = UDim2.new(1, -42, 0, 6)
    CloseBtn.BackgroundTransparency = 1
    CloseBtn.Text = "✕"
    CloseBtn.TextColor3 = Color3.fromRGB(245, 70, 90)
    CloseBtn.TextSize = 17
    CloseBtn.Font = Enum.Font.GothamBold
    CloseBtn.Parent = Topbar

    -- Ô TÌM KIẾM
    local SearchBox = Instance.new("TextBox")
    SearchBox.Name = "SearchBox"
    SearchBox.Size = UDim2.new(0, 160, 0, 28)
    SearchBox.Position = UDim2.new(1, -305, 0, 9)
    SearchBox.BackgroundColor3 = Color3.fromRGB(12, 14, 19)
    SearchBox.Text = ""
    SearchBox.PlaceholderText = "🔍 Tìm kiếm..."
    SearchBox.TextColor3 = Color3.fromRGB(240, 240, 245)
    SearchBox.PlaceholderColor3 = Color3.fromRGB(110, 115, 130)
    SearchBox.Font = Enum.Font.Gotham
    SearchBox.TextSize = 12
    SearchBox.Parent = Topbar

    local SearchCorner = Instance.new("UICorner")
    SearchCorner.CornerRadius = UDim.new(0, 8)
    SearchCorner.Parent = SearchBox

    if enableStroke then
        local SrchStroke = Instance.new("UIStroke")
        SrchStroke.Color = strokeColor
        SrchStroke.Thickness = 1
        SrchStroke.Parent = SearchBox
    end

    -- NÚT ↑ / ↓ (THU NHỎ)
    local MinimizeBtn = Instance.new("TextButton")
    MinimizeBtn.Size = UDim2.new(0, 34, 0, 34)
    MinimizeBtn.Position = UDim2.new(1, -78, 0, 6)
    MinimizeBtn.BackgroundTransparency = 1
    MinimizeBtn.Text = "↑"
    MinimizeBtn.TextColor3 = Color3.fromRGB(180, 185, 200)
    MinimizeBtn.TextSize = 18
    MinimizeBtn.Font = Enum.Font.GothamBold
    MinimizeBtn.Parent = Topbar

    -- NÚT ? (HƯỚNG DẪN)
    local HelpBtn = Instance.new("TextButton")
    HelpBtn.Size = UDim2.new(0, 34, 0, 34)
    HelpBtn.Position = UDim2.new(1, -114, 0, 6)
    HelpBtn.BackgroundTransparency = 1
    HelpBtn.Text = "?"
    HelpBtn.TextColor3 = Color3.fromRGB(180, 185, 200)
    HelpBtn.TextSize = 17
    HelpBtn.Font = Enum.Font.GothamBold
    HelpBtn.Parent = Topbar

    -- NÚT ⌨ (KEYBIND)
    local KeybindBtn = Instance.new("TextButton")
    KeybindBtn.Size = UDim2.new(0, 34, 0, 34)
    KeybindBtn.Position = UDim2.new(1, -150, 0, 6)
    KeybindBtn.BackgroundTransparency = 1
    KeybindBtn.Text = "⌨"
    KeybindBtn.TextColor3 = Color3.fromRGB(180, 185, 200)
    KeybindBtn.TextSize = 17
    KeybindBtn.Font = Enum.Font.GothamBold
    KeybindBtn.Parent = Topbar

    -- NÚT "⚡ MỞ GUI" (CỐ ĐỊNH Ở GIỮA TRÊN)
    local OpenBtnFrame = Instance.new("TextButton")
    OpenBtnFrame.Name = "OpenGuiBtn"
    OpenBtnFrame.Size = UDim2.new(0, 120, 0, 32)
    OpenBtnFrame.Position = UDim2.new(0.5, -60, 0, 8)
    OpenBtnFrame.BackgroundColor3 = Color3.fromRGB(22, 25, 34)
    OpenBtnFrame.BackgroundTransparency = 0.35
    OpenBtnFrame.Text = "⚡ Mở GUI"
    OpenBtnFrame.TextColor3 = themeColor
    OpenBtnFrame.Font = Enum.Font.GothamBold
    OpenBtnFrame.TextSize = 13
    OpenBtnFrame.Visible = false
    OpenBtnFrame.Parent = ScreenGui

    local OpenCorner = Instance.new("UICorner")
    OpenCorner.CornerRadius = UDim.new(0, 16)
    OpenCorner.Parent = OpenBtnFrame

    if enableStroke then
        local OStroke = Instance.new("UIStroke")
        OStroke.Color = themeColor
        OStroke.Transparency = 0.4
        OStroke.Thickness = 1
        OStroke.Parent = OpenBtnFrame
    end

    CloseBtn.MouseButton1Click:Connect(function()
        MainFrame.Visible = false
        OpenBtnFrame.Visible = true
    end)

    OpenBtnFrame.MouseButton1Click:Connect(function()
        MainFrame.Visible = true
        OpenBtnFrame.Visible = false
    end)

    local isMinimized = false
    MinimizeBtn.MouseButton1Click:Connect(function()
        isMinimized = not isMinimized
        if isMinimized then
            MinimizeBtn.Text = "↓"
            MainFrame:TweenSize(UDim2.new(0.75, 0, 0, 46), Enum.EasingDirection.Out, Enum.EasingStyle.Quart, 0.25, true)
        else
            MinimizeBtn.Text = "↑"
            MainFrame:TweenSize(UDim2.new(0.75, 0, 0.8, 0), Enum.EasingDirection.Out, Enum.EasingStyle.Quart, 0.25, true)
        end
    end)

    HelpBtn.MouseButton1Click:Connect(function()
        local helpText = config.HelpText or "• Kéo thanh topbar để di chuyển GUI.\n• Ô tìm kiếm ở kế nút X để lọc nhanh.\n• Nút [⚡ Mở GUI] cố định ở trên giữa."
        Window:Notify({
            Title = "Hướng Dẫn",
            Text = helpText,
            Duration = 5
        })
    end)

    local currentToggleKey = config.ToggleKey or Enum.KeyCode.RightControl
    KeybindBtn.MouseButton1Click:Connect(function()
        Window:Notify({
            Title = "Phím Tắt",
            Text = "Phím ẩn/hiện: " .. tostring(currentToggleKey.Name),
            Duration = 4
        })
    end)

    UserInputService.InputBegan:Connect(function(input, gpe)
        if not gpe and input.KeyCode == currentToggleKey then
            MainFrame.Visible = not MainFrame.Visible
            OpenBtnFrame.Visible = not MainFrame.Visible
        end
    end)

    -- KÉO THẢ MAINFRAME
    local dragging, dragStart, startPos
    Topbar.InputBegan:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
            dragging = true; dragStart = input.Position; startPos = MainFrame.Position
        end
    end)
    UserInputService.InputChanged:Connect(function(input)
        if dragging and (input.UserInputType == Enum.UserInputType.MouseMovement or input.UserInputType == Enum.UserInputType.Touch) then
            local delta = input.Position - dragStart
            MainFrame.Position = UDim2.new(startPos.X.Scale, startPos.X.Offset + delta.X, startPos.Y.Scale, startPos.Y.Offset + delta.Y)
        end
    end)
    Topbar.InputEnded:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then dragging = false end
    end)

    -- ----------------------------------------------------
    -- 3. SIDEBAR
    -- ----------------------------------------------------
    local Sidebar = Instance.new("ScrollingFrame")
    Sidebar.Name = "Sidebar"
    Sidebar.Size = UDim2.new(0, 160, 1, showPlaytime and -86 or -58)
    Sidebar.Position = UDim2.new(0, 10, 0, 54)
    Sidebar.BackgroundColor3 = Color3.fromRGB(19, 21, 28)
    Sidebar.BorderSizePixel = 0
    Sidebar.ScrollBarThickness = 2
    Sidebar.ScrollBarImageColor3 = themeColor
    Sidebar.Parent = MainFrame

    local SideCorner = Instance.new("UICorner")
    SideCorner.CornerRadius = UDim.new(0, cornerRadius)
    SideCorner.Parent = Sidebar

    if enableStroke then
        local SStroke = Instance.new("UIStroke")
        SStroke.Color = strokeColor
        SStroke.Thickness = 1
        SStroke.Parent = Sidebar
    end

    local TabListLayout = Instance.new("UIListLayout")
    TabListLayout.Padding = UDim.new(0, 6)
    TabListLayout.SortOrder = Enum.SortOrder.LayoutOrder
    TabListLayout.Parent = Sidebar

    TabListLayout:GetPropertyChangedSignal("AbsoluteContentSize"):Connect(function()
        Sidebar.CanvasSize = UDim2.new(0, 0, 0, TabListLayout.AbsoluteContentSize.Y + 10)
    end)

    if showPlaytime then
        local PlaytimeFrame = Instance.new("Frame")
        PlaytimeFrame.Size = UDim2.new(0, 160, 0, 24)
        PlaytimeFrame.Position = UDim2.new(0, 10, 1, -28)
        PlaytimeFrame.BackgroundColor3 = Color3.fromRGB(19, 21, 28)
        PlaytimeFrame.BorderSizePixel = 0
        PlaytimeFrame.Parent = MainFrame

        local PtCorner = Instance.new("UICorner")
        PtCorner.CornerRadius = UDim.new(0, cornerRadius)
        PtCorner.Parent = PlaytimeFrame

        if enableStroke then
            local PStroke = Instance.new("UIStroke")
            PStroke.Color = strokeColor
            PStroke.Thickness = 1
            PStroke.Parent = PlaytimeFrame
        end

        local PlaytimeLabel = Instance.new("TextLabel")
        PlaytimeLabel.Size = UDim2.new(1, 0, 1, 0)
        PlaytimeLabel.BackgroundTransparency = 1
        PlaytimeLabel.Text = "⏱️ 00:00:00"
        PlaytimeLabel.TextColor3 = Color3.fromRGB(150, 155, 170)
        PlaytimeLabel.Font = Enum.Font.Gotham
        PlaytimeLabel.TextSize = 12
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
    ContentArea.Name = "ContentArea"
    ContentArea.Size = UDim2.new(1, -188, 1, -62)
    ContentArea.Position = UDim2.new(0, 178, 0, 54)
    ContentArea.BackgroundTransparency = 1
    ContentArea.Parent = MainFrame

    local FirstTab = true

    SearchBox:GetPropertyChangedSignal("Text"):Connect(function()
        local query = string.lower(SearchBox.Text)
        for _, container in pairs(ContentArea:GetChildren()) do
            if container:IsA("ScrollingFrame") then
                for _, elem in pairs(container:GetChildren()) do
                    if elem:IsA("Frame") or elem:IsA("TextButton") or elem:IsA("TextLabel") then
                        local btnText = ""
                        local lbl = elem:FindFirstChildOfClass("TextLabel")
                        if lbl then
                            btnText = string.lower(lbl.Text)
                        elseif elem:IsA("TextButton") or elem:IsA("TextLabel") then
                            btnText = string.lower(elem.Text)
                        end
                        
                        if query == "" or string.find(btnText, query) then
                            elem.Visible = true
                        else
                            elem.Visible = false
                        end
                    end
                end
            end
        end
    end)

    -- ----------------------------------------------------
    -- 4. HỆ THỐNG THÔNG BÁO (NOTIFY)
    -- ----------------------------------------------------
    local NotifContainer = Instance.new("Frame")
    NotifContainer.Name = "NotifContainer"
    NotifContainer.Size = UDim2.new(0, 250, 1, -20)
    NotifContainer.Position = UDim2.new(1, -260, 0, 10)
    NotifContainer.BackgroundTransparency = 1
    NotifContainer.Parent = ScreenGui

    local NotifLayout = Instance.new("UIListLayout")
    NotifLayout.VerticalAlignment = Enum.VerticalAlignment.Bottom
    NotifLayout.Padding = UDim.new(0, 6)
    NotifLayout.SortOrder = Enum.SortOrder.LayoutOrder
    NotifLayout.Parent = NotifContainer

    function Window:Notify(notifConfig)
        notifConfig = notifConfig or {}
        local title = notifConfig.Title or "Thông Báo"
        local text = notifConfig.Text or ""
        local duration = notifConfig.Duration or 3
        local nColor = ParseColor(notifConfig.Color, themeColor)
        local pos = notifConfig.Position

        local Item = Instance.new("Frame")
        Item.Size = UDim2.new(1, 0, 0, 54)
        Item.BackgroundColor3 = Color3.fromRGB(22, 25, 34)
        Item.BorderSizePixel = 0
        Item.Parent = NotifContainer

        local ItemCorner = Instance.new("UICorner")
        ItemCorner.CornerRadius = UDim.new(0, cornerRadius)
        ItemCorner.Parent = Item

        if enableStroke then
            local NStroke = Instance.new("UIStroke")
            NStroke.Color = strokeColor
            NStroke.Thickness = 1
            NStroke.Parent = Item
        end

        local Bar = Instance.new("Frame")
        Bar.Size = UDim2.new(0, 4, 1, 0)
        Bar.BackgroundColor3 = nColor
        Bar.BorderSizePixel = 0
        Bar.Parent = Item

        local BarCorn = Instance.new("UICorner")
        BarCorn.CornerRadius = UDim.new(0, cornerRadius)
        BarCorn.Parent = Bar

        local NTitle = Instance.new("TextLabel")
        NTitle.Size = UDim2.new(1, -12, 0, 20)
        NTitle.Position = UDim2.new(0, 12, 0, 4)
        NTitle.BackgroundTransparency = 1
        NTitle.Text = title
        NTitle.TextColor3 = Color3.fromRGB(255, 255, 255)
        NTitle.Font = Enum.Font.GothamBold
        NTitle.TextSize = 13
        NTitle.TextXAlignment = Enum.TextXAlignment.Left
        NTitle.Parent = Item

        local NText = Instance.new("TextLabel")
        NText.Size = UDim2.new(1, -12, 0, 22)
        NText.Position = UDim2.new(0, 12, 0, 24)
        NText.BackgroundTransparency = 1
        NText.Text = text
        NText.TextColor3 = Color3.fromRGB(160, 165, 180)
        NText.Font = Enum.Font.Gotham
        NText.TextSize = 12
        NText.TextXAlignment = Enum.TextXAlignment.Left
        NText.Parent = Item

        if pos == "left" then
            NotifContainer.Position = UDim2.new(0, 10, 0, 10)
        elseif pos == "center" then
            NotifContainer.Position = UDim2.new(0.5, -125, 0, 10)
        else
            NotifContainer.Position = UDim2.new(1, -260, 0, 10)
        end

        task.spawn(function()
            task.wait(duration)
            TweenService:Create(Item, TweenInfo.new(0.3), {BackgroundTransparency = 1}):Play()
            TweenService:Create(NTitle, TweenInfo.new(0.3), {TextTransparency = 1}):Play()
            TweenService:Create(NText, TweenInfo.new(0.3), {TextTransparency = 1}):Play()
            task.wait(0.3)
            Item:Destroy()
        end)
    end

    -- ----------------------------------------------------
    -- 5. TẠO TAB & WIDGETS
    -- ----------------------------------------------------
    function Window:CreateTab(tabName)
        local Tab = {}

        local TabButton = Instance.new("TextButton")
        TabButton.Size = UDim2.new(1, -6, 0, 36)
        TabButton.BackgroundColor3 = FirstTab and Color3.fromRGB(28, 33, 46) or Color3.fromRGB(19, 21, 28)
        TabButton.Text = "  " .. tabName
        TabButton.TextColor3 = FirstTab and Color3.fromRGB(255, 255, 255) or Color3.fromRGB(140, 145, 160)
        TabButton.TextXAlignment = Enum.TextXAlignment.Left
        TabButton.Font = Enum.Font.GothamBold
        TabButton.TextSize = 13
        TabButton.Parent = Sidebar

        local TabBtnCorner = Instance.new("UICorner")
        TabBtnCorner.CornerRadius = UDim.new(0, cornerRadius)
        TabBtnCorner.Parent = TabButton

        if enableStroke then
            local TBStroke = Instance.new("UIStroke")
            TBStroke.Color = strokeColor
            TBStroke.Thickness = 1
            TBStroke.Parent = TabButton
        end

        local TabContainer = Instance.new("ScrollingFrame")
        TabContainer.Name = "Container_" .. tabName
        TabContainer.Size = UDim2.new(1, 0, 1, 0)
        TabContainer.BackgroundTransparency = 1
        TabContainer.BorderSizePixel = 0
        TabContainer.ScrollBarThickness = 3
        TabContainer.ScrollBarImageColor3 = themeColor
        TabContainer.Visible = FirstTab
        TabContainer.Parent = ContentArea

        local ContainerLayout = Instance.new("UIListLayout")
        ContainerLayout.Padding = UDim.new(0, 10)
        ContainerLayout.SortOrder = Enum.SortOrder.LayoutOrder
        ContainerLayout.Parent = TabContainer

        ContainerLayout:GetPropertyChangedSignal("AbsoluteContentSize"):Connect(function()
            TabContainer.CanvasSize = UDim2.new(0, 0, 0, ContainerLayout.AbsoluteContentSize.Y + 15)
        end)

        TabButton.MouseButton1Click:Connect(function()
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
        end)

        FirstTab = false

        function Tab:CreateLabel(text)
            local Label = Instance.new("TextLabel")
            Label.Size = UDim2.new(1, -10, 0, 28)
            Label.BackgroundColor3 = Color3.fromRGB(22, 25, 34)
            Label.Text = "  " .. text
            Label.TextColor3 = Color3.fromRGB(170, 175, 190)
            Label.TextXAlignment = Enum.TextXAlignment.Left
            Label.Font = Enum.Font.Gotham
            Label.TextSize = 12
            Label.Parent = TabContainer

            local LblCorner = Instance.new("UICorner")
            LblCorner.CornerRadius = UDim.new(0, cornerRadius)
            LblCorner.Parent = Label

            if enableStroke then
                local LbStroke = Instance.new("UIStroke")
                LbStroke.Color = strokeColor
                LbStroke.Thickness = 1
                LbStroke.Parent = Label
            end
        end

        function Tab:CreateSection(text)
            local SectionFrame = Instance.new("Frame")
            SectionFrame.Size = UDim2.new(1, -10, 0, 24)
            SectionFrame.BackgroundTransparency = 1
            SectionFrame.Parent = TabContainer

            local SecText = Instance.new("TextLabel")
            SecText.Size = UDim2.new(1, 0, 1, 0)
            SecText.BackgroundTransparency = 1
            SecText.Text = "───  " .. string.upper(text or "SECTION") .. "  ───"
            SecText.TextColor3 = themeColor
            SecText.Font = Enum.Font.GothamBold
            SecText.TextSize = 12
            SecText.Parent = SectionFrame
        end

        function Tab:CreateButton(btnText, callback)
            callback = callback or function() end
            local Button = Instance.new("TextButton")
            Button.Size = UDim2.new(1, -10, 0, 38)
            Button.BackgroundColor3 = Color3.fromRGB(24, 28, 38)
            Button.Text = btnText
            Button.TextColor3 = Color3.fromRGB(240, 240, 245)
            Button.Font = Enum.Font.Gotham
            Button.TextSize = 13
            Button.Parent = TabContainer

            local BtnCorner = Instance.new("UICorner")
            BtnCorner.CornerRadius = UDim.new(0, cornerRadius)
            BtnCorner.Parent = Button

            if enableStroke then
                local BStroke = Instance.new("UIStroke")
                BStroke.Color = strokeColor
                BStroke.Thickness = 1
                BStroke.Parent = Button
            end

            Button.MouseButton1Click:Connect(function()
                TweenService:Create(Button, TweenInfo.new(0.1), {BackgroundColor3 = themeColor}):Play()
                task.wait(0.1)
                TweenService:Create(Button, TweenInfo.new(0.1), {BackgroundColor3 = Color3.fromRGB(24, 28, 38)}):Play()
                callback()
            end)
        end

        function Tab:CreateToggle(toggleText, flagName, defaultState, callback)
            callback = callback or function() end
            
            if flagName and savedData[flagName] ~= nil then
                defaultState = savedData[flagName]
            end
            local state = defaultState or false

            local ToggleFrame = Instance.new("TextButton")
            ToggleFrame.Size = UDim2.new(1, -10, 0, 38)
            ToggleFrame.BackgroundColor3 = Color3.fromRGB(24, 28, 38)
            ToggleFrame.Text = ""
            ToggleFrame.Parent = TabContainer

            local FrameCorner = Instance.new("UICorner")
            FrameCorner.CornerRadius = UDim.new(0, cornerRadius)
            FrameCorner.Parent = ToggleFrame

            if enableStroke then
                local TStroke = Instance.new("UIStroke")
                TStroke.Color = strokeColor
                TStroke.Thickness = 1
                TStroke.Parent = ToggleFrame
            end

            local Label = Instance.new("TextLabel")
            Label.Size = UDim2.new(1, -55, 1, 0)
            Label.Position = UDim2.new(0, 14, 0, 0)
            Label.BackgroundTransparency = 1
            Label.Text = toggleText
            Label.TextColor3 = Color3.fromRGB(240, 240, 245)
            Label.TextXAlignment = Enum.TextXAlignment.Left
            Label.Font = Enum.Font.Gotham
            Label.TextSize = 13
            Label.Parent = ToggleFrame

            local Indicator = Instance.new("Frame")
            Indicator.Size = UDim2.new(0, 40, 0, 20)
            Indicator.Position = UDim2.new(1, -50, 0.5, -10)
            Indicator.BackgroundColor3 = state and themeColor or Color3.fromRGB(50, 56, 72)
            Indicator.Parent = ToggleFrame

            local IndCorner = Instance.new("UICorner")
            IndCorner.CornerRadius = UDim.new(1, 0)
            IndCorner.Parent = Indicator

            ToggleFrame.MouseButton1Click:Connect(function()
                state = not state
                TweenService:Create(Indicator, TweenInfo.new(0.2), {
                    BackgroundColor3 = state and themeColor or Color3.fromRGB(50, 56, 72)
                }):Play()

                if flagName then
                    savedData[flagName] = state
                    SaveCurrentConfig()
                end
                callback(state)
            end)

            if defaultState then callback(true) end
        end

        function Tab:CreateSlider(sliderText, flagName, minVal, maxVal, defaultVal, callback)
            callback = callback or function() end
            minVal = minVal or 0
            maxVal = maxVal or 1000

            if flagName and savedData[flagName] ~= nil then
                defaultVal = savedData[flagName]
            end
            defaultVal = math.clamp(defaultVal or minVal, minVal, maxVal)

            local SliderFrame = Instance.new("Frame")
            SliderFrame.Size = UDim2.new(1, -10, 0, 48)
            SliderFrame.BackgroundColor3 = Color3.fromRGB(24, 28, 38)
            SliderFrame.Parent = TabContainer

            local SldCorner = Instance.new("UICorner")
            SldCorner.CornerRadius = UDim.new(0, cornerRadius)
            SldCorner.Parent = SliderFrame

            if enableStroke then
                local SStroke = Instance.new("UIStroke")
                SStroke.Color = strokeColor
                SStroke.Thickness = 1
                SStroke.Parent = SliderFrame
            end

            local Label = Instance.new("TextLabel")
            Label.Size = UDim2.new(1, -60, 0, 22)
            Label.Position = UDim2.new(0, 14, 0, 3)
            Label.BackgroundTransparency = 1
            Label.Text = sliderText
            Label.TextColor3 = Color3.fromRGB(240, 240, 245)
            Label.TextXAlignment = Enum.TextXAlignment.Left
            Label.Font = Enum.Font.Gotham
            Label.TextSize = 13
            Label.Parent = SliderFrame

            local ValueLabel = Instance.new("TextLabel")
            ValueLabel.Size = UDim2.new(0, 50, 0, 22)
            ValueLabel.Position = UDim2.new(1, -60, 0, 3)
            ValueLabel.BackgroundTransparency = 1
            ValueLabel.Text = tostring(defaultVal)
            ValueLabel.TextColor3 = themeColor
            ValueLabel.Font = Enum.Font.GothamBold
            ValueLabel.TextSize = 13
            ValueLabel.Parent = SliderFrame

            local SliderBar = Instance.new("TextButton")
            SliderBar.Size = UDim2.new(1, -28, 0, 7)
            SliderBar.Position = UDim2.new(0, 14, 0, 30)
            SliderBar.BackgroundColor3 = Color3.fromRGB(50, 56, 72)
            SliderBar.Text = ""
            SliderBar.AutoButtonColor = false
            SliderBar.Parent = SliderFrame

            local BarCorner = Instance.new("UICorner")
            BarCorner.CornerRadius = UDim.new(1, 0)
            BarCorner.Parent = SliderBar

            local FillBar = Instance.new("Frame")
            local initPercent = (defaultVal - minVal) / (maxVal - minVal)
            FillBar.Size = UDim2.new(initPercent, 0, 1, 0)
            FillBar.BackgroundColor3 = themeColor
            FillBar.BorderSizePixel = 0
            FillBar.Parent = SliderBar

            local FillCorner = Instance.new("UICorner")
            FillCorner.CornerRadius = UDim.new(1, 0)
            FillCorner.Parent = FillBar

            local sliding = false
            local function updateSlider(input)
                local percent = math.clamp((input.Position.X - SliderBar.AbsolutePosition.X) / SliderBar.AbsoluteSize.X, 0, 1)
                local val = math.floor(minVal + (maxVal - minVal) * percent)
                FillBar.Size = UDim2.new(percent, 0, 1, 0)
                ValueLabel.Text = tostring(val)
                
                if flagName then
                    savedData[flagName] = val
                    SaveCurrentConfig()
                end
                callback(val)
            end

            SliderBar.InputBegan:Connect(function(input)
                if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
                    sliding = true
                    updateSlider(input)
                end
            end)

            UserInputService.InputChanged:Connect(function(input)
                if sliding and (input.UserInputType == Enum.UserInputType.MouseMovement or input.UserInputType == Enum.UserInputType.Touch) then
                    updateSlider(input)
                end
            end)

            UserInputService.InputEnded:Connect(function(input)
                if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
                    sliding = false
                end
            end)
        end

        function Tab:CreateDropdown(dropText, options, defaultOption, callback)
            callback = callback or function() end
            options = options or {}
            local currentChoice = defaultOption or options[1] or "Chưa chọn"

            local DropFrame = Instance.new("Frame")
            DropFrame.Size = UDim2.new(1, -10, 0, 38)
            DropFrame.BackgroundColor3 = Color3.fromRGB(24, 28, 38)
            DropFrame.ClipsDescendants = true
            DropFrame.Parent = TabContainer

            local DropCorner = Instance.new("UICorner")
            DropCorner.CornerRadius = UDim.new(0, cornerRadius)
            DropCorner.Parent = DropFrame

            if enableStroke then
                local DStroke = Instance.new("UIStroke")
                DStroke.Color = strokeColor
                DStroke.Thickness = 1
                DStroke.Parent = DropFrame
            end

            local HeaderBtn = Instance.new("TextButton")
            HeaderBtn.Size = UDim2.new(1, 0, 0, 38)
            HeaderBtn.BackgroundTransparency = 1
            HeaderBtn.Text = ""
            HeaderBtn.Parent = DropFrame

            local Label = Instance.new("TextLabel")
            Label.Size = UDim2.new(0.5, 0, 1, 0)
            Label.Position = UDim2.new(0, 14, 0, 0)
            Label.BackgroundTransparency = 1
            Label.Text = dropText
            Label.TextColor3 = Color3.fromRGB(240, 240, 245)
            Label.TextXAlignment = Enum.TextXAlignment.Left
            Label.Font = Enum.Font.Gotham
            Label.TextSize = 13
            Label.Parent = HeaderBtn

            local SelectedLabel = Instance.new("TextLabel")
            SelectedLabel.Size = UDim2.new(0.5, -20, 1, 0)
            SelectedLabel.Position = UDim2.new(0.5, -5, 0, 0)
            SelectedLabel.BackgroundTransparency = 1
            SelectedLabel.Text = currentChoice .. " ▼"
            SelectedLabel.TextColor3 = themeColor
            SelectedLabel.TextXAlignment = Enum.TextXAlignment.Right
            SelectedLabel.Font = Enum.Font.GothamBold
            SelectedLabel.TextSize = 12
            SelectedLabel.Parent = HeaderBtn

            local OptionContainer = Instance.new("Frame")
            OptionContainer.Size = UDim2.new(1, -16, 0, #options * 28)
            OptionContainer.Position = UDim2.new(0, 8, 0, 40)
            OptionContainer.BackgroundTransparency = 1
            OptionContainer.Parent = DropFrame

            local OptLayout = Instance.new("UIListLayout")
            OptLayout.Padding = UDim.new(0, 3)
            OptLayout.Parent = OptionContainer

            local isExpanded = false
            HeaderBtn.MouseButton1Click:Connect(function()
                isExpanded = not isExpanded
                SelectedLabel.Text = currentChoice .. (isExpanded and " ▲" or " ▼")
                local targetHeight = isExpanded and (40 + #options * 30) or 38
                DropFrame:TweenSize(UDim2.new(1, -10, 0, targetHeight), Enum.EasingDirection.Out, Enum.EasingStyle.Quart, 0.2, true)
            end)

            for _, opt in ipairs(options) do
                local OptBtn = Instance.new("TextButton")
                OptBtn.Size = UDim2.new(1, 0, 0, 28)
                OptBtn.BackgroundColor3 = Color3.fromRGB(18, 20, 26)
                OptBtn.Text = opt
                OptBtn.TextColor3 = Color3.fromRGB(200, 205, 220)
                OptBtn.Font = Enum.Font.Gotham
                OptBtn.TextSize = 12
                OptBtn.Parent = OptionContainer

                local OptCorner = Instance.new("UICorner")
                OptCorner.CornerRadius = UDim.new(0, cornerRadius - 2)
                OptCorner.Parent = OptBtn

                OptBtn.MouseButton1Click:Connect(function()
                    currentChoice = opt
                    SelectedLabel.Text = currentChoice .. " ▼"
                    isExpanded = false
                    DropFrame:TweenSize(UDim2.new(1, -10, 0, 38), Enum.EasingDirection.Out, Enum.EasingStyle.Quart, 0.2, true)
                    callback(opt)
                end)
            end
        end

        function Tab:CreateTextbox(boxText, flagName, maxChars, callback)
            callback = callback or function() end

            local defaultVal = ""
            if flagName and savedData[flagName] ~= nil then
                defaultVal = savedData[flagName]
            end

            local BoxFrame = Instance.new("Frame")
            BoxFrame.Size = UDim2.new(1, -10, 0, 38)
            BoxFrame.BackgroundColor3 = Color3.fromRGB(24, 28, 38)
            BoxFrame.Parent = TabContainer

            local FrameCorner = Instance.new("UICorner")
            FrameCorner.CornerRadius = UDim.new(0, cornerRadius)
            FrameCorner.Parent = BoxFrame

            if enableStroke then
                local TxStroke = Instance.new("UIStroke")
                TxStroke.Color = strokeColor
                TxStroke.Thickness = 1
                TxStroke.Parent = BoxFrame
            end

            local Label = Instance.new("TextLabel")
            Label.Size = UDim2.new(0.5, -10, 1, 0)
            Label.Position = UDim2.new(0, 14, 0, 0)
            Label.BackgroundTransparency = 1
            Label.Text = boxText
            Label.TextColor3 = Color3.fromRGB(240, 240, 245)
            Label.TextXAlignment = Enum.TextXAlignment.Left
            Label.Font = Enum.Font.Gotham
            Label.TextSize = 13
            Label.Parent = BoxFrame

            local InputBox = Instance.new("TextBox")
            InputBox.Size = UDim2.new(0.5, -10, 0, 26)
            InputBox.Position = UDim2.new(0.5, 0, 0.5, -13)
            InputBox.BackgroundColor3 = Color3.fromRGB(15, 17, 23)
            InputBox.Text = defaultVal
            InputBox.PlaceholderText = maxChars and ("Max " .. maxChars .. " chữ") or "Nhập..."
            InputBox.TextColor3 = Color3.fromRGB(255, 255, 255)
            InputBox.PlaceholderColor3 = Color3.fromRGB(120, 125, 140)
            InputBox.Font = Enum.Font.Gotham
            InputBox.TextSize = 12
            InputBox.Parent = BoxFrame

            local BoxCorner = Instance.new("UICorner")
            BoxCorner.CornerRadius = UDim.new(0, cornerRadius - 2)
            BoxCorner.Parent = InputBox

            if maxChars then
                InputBox:GetPropertyChangedSignal("Text"):Connect(function()
                    if #InputBox.Text > maxChars then
                        InputBox.Text = string.sub(InputBox.Text, 1, maxChars)
                    end
                end)
            end

            InputBox.FocusLost:Connect(function(enterPressed)
                if flagName then
                    savedData[flagName] = InputBox.Text
                    SaveCurrentConfig()
                end
                callback(InputBox.Text, enterPressed)
            end)
        end

        return Tab
    end

    return Window
end

return AlatferaLib

