--[[
╔══════════════════════════════════════════╗
║      tex8s RIVALS Premium v5.1          ║
║   Ranked Auto-Detect + Score Tracker    ║
║     1v1 / 2v2 / 3v3 / 4v4 / 5v5       ║
╚══════════════════════════════════════════╝
]]

local Players = game:GetService("Players")
local TweenService = game:GetService("TweenService")
local RunService = game:GetService("RunService")
local UIS = game:GetService("UserInputService")
local CoreGui = game:GetService("CoreGui")
local Workspace = game:GetService("Workspace")
local VirtualInputManager = game:GetService("VirtualInputManager")
local Stats = game:GetService("Stats")
local ReplicatedStorage = game:GetService("ReplicatedStorage")

local LocalPlayer = Players.LocalPlayer
local Mouse = LocalPlayer:GetMouse()
local Camera = Workspace.CurrentCamera

-- ════════════════════════════════════════
-- RIVALS RANKED ALGILAMA SİSTEMİ
-- ════════════════════════════════════════

local RivalsDetection = {
    IsInMatch = false,
    MatchType = "None",
    LastPosition = nil,
    TeleportThreshold = 50,
    TeleportCount = 0,
    MatchStartTime = 0,
    MatchCooldown = false,
    PreviousPlayerCount = 0,
    LastMapCheck = 0,
    DetectedMap = "Unknown",
    WasInLobby = true,
}

-- ════════════════════════════════════════
-- SKOR TAKİP SİSTEMİ
-- ════════════════════════════════════════

local ScoreTracker = {
    MyScore = 0,
    EnemyScore = 0,
    LastRoundPlayers = {},
    RoundStartHP = {},
    RoundEndChecked = false,
    LastAliveState = true,
    MatchRounds = 0,
    LastRoundCheckTime = 0,
    RoundInProgress = false,
    PreviousRoundState = false,
    DeathCooldown = false,
    KillCooldown = false,
    PlayersAtRoundStart = {},
    MyTeamAtStart = nil,
    EnemyTeamAtStart = nil,
}

local function GetMyTeamPlayers()
    local myTeam = LocalPlayer.Team
    local teamPlayers = {}
    for _, p in ipairs(Players:GetPlayers()) do
        if p ~= LocalPlayer and p.Team and myTeam and p.Team == myTeam then
            table.insert(teamPlayers, p)
        end
    end
    return teamPlayers
end

local function GetEnemyPlayers()
    local myTeam = LocalPlayer.Team
    local enemies = {}
    for _, p in ipairs(Players:GetPlayers()) do
        if p ~= LocalPlayer then
            if myTeam then
                if p.Team ~= myTeam then
                    table.insert(enemies, p)
                end
            else
                table.insert(enemies, p)
            end
        end
    end
    return enemies
end

local function CountAliveInTeam(playerList)
    local alive = 0
    for _, p in ipairs(playerList) do
        if p.Character then
            local hum = p.Character:FindFirstChild("Humanoid")
            if hum and hum.Health > 0 then
                alive = alive + 1
            end
        end
    end
    return alive
end

local function IsMyCharAlive()
    local char = LocalPlayer.Character
    if char then
        local hum = char:FindFirstChild("Humanoid")
        if hum and hum.Health > 0 then
            return true
        end
    end
    return false
end

local function TryReadScoreFromGame()
    local score = {myScore = nil, enemyScore = nil, found = false}

    pcall(function()
        -- Leaderstats kontrolü
        local ls = LocalPlayer:FindFirstChild("leaderstats")
        if ls then
            for _, stat in pairs(ls:GetChildren()) do
                local n = stat.Name:lower()
                if (n:find("win") or n:find("score") or n:find("kill") or n:find("round")) then
                    if stat:IsA("IntValue") or stat:IsA("NumberValue") then
                        score.myScore = stat.Value
                        score.found = true
                    end
                end
            end
        end

        -- PlayerGui'deki skor UI'larını tara
        local playerGui = LocalPlayer:FindFirstChild("PlayerGui")
        if playerGui then
            for _, gui in pairs(playerGui:GetChildren()) do
                if gui:IsA("ScreenGui") and gui.Enabled then
                    for _, desc in pairs(gui:GetDescendants()) do
                        if desc:IsA("TextLabel") and desc.Visible then
                            local txt = desc.Text

                            -- "2 - 1", "2-1", "2 : 1" gibi skor formatlarını yakala
                            local s1, s2 = txt:match("^%s*(%d+)%s*[%-–:]%s*(%d+)%s*$")
                            if s1 and s2 then
                                score.myScore = tonumber(s1)
                                score.enemyScore = tonumber(s2)
                                score.found = true
                            end
                        end
                    end
                end
            end
        end

        -- ReplicatedStorage'dan skor oku
        local rs = ReplicatedStorage
        local searchTargets = {"Score", "Scores", "RoundScore", "MatchScore", "TeamScore", "GameScore"}
        for _, name in ipairs(searchTargets) do
            local obj = rs:FindFirstChild(name, true)
            if obj then
                if obj:IsA("Folder") or obj:IsA("Configuration") then
                    local children = obj:GetChildren()
                    if #children >= 2 then
                        local vals = {}
                        for _, c in pairs(children) do
                            if c:IsA("IntValue") or c:IsA("NumberValue") then
                                table.insert(vals, {name = c.Name, value = c.Value})
                            end
                        end
                        if #vals >= 2 then
                            -- Takım adına göre eşleştir
                            local myTeam = LocalPlayer.Team
                            if myTeam then
                                for _, v in ipairs(vals) do
                                    if v.name == myTeam.Name then
                                        score.myScore = v.value
                                    else
                                        score.enemyScore = v.value
                                    end
                                end
                                if score.myScore and score.enemyScore then
                                    score.found = true
                                end
                            end
                        end
                    end
                elseif obj:IsA("IntValue") or obj:IsA("NumberValue") then
                    score.myScore = obj.Value
                    score.found = true
                end
            end
        end
    end)

    return score
end

local function DetectRoundEnd()
    -- Round sonu algılama: Tüm düşmanlar veya tüm takım arkadaşları öldüyse
    local enemies = GetEnemyPlayers()
    local teammates = GetMyTeamPlayers()

    local enemiesAlive = CountAliveInTeam(enemies)
    local myAlive = IsMyCharAlive()
    local teammatesAlive = CountAliveInTeam(teammates)

    -- 1v1 durumu
    if #enemies == 1 and #teammates == 0 then
        if enemiesAlive == 0 and myAlive and not ScoreTracker.KillCooldown then
            -- Ben hayattayım, düşman öldü = Ben kazandım
            ScoreTracker.MyScore = ScoreTracker.MyScore + 1
            ScoreTracker.MatchRounds = ScoreTracker.MatchRounds + 1
            ScoreTracker.KillCooldown = true
            task.delay(5, function() ScoreTracker.KillCooldown = false end)
            return "win"
        elseif not myAlive and enemiesAlive > 0 and not ScoreTracker.DeathCooldown then
            -- Ben öldüm, düşman hayatta = Düşman kazandı
            ScoreTracker.EnemyScore = ScoreTracker.EnemyScore + 1
            ScoreTracker.MatchRounds = ScoreTracker.MatchRounds + 1
            ScoreTracker.DeathCooldown = true
            task.delay(5, function() ScoreTracker.DeathCooldown = false end)
            return "lose"
        end
    else
        -- Takım maçı
        local totalTeamAlive = (myAlive and 1 or 0) + teammatesAlive

        if enemiesAlive == 0 and totalTeamAlive > 0 and not ScoreTracker.KillCooldown then
            ScoreTracker.MyScore = ScoreTracker.MyScore + 1
            ScoreTracker.MatchRounds = ScoreTracker.MatchRounds + 1
            ScoreTracker.KillCooldown = true
            task.delay(5, function() ScoreTracker.KillCooldown = false end)
            return "win"
        elseif totalTeamAlive == 0 and enemiesAlive > 0 and not ScoreTracker.DeathCooldown then
            ScoreTracker.EnemyScore = ScoreTracker.EnemyScore + 1
            ScoreTracker.MatchRounds = ScoreTracker.MatchRounds + 1
            ScoreTracker.DeathCooldown = true
            task.delay(5, function() ScoreTracker.DeathCooldown = false end)
            return "lose"
        end
    end

    return nil
end

-- ════════════════════════════════════════
-- TEMİZLİK
-- ════════════════════════════════════════

local UI_NAME = "Tex8s_WAR"
local connections = {}
local bindButtons = {}
local espCache = {}
local bindingTarget = nil
local CachedTarget = nil
local dynamicGradientObjects = {}

local function SelfDestruct()
    for _, conn in pairs(connections) do
        if typeof(conn) == "RBXScriptConnection" and conn.Connected then
            conn:Disconnect()
        end
    end
    for _, esp in pairs(espCache) do
        pcall(function()
            if esp.BoxLines then
                for i = 1, 4 do
                    esp.BoxLines[i]:Remove()
                    esp.BoxOutlines[i]:Remove()
                end
            end
            if esp.Drawings then
                esp.Drawings.Tracer:Remove()
                esp.Drawings.Name:Remove()
                esp.Drawings.Dist:Remove()
                esp.Drawings.HPText:Remove()
                esp.Drawings.Faction:Remove()
                esp.Drawings.HPBarBg:Remove()
                esp.Drawings.HPBarFill:Remove()
            end
            if esp.Skeleton then
                for _, bone in pairs(esp.Skeleton) do
                    bone:Remove()
                end
            end
        end)
    end
    table.clear(espCache)
    table.clear(connections)
    if CoreGui:FindFirstChild(UI_NAME) then CoreGui[UI_NAME]:Destroy() end
    if CoreGui:FindFirstChild("Tex8s_FOV") then CoreGui.Tex8s_FOV:Destroy() end
    if CoreGui:FindFirstChild("Tex8s_WM") then CoreGui.Tex8s_WM:Destroy() end
    if CoreGui:FindFirstChild("Tex8s_MatchNotif") then CoreGui.Tex8s_MatchNotif:Destroy() end
    if LocalPlayer.Character then
        for _, obj in pairs(LocalPlayer.Character:GetDescendants()) do
            if obj.Name == "Tex8s_Fly" or obj.Name == "Tex8s_FlyGyro" then
                obj:Destroy()
            end
        end
    end
end

if CoreGui:FindFirstChild(UI_NAME) then SelfDestruct() end

-- ════════════════════════════════════════
-- AYARLAR
-- ════════════════════════════════════════

local Config = {
    UIColor1 = Color3.fromRGB(199, 149, 237),
    UIColor2 = Color3.fromRGB(85, 0, 255),

    Rainbow_Enabled = false,
    Rainbow_Speed = 0.5,
    Watermark_Enabled = true,
    Keybinds_Enabled = true,

    Aim_Enabled = false,
    Aim_Silent = false,
    Aim_Bind = Enum.UserInputType.MouseButton2,
    Aim_Target = "Head",
    Aim_Smooth = 20,
    Aim_Predict = 0,
    Aim_TeamCheck = false,
    Aim_FOV_Show = true,
    Aim_FOV_Radius = 150,

    TP_Enabled = false,
    TP_Bind = Enum.KeyCode.E,
    Noclip_Enabled = false,
    Noclip_Bind = Enum.KeyCode.N,
    Fly_Enabled = false,
    Fly_Bind = Enum.KeyCode.F,
    Fly_Speed = 50,
    Fly_AntiFall = true,

    ESP_Enabled = false,
    ESP_TeamCheck = false,
    ESP_ShowSelf = false,
    ESP_Box = true,
    ESP_Skeleton = false,
    ESP_Tracers = false,
    ESP_Name = true,
    ESP_Name_Pos = "Top",
    ESP_HPText = true,
    ESP_HPText_Pos = "Right",
    ESP_Dist = true,
    ESP_Dist_Pos = "Bottom",
    ESP_Faction = true,
    ESP_Faction_Pos = "Top",
    ESP_HPBar = true,
    ESP_HPBar_Pos = "Left",

    Anti_Kick = true,
    Anti_Idle = true,

    Auto_Detect_Match = true,
    Auto_Enable_ESP = true,
    Auto_Enable_Aim = false,
    Match_Detect_Sensitivity = 50,

    Menu_Bind = Enum.KeyCode.RightShift,
    Unbind_Key = Enum.KeyCode.End,
}

-- ════════════════════════════════════════
-- YARDIMCI FONKSİYONLAR
-- ════════════════════════════════════════

local TweenInfoFast = TweenInfo.new(0.2, Enum.EasingStyle.Sine, Enum.EasingDirection.Out)

local function GetCIELUVColor(percentage)
    local green = Color3.fromRGB(46, 204, 113)
    local red = Color3.fromRGB(231, 76, 60)
    return red:Lerp(green, math.clamp(percentage, 0, 1))
end

local function IsBindActive(bind)
    if not bind then return false end
    if bind.EnumType == Enum.KeyCode then
        return UIS:IsKeyDown(bind)
    elseif bind.EnumType == Enum.UserInputType then
        return UIS:IsMouseButtonPressed(bind)
    end
    return false
end

-- ════════════════════════════════════════════════════════
-- RIVALS MAÇ ALGILAMA FONKSİYONLARI
-- ════════════════════════════════════════════════════════

local function CountNearbyPlayers(radius)
    local count = 0
    local char = LocalPlayer.Character
    if not char or not char:FindFirstChild("HumanoidRootPart") then return 0 end
    local myPos = char.HumanoidRootPart.Position

    for _, player in ipairs(Players:GetPlayers()) do
        if player ~= LocalPlayer and player.Character and player.Character:FindFirstChild("HumanoidRootPart") then
            local dist = (player.Character.HumanoidRootPart.Position - myPos).Magnitude
            if dist < (radius or 150) then
                count = count + 1
            end
        end
    end
    return count
end

local function GuessMatchType(nearbyPlayers)
    if nearbyPlayers == 1 then return "1v1"
    elseif nearbyPlayers >= 2 and nearbyPlayers <= 3 then return "2v2"
    elseif nearbyPlayers >= 4 and nearbyPlayers <= 5 then return "3v3"
    elseif nearbyPlayers >= 6 and nearbyPlayers <= 7 then return "4v4"
    elseif nearbyPlayers >= 8 and nearbyPlayers <= 9 then return "5v5"
    elseif nearbyPlayers >= 10 then return "5v5+"
    end
    return "Unknown"
end

local function ScanForRivalsMatchData()
    local info = {inMatch = false, matchType = "None", ranked = false}

    pcall(function()
        local rs = ReplicatedStorage
        local searchNames = {
            "GameState", "MatchData", "RoundInfo", "GameData",
            "MatchInfo", "RoundData", "GameInfo", "State",
            "CurrentMatch", "ActiveMatch", "Match", "Round",
            "GameStatus", "MatchStatus", "SharedData", "Remotes"
        }

        local function deepSearch(parent, depth)
            if depth > 3 then return end
            for _, child in pairs(parent:GetChildren()) do
                local name = child.Name:lower()
                if child:IsA("StringValue") then
                    local val = child.Value:lower()
                    if val:find("ranked") then info.ranked = true; info.inMatch = true end
                    if val:find("1v1") then info.matchType = "1v1"; info.inMatch = true
                    elseif val:find("2v2") then info.matchType = "2v2"; info.inMatch = true
                    elseif val:find("3v3") then info.matchType = "3v3"; info.inMatch = true
                    elseif val:find("4v4") then info.matchType = "4v4"; info.inMatch = true
                    elseif val:find("5v5") then info.matchType = "5v5"; info.inMatch = true end
                    if val:find("match") or val:find("fight") or val:find("round") or val:find("battle") then
                        info.inMatch = true
                    end
                elseif child:IsA("BoolValue") then
                    if (name:find("match") or name:find("round") or name:find("ranked") or name:find("active") or name:find("started") or name:find("ingame")) and child.Value then
                        info.inMatch = true
                    end
                elseif child:IsA("IntValue") or child:IsA("NumberValue") then
                    if (name:find("round") or name:find("score") or name:find("timer")) and child.Value > 0 then
                        info.inMatch = true
                    end
                end
                if child:IsA("Folder") or child:IsA("Configuration") or child:IsA("Model") then
                    deepSearch(child, depth + 1)
                end
            end
        end

        for _, searchName in ipairs(searchNames) do
            local found = rs:FindFirstChild(searchName)
            if found then
                deepSearch(found, 0)
            end
        end
        deepSearch(rs, 0)

        local mapSearchNames = {"Arena", "Map", "BattleArena", "FightArea", "MatchArea", "GameMap", "ActiveMap", "Stage", "Combat"}
        for _, mapName in ipairs(mapSearchNames) do
            local map = Workspace:FindFirstChild(mapName, true)
            if map then
                info.inMatch = true
            end
        end

        local playerGui = LocalPlayer:FindFirstChild("PlayerGui")
        if playerGui then
            for _, gui in pairs(playerGui:GetChildren()) do
                if gui:IsA("ScreenGui") and gui.Enabled then
                    local guiName = gui.Name:lower()
                    if guiName:find("match") or guiName:find("ranked") or guiName:find("battle") or guiName:find("fight") or guiName:find("combat") or guiName:find("round") then
                        info.inMatch = true
                    end
                    pcall(function()
                        for _, desc in pairs(gui:GetDescendants()) do
                            if desc:IsA("TextLabel") and desc.Visible then
                                local txt = desc.Text:lower()
                                if txt:find("round") or txt:find("vs") or txt:find("score") or txt:find("match") then
                                    info.inMatch = true
                                end
                            end
                        end
                    end)
                end
            end
        end
    end)

    if info.ranked and info.matchType ~= "None" then
        info.matchType = "Ranked " .. info.matchType
    elseif info.ranked then
        info.matchType = "Ranked"
    end

    return info
end

-- ════════════════════════════════════════════════════════
-- MAÇ BİLDİRİM SİSTEMİ (SKOR GÖSTERİMLİ)
-- ════════════════════════════════════════════════════════

local NotifGui = Instance.new("ScreenGui")
NotifGui.Name = "Tex8s_MatchNotif"
NotifGui.IgnoreGuiInset = true
NotifGui.DisplayOrder = 10000
NotifGui.Parent = CoreGui

local function ShowMatchNotification(matchType)
    for _, child in pairs(NotifGui:GetChildren()) do child:Destroy() end

    local notifFrame = Instance.new("Frame")
    notifFrame.Size = UDim2.new(0, 380, 0, 130)
    notifFrame.Position = UDim2.new(0.5, 0, 0, -140)
    notifFrame.AnchorPoint = Vector2.new(0.5, 0)
    notifFrame.BackgroundColor3 = Color3.fromRGB(12, 12, 18)
    notifFrame.BorderSizePixel = 0
    notifFrame.ClipsDescendants = true
    notifFrame.Parent = NotifGui

    local corner = Instance.new("UICorner")
    corner.CornerRadius = UDim.new(0, 12)
    corner.Parent = notifFrame

    local nStroke = Instance.new("UIStroke")
    nStroke.Thickness = 2
    nStroke.Parent = notifFrame

    local nGrad = Instance.new("UIGradient")
    nGrad.Color = ColorSequence.new({
        ColorSequenceKeypoint.new(0, Color3.fromRGB(199, 149, 237)),
        ColorSequenceKeypoint.new(1, Color3.fromRGB(85, 0, 255))
    })
    nGrad.Rotation = 90
    nGrad.Parent = nStroke

    local topAccent = Instance.new("Frame")
    topAccent.Size = UDim2.new(1, 0, 0, 3)
    topAccent.Position = UDim2.new(0, 0, 0, 0)
    topAccent.BackgroundColor3 = Color3.fromRGB(140, 80, 255)
    topAccent.BorderSizePixel = 0
    topAccent.Parent = notifFrame

    local topAccGrad = Instance.new("UIGradient")
    topAccGrad.Color = ColorSequence.new({
        ColorSequenceKeypoint.new(0, Color3.fromRGB(88, 101, 242)),
        ColorSequenceKeypoint.new(0.5, Color3.fromRGB(199, 149, 237)),
        ColorSequenceKeypoint.new(1, Color3.fromRGB(88, 101, 242))
    })
    topAccGrad.Parent = topAccent

    local matchIcon = Instance.new("TextLabel")
    matchIcon.Size = UDim2.new(0, 40, 0, 40)
    matchIcon.Position = UDim2.new(0, 15, 0, 18)
    matchIcon.BackgroundTransparency = 1
    matchIcon.Text = "⚔"
    matchIcon.TextSize = 28
    matchIcon.Font = Enum.Font.GothamBold
    matchIcon.TextColor3 = Color3.fromRGB(199, 149, 237)
    matchIcon.Parent = notifFrame

    local matchTitle = Instance.new("TextLabel")
    matchTitle.Size = UDim2.new(1, -70, 0, 22)
    matchTitle.Position = UDim2.new(0, 60, 0, 12)
    matchTitle.BackgroundTransparency = 1
    matchTitle.Text = "⚡ MATCH DETECTED"
    matchTitle.TextColor3 = Color3.fromRGB(255, 200, 100)
    matchTitle.TextSize = 16
    matchTitle.Font = Enum.Font.GothamBlack
    matchTitle.TextXAlignment = Enum.TextXAlignment.Left
    matchTitle.Parent = notifFrame

    local matchTypeLabel = Instance.new("TextLabel")
    matchTypeLabel.Size = UDim2.new(1, -70, 0, 20)
    matchTypeLabel.Position = UDim2.new(0, 60, 0, 36)
    matchTypeLabel.BackgroundTransparency = 1
    matchTypeLabel.Text = "Mode: " .. matchType
    matchTypeLabel.TextColor3 = Color3.fromRGB(180, 130, 255)
    matchTypeLabel.TextSize = 14
    matchTypeLabel.Font = Enum.Font.GothamSemibold
    matchTypeLabel.TextXAlignment = Enum.TextXAlignment.Left
    matchTypeLabel.Parent = notifFrame

    -- SKOR GÖSTERGESİ
    local scoreFrame = Instance.new("Frame")
    scoreFrame.Size = UDim2.new(1, -30, 0, 32)
    scoreFrame.Position = UDim2.new(0, 15, 0, 62)
    scoreFrame.BackgroundColor3 = Color3.fromRGB(20, 20, 30)
    scoreFrame.BorderSizePixel = 0
    scoreFrame.Parent = notifFrame

    local scoreCorner = Instance.new("UICorner")
    scoreCorner.CornerRadius = UDim.new(0, 8)
    scoreCorner.Parent = scoreFrame

    local scoreStroke = Instance.new("UIStroke")
    scoreStroke.Thickness = 1
    scoreStroke.Color = Color3.fromRGB(60, 40, 120)
    scoreStroke.Parent = scoreFrame

    -- Oyun verilerinden skor okumayı dene
    local gameScore = TryReadScoreFromGame()
    local myS = gameScore.found and gameScore.myScore or ScoreTracker.MyScore
    local enS = gameScore.found and (gameScore.enemyScore or ScoreTracker.EnemyScore) or ScoreTracker.EnemyScore

    -- Eğer oyundan skor okunduysa tracker'ı güncelle
    if gameScore.found then
        if gameScore.myScore then ScoreTracker.MyScore = gameScore.myScore end
        if gameScore.enemyScore then ScoreTracker.EnemyScore = gameScore.enemyScore end
    end

    local meLabel = Instance.new("TextLabel")
    meLabel.Size = UDim2.new(0.35, 0, 1, 0)
    meLabel.Position = UDim2.new(0, 8, 0, 0)
    meLabel.BackgroundTransparency = 1
    meLabel.Text = "ME: " .. tostring(myS)
    meLabel.TextColor3 = Color3.fromRGB(80, 220, 100)
    meLabel.TextSize = 16
    meLabel.Font = Enum.Font.GothamBlack
    meLabel.TextXAlignment = Enum.TextXAlignment.Left
    meLabel.Parent = scoreFrame

    local vsLabel = Instance.new("TextLabel")
    vsLabel.Size = UDim2.new(0.3, 0, 1, 0)
    vsLabel.Position = UDim2.new(0.35, 0, 0, 0)
    vsLabel.BackgroundTransparency = 1
    vsLabel.Text = "—"
    vsLabel.TextColor3 = Color3.fromRGB(150, 150, 170)
    vsLabel.TextSize = 18
    vsLabel.Font = Enum.Font.GothamBlack
    vsLabel.Parent = scoreFrame

    local enemyLabel = Instance.new("TextLabel")
    enemyLabel.Size = UDim2.new(0.35, 0, 1, 0)
    enemyLabel.Position = UDim2.new(0.65, 0, 0, 0)
    enemyLabel.BackgroundTransparency = 1
    enemyLabel.Text = "ENEMY: " .. tostring(enS)
    enemyLabel.TextColor3 = Color3.fromRGB(231, 76, 60)
    enemyLabel.TextSize = 16
    enemyLabel.Font = Enum.Font.GothamBlack
    enemyLabel.TextXAlignment = Enum.TextXAlignment.Right
    enemyLabel.Parent = scoreFrame

    local infoLabel = Instance.new("TextLabel")
    infoLabel.Size = UDim2.new(1, -20, 0, 16)
    infoLabel.Position = UDim2.new(0, 10, 1, -20)
    infoLabel.BackgroundTransparency = 1
    infoLabel.Text = "tex8s Premium • Rivals Ranked • Round " .. tostring(ScoreTracker.MatchRounds + 1)
    infoLabel.TextColor3 = Color3.fromRGB(80, 80, 100)
    infoLabel.TextSize = 10
    infoLabel.Font = Enum.Font.Gotham
    infoLabel.TextXAlignment = Enum.TextXAlignment.Center
    infoLabel.Parent = notifFrame

    TweenService:Create(notifFrame, TweenInfo.new(0.6, Enum.EasingStyle.Back, Enum.EasingDirection.Out), {
        Position = UDim2.new(0.5, 0, 0, 20)
    }):Play()

    task.spawn(function()
        local startT = tick()
        while tick() - startT < 6 and notifFrame and notifFrame.Parent do
            local hue = ((tick() * 0.5) % 1)
            nGrad.Color = ColorSequence.new({
                ColorSequenceKeypoint.new(0, Color3.fromHSV(hue, 0.6, 1)),
                ColorSequenceKeypoint.new(1, Color3.fromHSV((hue + 0.3) % 1, 0.6, 0.8))
            })

            -- Canlı skor güncelle
            pcall(function()
                local liveScore = TryReadScoreFromGame()
                local lMyS = liveScore.found and liveScore.myScore or ScoreTracker.MyScore
                local lEnS = liveScore.found and (liveScore.enemyScore or ScoreTracker.EnemyScore) or ScoreTracker.EnemyScore
                meLabel.Text = "ME: " .. tostring(lMyS)
                enemyLabel.Text = "ENEMY: " .. tostring(lEnS)
            end)

            RunService.Heartbeat:Wait()
        end
    end)

    task.delay(5, function()
        if notifFrame and notifFrame.Parent then
            TweenService:Create(notifFrame, TweenInfo.new(0.5, Enum.EasingStyle.Quint, Enum.EasingDirection.In), {
                Position = UDim2.new(0.5, 0, 0, -160)
            }):Play()
            task.wait(0.6)
            if notifFrame and notifFrame.Parent then notifFrame:Destroy() end
        end
    end)
end

-- Round kazanma/kaybetme bildirimi
local function ShowRoundResultNotification(result)
    for _, child in pairs(NotifGui:GetChildren()) do child:Destroy() end

    local notifFrame = Instance.new("Frame")
    notifFrame.Size = UDim2.new(0, 340, 0, 100)
    notifFrame.Position = UDim2.new(0.5, 0, 0, -120)
    notifFrame.AnchorPoint = Vector2.new(0.5, 0)
    notifFrame.BackgroundColor3 = Color3.fromRGB(12, 12, 18)
    notifFrame.BorderSizePixel = 0
    notifFrame.ClipsDescendants = true
    notifFrame.Parent = NotifGui

    local corner = Instance.new("UICorner")
    corner.CornerRadius = UDim.new(0, 12)
    corner.Parent = notifFrame

    local nStroke = Instance.new("UIStroke")
    nStroke.Thickness = 2
    nStroke.Parent = notifFrame

    local resultColor = result == "win" and Color3.fromRGB(80, 220, 100) or Color3.fromRGB(231, 76, 60)
    local resultText = result == "win" and "🏆 ROUND WON!" or "💀 ROUND LOST"
    local resultIcon = result == "win" and "✓" or "✗"

    nStroke.Color = resultColor

    local topAccent = Instance.new("Frame")
    topAccent.Size = UDim2.new(1, 0, 0, 3)
    topAccent.Position = UDim2.new(0, 0, 0, 0)
    topAccent.BackgroundColor3 = resultColor
    topAccent.BorderSizePixel = 0
    topAccent.Parent = notifFrame

    local iconLabel = Instance.new("TextLabel")
    iconLabel.Size = UDim2.new(0, 40, 0, 40)
    iconLabel.Position = UDim2.new(0, 12, 0, 12)
    iconLabel.BackgroundTransparency = 1
    iconLabel.Text = resultIcon
    iconLabel.TextSize = 28
    iconLabel.Font = Enum.Font.GothamBlack
    iconLabel.TextColor3 = resultColor
    iconLabel.Parent = notifFrame

    local titleLabel = Instance.new("TextLabel")
    titleLabel.Size = UDim2.new(1, -65, 0, 22)
    titleLabel.Position = UDim2.new(0, 55, 0, 10)
    titleLabel.BackgroundTransparency = 1
    titleLabel.Text = resultText
    titleLabel.TextColor3 = resultColor
    titleLabel.TextSize = 16
    titleLabel.Font = Enum.Font.GothamBlack
    titleLabel.TextXAlignment = Enum.TextXAlignment.Left
    titleLabel.Parent = notifFrame

    -- Skor göster
    local scoreLabel = Instance.new("TextLabel")
    scoreLabel.Size = UDim2.new(1, -65, 0, 28)
    scoreLabel.Position = UDim2.new(0, 55, 0, 35)
    scoreLabel.BackgroundTransparency = 1
    scoreLabel.Text = "ME: " .. ScoreTracker.MyScore .. "  —  ENEMY: " .. ScoreTracker.EnemyScore
    scoreLabel.TextColor3 = Color3.new(1, 1, 1)
    scoreLabel.TextSize = 18
    scoreLabel.Font = Enum.Font.GothamBlack
    scoreLabel.TextXAlignment = Enum.TextXAlignment.Left
    scoreLabel.Parent = notifFrame

    local roundLabel = Instance.new("TextLabel")
    roundLabel.Size = UDim2.new(1, -20, 0, 16)
    roundLabel.Position = UDim2.new(0, 10, 1, -22)
    roundLabel.BackgroundTransparency = 1
    roundLabel.Text = "Round " .. ScoreTracker.MatchRounds .. " completed • tex8s Premium"
    roundLabel.TextColor3 = Color3.fromRGB(80, 80, 100)
    roundLabel.TextSize = 10
    roundLabel.Font = Enum.Font.Gotham
    roundLabel.TextXAlignment = Enum.TextXAlignment.Center
    roundLabel.Parent = notifFrame

    TweenService:Create(notifFrame, TweenInfo.new(0.5, Enum.EasingStyle.Back, Enum.EasingDirection.Out), {
        Position = UDim2.new(0.5, 0, 0, 20)
    }):Play()

    task.delay(4, function()
        if notifFrame and notifFrame.Parent then
            TweenService:Create(notifFrame, TweenInfo.new(0.4, Enum.EasingStyle.Quint, Enum.EasingDirection.In), {
                Position = UDim2.new(0.5, 0, 0, -140)
            }):Play()
            task.wait(0.5)
            if notifFrame and notifFrame.Parent then notifFrame:Destroy() end
        end
    end)
end

-- ════════════════════════════════════════
-- FOV CIRCLE
-- ════════════════════════════════════════

local FOV_Gui = Instance.new("ScreenGui")
FOV_Gui.Name = "Tex8s_FOV"
FOV_Gui.IgnoreGuiInset = true
FOV_Gui.Parent = CoreGui

local FOV_Circle = Instance.new("Frame")
FOV_Circle.BackgroundTransparency = 1
FOV_Circle.Size = UDim2.new(0, Config.Aim_FOV_Radius * 2, 0, Config.Aim_FOV_Radius * 2)
FOV_Circle.AnchorPoint = Vector2.new(0.5, 0.5)
FOV_Circle.Parent = FOV_Gui

local FOV_Stroke = Instance.new("UIStroke")
FOV_Stroke.Thickness = 1
FOV_Stroke.Parent = FOV_Circle

local FOV_Grad = Instance.new("UIGradient")
FOV_Grad.Color = ColorSequence.new(Config.UIColor1)
FOV_Grad.Parent = FOV_Stroke

local FOV_Corner = Instance.new("UICorner")
FOV_Corner.CornerRadius = UDim.new(1, 0)
FOV_Corner.Parent = FOV_Circle

-- ════════════════════════════════════════
-- ANA MENÜ UI
-- ════════════════════════════════════════

local ScreenGui = Instance.new("ScreenGui")
ScreenGui.Name = UI_NAME
ScreenGui.ResetOnSpawn = false
ScreenGui.Parent = CoreGui

local MainFrame = Instance.new("Frame")
MainFrame.Size = UDim2.new(0, 560, 0, 480)
MainFrame.Position = UDim2.new(0.5, -280, 0.5, -240)
MainFrame.BackgroundColor3 = Color3.fromRGB(15, 15, 15)
MainFrame.BorderSizePixel = 0
MainFrame.Active = true
MainFrame.ClipsDescendants = true
MainFrame.Visible = false
MainFrame.Parent = ScreenGui

local mainCorner = Instance.new("UICorner")
mainCorner.CornerRadius = UDim.new(0, 8)
mainCorner.Parent = MainFrame

local MainStroke = Instance.new("UIStroke")
MainStroke.Thickness = 1.5
MainStroke.Transparency = 0.3
MainStroke.Parent = MainFrame

local MainGradient = Instance.new("UIGradient")
MainGradient.Parent = MainStroke
table.insert(dynamicGradientObjects, MainGradient)

-- Top Bar
local MenuTopBar = Instance.new("Frame")
MenuTopBar.Size = UDim2.new(1, 0, 0, 40)
MenuTopBar.BackgroundColor3 = Color3.fromRGB(20, 20, 20)
MenuTopBar.BorderSizePixel = 0
MenuTopBar.Parent = MainFrame

local topBarCorner = Instance.new("UICorner")
topBarCorner.CornerRadius = UDim.new(0, 8)
topBarCorner.Parent = MenuTopBar

local TopBarFix = Instance.new("Frame")
TopBarFix.Size = UDim2.new(1, 0, 0, 8)
TopBarFix.Position = UDim2.new(0, 0, 1, -8)
TopBarFix.BackgroundColor3 = Color3.fromRGB(20, 20, 20)
TopBarFix.BorderSizePixel = 0
TopBarFix.Parent = MenuTopBar

local Title = Instance.new("TextLabel")
Title.Size = UDim2.new(1, -150, 1, 0)
Title.Position = UDim2.new(0, 12, 0, 0)
Title.Text = "TEX8S RİVALS Premium | v5.1"
Title.TextColor3 = Color3.new(1, 1, 1)
Title.Font = Enum.Font.GothamBold
Title.TextSize = 14
Title.BackgroundTransparency = 1
Title.TextXAlignment = Enum.TextXAlignment.Left
Title.Parent = MenuTopBar

local TitleGradient = Instance.new("UIGradient")
TitleGradient.Parent = Title
table.insert(dynamicGradientObjects, TitleGradient)

-- Maç Durumu Göstergesi
local MatchIndicator = Instance.new("TextLabel")
MatchIndicator.Size = UDim2.new(0, 120, 0, 22)
MatchIndicator.Position = UDim2.new(1, -130, 0.5, -11)
MatchIndicator.BackgroundColor3 = Color3.fromRGB(30, 30, 30)
MatchIndicator.Text = "● LOBBY"
MatchIndicator.TextColor3 = Color3.fromRGB(100, 100, 100)
MatchIndicator.TextSize = 10
MatchIndicator.Font = Enum.Font.GothamBold
MatchIndicator.BorderSizePixel = 0
MatchIndicator.Parent = MenuTopBar

local matchIndCorner = Instance.new("UICorner")
matchIndCorner.CornerRadius = UDim.new(0, 4)
matchIndCorner.Parent = MatchIndicator

-- Sürükleme
local dragging, dragInput, dragStart, startPos

MenuTopBar.InputBegan:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseButton1 then
        dragging = true
        dragStart = input.Position
        startPos = MainFrame.Position
        input.Changed:Connect(function()
            if input.UserInputState == Enum.UserInputState.End then
                dragging = false
            end
        end)
    end
end)

table.insert(connections, UIS.InputChanged:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseMovement then
        dragInput = input
    end
end))

-- Resize
local ResizeGrip = Instance.new("TextButton")
ResizeGrip.Size = UDim2.new(0, 15, 0, 15)
ResizeGrip.Position = UDim2.new(1, -15, 1, -15)
ResizeGrip.BackgroundTransparency = 1
ResizeGrip.Text = "◢"
ResizeGrip.TextColor3 = Color3.fromRGB(100, 100, 100)
ResizeGrip.TextSize = 14
ResizeGrip.Parent = MainFrame

local resizing, rDragStart, rStartSize

ResizeGrip.InputBegan:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseButton1 then
        resizing = true
        rDragStart = input.Position
        rStartSize = MainFrame.Size
        input.Changed:Connect(function()
            if input.UserInputState == Enum.UserInputState.End then
                resizing = false
            end
        end)
    end
end)

table.insert(connections, RunService.Heartbeat:Connect(function()
    if dragging and dragInput then
        local delta = dragInput.Position - dragStart
        MainFrame.Position = UDim2.new(
            startPos.X.Scale, startPos.X.Offset + delta.X,
            startPos.Y.Scale, startPos.Y.Offset + delta.Y
        )
    elseif resizing and dragInput then
        local delta = dragInput.Position - rDragStart
        MainFrame.Size = UDim2.new(
            0, math.clamp(rStartSize.X.Offset + delta.X, 400, 800),
            0, math.clamp(rStartSize.Y.Offset + delta.Y, 300, 650)
        )
    end
end))

-- ════════════════════════════════════════
-- TAB SİSTEMİ
-- ════════════════════════════════════════

local TabContainer = Instance.new("Frame")
TabContainer.Size = UDim2.new(1, -20, 0, 30)
TabContainer.Position = UDim2.new(0, 10, 0, 50)
TabContainer.BackgroundTransparency = 1
TabContainer.Parent = MainFrame

local TabLayout = Instance.new("UIListLayout")
TabLayout.FillDirection = Enum.FillDirection.Horizontal
TabLayout.SortOrder = Enum.SortOrder.LayoutOrder
TabLayout.Padding = UDim.new(0, 6)
TabLayout.Parent = TabContainer

local PagesContainer = Instance.new("Frame")
PagesContainer.Size = UDim2.new(1, -20, 1, -95)
PagesContainer.Position = UDim2.new(0, 10, 0, 90)
PagesContainer.BackgroundTransparency = 1
PagesContainer.Parent = MainFrame

local tabs, pages = {}, {}

local function CreateTab(name, isFirst)
    local tabBtn = Instance.new("TextButton")
    tabBtn.Size = UDim2.new(0, 85, 1, 0)
    tabBtn.BackgroundColor3 = isFirst and Color3.new(1, 1, 1) or Color3.fromRGB(25, 25, 25)
    tabBtn.Text = name
    tabBtn.TextColor3 = isFirst and Color3.new(0, 0, 0) or Color3.fromRGB(200, 200, 200)
    tabBtn.Font = Enum.Font.GothamBold
    tabBtn.TextSize = 11
    tabBtn.AutoButtonColor = false
    tabBtn.BorderSizePixel = 0
    tabBtn.Parent = TabContainer

    local tabCorner = Instance.new("UICorner")
    tabCorner.CornerRadius = UDim.new(0, 6)
    tabCorner.Parent = tabBtn

    local grad = Instance.new("UIGradient")
    grad.Parent = tabBtn
    if isFirst then
        table.insert(dynamicGradientObjects, grad)
    end

    local page = Instance.new("ScrollingFrame")
    page.Size = UDim2.new(1, 0, 1, 0)
    page.BackgroundTransparency = 1
    page.CanvasSize = UDim2.new(0, 0, 0, 0)
    page.AutomaticCanvasSize = Enum.AutomaticSize.Y
    page.ScrollBarThickness = 4
    page.Visible = isFirst
    page.BorderSizePixel = 0
    page.Parent = PagesContainer

    local pageLayout = Instance.new("UIListLayout")
    pageLayout.Padding = UDim.new(0, 8)
    pageLayout.SortOrder = Enum.SortOrder.LayoutOrder
    pageLayout.Parent = page

    table.insert(tabs, tabBtn)
    table.insert(pages, page)

    tabBtn.MouseButton1Click:Connect(function()
        for i, p in ipairs(pages) do
            local isSel = (p == page)
            p.Visible = isSel
            local tBtn = tabs[i]
            local tGrad = tBtn:FindFirstChildOfClass("UIGradient")

            if isSel then
                TweenService:Create(tBtn, TweenInfoFast, {
                    BackgroundColor3 = Color3.new(1, 1, 1),
                    TextColor3 = Color3.new(0, 0, 0)
                }):Play()
                if tGrad then
                    local found = false
                    for _, v in pairs(dynamicGradientObjects) do
                        if v == tGrad then found = true; break end
                    end
                    if not found then table.insert(dynamicGradientObjects, tGrad) end
                end
            else
                TweenService:Create(tBtn, TweenInfoFast, {
                    BackgroundColor3 = Color3.fromRGB(25, 25, 25),
                    TextColor3 = Color3.fromRGB(200, 200, 200)
                }):Play()
                if tGrad then
                    tGrad.Color = ColorSequence.new(Color3.new(1, 1, 1))
                    for k, v in pairs(dynamicGradientObjects) do
                        if v == tGrad then
                            table.remove(dynamicGradientObjects, k)
                            break
                        end
                    end
                end
            end
        end
    end)

    return page
end

local TabAim = CreateTab("Aim", true)
local TabESP = CreateTab("ESP", false)
local TabMovement = CreateTab("Move", false)
local TabRivals = CreateTab("Rivals", false)
local TabSettings = CreateTab("Settings", false)

-- ════════════════════════════════════════
-- AYAR OLUŞTURUCULAR
-- ════════════════════════════════════════

local function AddSetting(parentTab, text, configKey, bindKey, modeKey, cycleOptions)
    local frame = Instance.new("Frame")
    frame.Size = UDim2.new(1, -8, 0, 36)
    frame.BackgroundColor3 = Color3.fromRGB(22, 22, 22)
    frame.BorderSizePixel = 0
    frame.Parent = parentTab

    local fCorner = Instance.new("UICorner")
    fCorner.CornerRadius = UDim.new(0, 6)
    fCorner.Parent = frame

    local label = Instance.new("TextLabel")
    label.Size = UDim2.new(0.4, 0, 1, 0)
    label.Position = UDim2.new(0, 12, 0, 0)
    label.Text = text
    label.TextColor3 = Color3.fromRGB(220, 220, 220)
    label.Font = Enum.Font.GothamMedium
    label.TextSize = 13
    label.TextXAlignment = Enum.TextXAlignment.Left
    label.BackgroundTransparency = 1
    label.Parent = frame

    local xOffset = 0.43

    if configKey and type(Config[configKey]) == "boolean" then
        local toggleBg = Instance.new("TextButton")
        toggleBg.Size = UDim2.new(0, 40, 0, 20)
        toggleBg.Position = UDim2.new(xOffset, 0, 0.5, -10)
        toggleBg.BackgroundColor3 = Config[configKey] and Color3.new(1, 1, 1) or Color3.fromRGB(40, 40, 40)
        toggleBg.Text = ""
        toggleBg.AutoButtonColor = false
        toggleBg.BorderSizePixel = 0
        toggleBg.Parent = frame

        local tbCorner = Instance.new("UICorner")
        tbCorner.CornerRadius = UDim.new(1, 0)
        tbCorner.Parent = toggleBg

        local toggleKnob = Instance.new("Frame")
        toggleKnob.Size = UDim2.new(0, 16, 0, 16)
        toggleKnob.Position = UDim2.new(0, Config[configKey] and 22 or 2, 0.5, -8)
        toggleKnob.BackgroundColor3 = Color3.new(1, 1, 1)
        toggleKnob.Parent = toggleBg

        local tkCorner = Instance.new("UICorner")
        tkCorner.CornerRadius = UDim.new(1, 0)
        tkCorner.Parent = toggleKnob

        toggleBg.MouseButton1Click:Connect(function()
            Config[configKey] = not Config[configKey]
            local state = Config[configKey]
            TweenService:Create(toggleKnob, TweenInfoFast, {
                Position = UDim2.new(0, state and 22 or 2, 0.5, -8)
            }):Play()
            TweenService:Create(toggleBg, TweenInfoFast, {
                BackgroundColor3 = state and Color3.new(1, 1, 1) or Color3.fromRGB(40, 40, 40)
            }):Play()
        end)
        xOffset = xOffset + 0.1
    end

    if modeKey then
        local modeBtn = Instance.new("TextButton")
        modeBtn.Size = UDim2.new(0, 90, 0, 24)
        modeBtn.Position = UDim2.new(xOffset, 0, 0.5, -12)
        modeBtn.BackgroundColor3 = Color3.fromRGB(35, 35, 35)
        modeBtn.Text = cycleOptions and ("Mode: " .. Config[modeKey]) or Config[modeKey]
        modeBtn.TextColor3 = Color3.new(1, 1, 1)
        modeBtn.Font = Enum.Font.GothamSemibold
        modeBtn.TextSize = 11
        modeBtn.BorderSizePixel = 0
        modeBtn.Parent = frame

        local mbCorner = Instance.new("UICorner")
        mbCorner.CornerRadius = UDim.new(0, 4)
        mbCorner.Parent = modeBtn

        local mbStroke = Instance.new("UIStroke")
        mbStroke.Color = Color3.fromRGB(50, 50, 50)
        mbStroke.Parent = modeBtn

        modeBtn.MouseButton1Click:Connect(function()
            if cycleOptions then
                local idx = table.find(cycleOptions, Config[modeKey]) or 1
                Config[modeKey] = cycleOptions[idx % #cycleOptions + 1]
                modeBtn.Text = "Mode: " .. Config[modeKey]
            else
                Config[modeKey] = Config[modeKey] == "Hold" and "Toggle" or "Hold"
                modeBtn.Text = Config[modeKey]
            end
        end)
        xOffset = xOffset + 0.18
    end

    if bindKey then
        local bindBtn = Instance.new("TextButton")
        bindBtn.Size = UDim2.new(0, 95, 0, 24)
        bindBtn.Position = UDim2.new(xOffset, 0, 0.5, -12)
        bindBtn.BackgroundColor3 = Color3.fromRGB(30, 30, 30)
        bindBtn.Text = "Bind: " .. Config[bindKey].Name
        bindBtn.TextColor3 = Color3.new(1, 1, 1)
        bindBtn.Font = Enum.Font.GothamSemibold
        bindBtn.TextSize = 11
        bindBtn.BorderSizePixel = 0
        bindBtn.Parent = frame

        local bbCorner = Instance.new("UICorner")
        bbCorner.CornerRadius = UDim.new(0, 4)
        bbCorner.Parent = bindBtn

        local bbStroke = Instance.new("UIStroke")
        bbStroke.Color = Color3.fromRGB(50, 50, 50)
        bbStroke.Parent = bindBtn

        bindButtons[bindKey] = bindBtn
        bindBtn.MouseButton1Click:Connect(function()
            bindBtn.Text = "..."
            bindingTarget = bindKey
        end)
    end
end

local function AddSlider(parentTab, text, configKey, min, max)
    local frame = Instance.new("Frame")
    frame.Size = UDim2.new(1, -8, 0, 50)
    frame.BackgroundColor3 = Color3.fromRGB(22, 22, 22)
    frame.BorderSizePixel = 0
    frame.Parent = parentTab

    local fCorner = Instance.new("UICorner")
    fCorner.CornerRadius = UDim.new(0, 6)
    fCorner.Parent = frame

    local label = Instance.new("TextLabel")
    label.Size = UDim2.new(1, -24, 0, 20)
    label.Position = UDim2.new(0, 12, 0, 6)
    label.Text = text
    label.TextColor3 = Color3.fromRGB(220, 220, 220)
    label.Font = Enum.Font.GothamMedium
    label.TextSize = 13
    label.TextXAlignment = Enum.TextXAlignment.Left
    label.BackgroundTransparency = 1
    label.Parent = frame

    local valueLabel = Instance.new("TextLabel")
    valueLabel.Size = UDim2.new(0, 50, 0, 20)
    valueLabel.Position = UDim2.new(1, -62, 0, 6)
    valueLabel.Text = tostring(Config[configKey])
    valueLabel.TextColor3 = Color3.new(1, 1, 1)
    valueLabel.Font = Enum.Font.GothamBold
    valueLabel.TextSize = 13
    valueLabel.TextXAlignment = Enum.TextXAlignment.Right
    valueLabel.BackgroundTransparency = 1
    valueLabel.Parent = frame

    local sliderBg = Instance.new("Frame")
    sliderBg.Size = UDim2.new(1, -24, 0, 6)
    sliderBg.Position = UDim2.new(0, 12, 0, 34)
    sliderBg.BackgroundColor3 = Color3.fromRGB(40, 40, 40)
    sliderBg.BorderSizePixel = 0
    sliderBg.Parent = frame

    local sbCorner = Instance.new("UICorner")
    sbCorner.CornerRadius = UDim.new(1, 0)
    sbCorner.Parent = sliderBg

    local sliderFill = Instance.new("Frame")
    sliderFill.Size = UDim2.new((Config[configKey] - min) / (max - min), 0, 1, 0)
    sliderFill.BackgroundColor3 = Color3.new(1, 1, 1)
    sliderFill.BorderSizePixel = 0
    sliderFill.Parent = sliderBg

    local sfCorner = Instance.new("UICorner")
    sfCorner.CornerRadius = UDim.new(1, 0)
    sfCorner.Parent = sliderFill

    local sliderKnob = Instance.new("Frame")
    sliderKnob.Size = UDim2.new(0, 12, 0, 12)
    sliderKnob.Position = UDim2.new(1, -6, 0.5, -6)
    sliderKnob.BackgroundColor3 = Color3.new(1, 1, 1)
    sliderKnob.Parent = sliderFill

    local skCorner = Instance.new("UICorner")
    skCorner.CornerRadius = UDim.new(1, 0)
    skCorner.Parent = sliderKnob

    local trigger = Instance.new("TextButton")
    trigger.Size = UDim2.new(1, 0, 1, 20)
    trigger.Position = UDim2.new(0, 0, 0.5, -10)
    trigger.BackgroundTransparency = 1
    trigger.Text = ""
    trigger.Parent = sliderBg

    local isSliding = false

    trigger.InputBegan:Connect(function(i)
        if i.UserInputType == Enum.UserInputType.MouseButton1 then
            isSliding = true
            TweenService:Create(sliderKnob, TweenInfoFast, {
                Size = UDim2.new(0, 16, 0, 16),
                Position = UDim2.new(1, -8, 0.5, -8)
            }):Play()
        end
    end)

    table.insert(connections, UIS.InputEnded:Connect(function(i)
        if i.UserInputType == Enum.UserInputType.MouseButton1 and isSliding then
            isSliding = false
            TweenService:Create(sliderKnob, TweenInfoFast, {
                Size = UDim2.new(0, 12, 0, 12),
                Position = UDim2.new(1, -6, 0.5, -6)
            }):Play()
        end
    end))

    table.insert(connections, RunService.RenderStepped:Connect(function()
        if isSliding then
            local relPos = math.clamp(
                (UIS:GetMouseLocation().X - sliderBg.AbsolutePosition.X) / sliderBg.AbsoluteSize.X,
                0, 1
            )
            local value = math.floor(min + (max - min) * relPos)
            Config[configKey] = value
            valueLabel.Text = tostring(value)
            TweenService:Create(sliderFill, TweenInfo.new(0.05), {
                Size = UDim2.new(relPos, 0, 1, 0)
            }):Play()

            if configKey == "Aim_FOV_Radius" then
                FOV_Circle.Size = UDim2.new(0, value * 2, 0, value * 2)
            end
        end
    end))
end

local function AddRGBPicker(parentTab, text, colorKey)
    local frame = Instance.new("Frame")
    frame.Size = UDim2.new(1, -8, 0, 90)
    frame.BackgroundColor3 = Color3.fromRGB(22, 22, 22)
    frame.BorderSizePixel = 0
    frame.Parent = parentTab

    local fCorner = Instance.new("UICorner")
    fCorner.CornerRadius = UDim.new(0, 6)
    fCorner.Parent = frame

    local label = Instance.new("TextLabel")
    label.Size = UDim2.new(1, -24, 0, 20)
    label.Position = UDim2.new(0, 12, 0, 6)
    label.Text = text
    label.TextColor3 = Color3.fromRGB(220, 220, 220)
    label.Font = Enum.Font.GothamMedium
    label.TextSize = 13
    label.TextXAlignment = Enum.TextXAlignment.Left
    label.BackgroundTransparency = 1
    label.Parent = frame

    local preview = Instance.new("Frame")
    preview.Size = UDim2.new(0, 30, 0, 14)
    preview.Position = UDim2.new(1, -42, 0, 9)
    preview.BackgroundColor3 = Config[colorKey]
    preview.BorderSizePixel = 0
    preview.Parent = frame

    local pCorner = Instance.new("UICorner")
    pCorner.CornerRadius = UDim.new(0, 4)
    pCorner.Parent = preview

    local function MakeColorSlider(yOffset, cType, default)
        local sliderBg = Instance.new("Frame")
        sliderBg.Size = UDim2.new(1, -24, 0, 6)
        sliderBg.Position = UDim2.new(0, 12, 0, yOffset)
        sliderBg.BackgroundColor3 = Color3.fromRGB(40, 40, 40)
        sliderBg.BorderSizePixel = 0
        sliderBg.Parent = frame

        local sbCorner = Instance.new("UICorner")
        sbCorner.CornerRadius = UDim.new(1, 0)
        sbCorner.Parent = sliderBg

        local fillColor
        if cType == "R" then fillColor = Color3.fromRGB(255, 50, 50)
        elseif cType == "G" then fillColor = Color3.fromRGB(50, 255, 50)
        else fillColor = Color3.fromRGB(50, 50, 255) end

        local sliderFill = Instance.new("Frame")
        sliderFill.Size = UDim2.new(default / 255, 0, 1, 0)
        sliderFill.BackgroundColor3 = fillColor
        sliderFill.BorderSizePixel = 0
        sliderFill.Parent = sliderBg

        local sfCorner = Instance.new("UICorner")
        sfCorner.CornerRadius = UDim.new(1, 0)
        sfCorner.Parent = sliderFill

        local trigger = Instance.new("TextButton")
        trigger.Size = UDim2.new(1, 0, 1, 10)
        trigger.Position = UDim2.new(0, 0, 0.5, -5)
        trigger.BackgroundTransparency = 1
        trigger.Text = ""
        trigger.Parent = sliderBg

        local isSliding = false

        trigger.InputBegan:Connect(function(i)
            if i.UserInputType == Enum.UserInputType.MouseButton1 then
                isSliding = true
            end
        end)

        table.insert(connections, UIS.InputEnded:Connect(function(i)
            if i.UserInputType == Enum.UserInputType.MouseButton1 then
                isSliding = false
            end
        end))

        table.insert(connections, RunService.RenderStepped:Connect(function()
            if isSliding then
                local relPos = math.clamp(
                    (UIS:GetMouseLocation().X - sliderBg.AbsolutePosition.X) / sliderBg.AbsoluteSize.X,
                    0, 1
                )
                sliderFill.Size = UDim2.new(relPos, 0, 1, 0)

                local currentC = Config[colorKey]
                local r, g, b
                if cType == "R" then
                    r = relPos * 255; g = currentC.G * 255; b = currentC.B * 255
                elseif cType == "G" then
                    r = currentC.R * 255; g = relPos * 255; b = currentC.B * 255
                else
                    r = currentC.R * 255; g = currentC.G * 255; b = relPos * 255
                end
                Config[colorKey] = Color3.fromRGB(r, g, b)
                preview.BackgroundColor3 = Config[colorKey]
            end
        end))
    end

    MakeColorSlider(35, "R", Config[colorKey].R * 255)
    MakeColorSlider(55, "G", Config[colorKey].G * 255)
    MakeColorSlider(75, "B", Config[colorKey].B * 255)
end

local function AddInfoLabel(parentTab, text, textColor)
    local frame = Instance.new("Frame")
    frame.Size = UDim2.new(1, -8, 0, 30)
    frame.BackgroundColor3 = Color3.fromRGB(18, 18, 25)
    frame.BorderSizePixel = 0
    frame.Parent = parentTab

    local fCorner = Instance.new("UICorner")
    fCorner.CornerRadius = UDim.new(0, 6)
    fCorner.Parent = frame

    local lbl = Instance.new("TextLabel")
    lbl.Size = UDim2.new(1, -20, 1, 0)
    lbl.Position = UDim2.new(0, 10, 0, 0)
    lbl.Text = text
    lbl.TextColor3 = textColor or Color3.fromRGB(150, 150, 170)
    lbl.Font = Enum.Font.GothamMedium
    lbl.TextSize = 12
    lbl.TextXAlignment = Enum.TextXAlignment.Left
    lbl.BackgroundTransparency = 1
    lbl.Parent = frame

    return lbl
end

local function AddButton(parentTab, text, callback, btnColor)
    local frame = Instance.new("Frame")
    frame.Size = UDim2.new(1, -8, 0, 36)
    frame.BackgroundColor3 = Color3.fromRGB(22, 22, 22)
    frame.BorderSizePixel = 0
    frame.Parent = parentTab

    local fCorner = Instance.new("UICorner")
    fCorner.CornerRadius = UDim.new(0, 6)
    fCorner.Parent = frame

    local btn = Instance.new("TextButton")
    btn.Size = UDim2.new(1, -20, 0, 26)
    btn.Position = UDim2.new(0, 10, 0.5, -13)
    btn.BackgroundColor3 = btnColor or Color3.fromRGB(40, 30, 60)
    btn.Text = text
    btn.TextColor3 = Color3.new(1, 1, 1)
    btn.Font = Enum.Font.GothamBold
    btn.TextSize = 12
    btn.BorderSizePixel = 0
    btn.Parent = frame

    local bCorner = Instance.new("UICorner")
    bCorner.CornerRadius = UDim.new(0, 4)
    bCorner.Parent = btn

    btn.MouseButton1Click:Connect(callback)
end

-- ════════════════════════════════════════
-- TAB İÇERİKLERİ
-- ════════════════════════════════════════

local espPosOpts = {"Top", "Bottom", "Left", "Right"}

-- Aim Tab
AddSetting(TabAim, "Enable Aimbot", "Aim_Enabled", "Aim_Bind")
AddSetting(TabAim, "Silent Aim", "Aim_Silent")
AddSetting(TabAim, "Aim Target", nil, nil, "Aim_Target", {"Head", "Torso"})
AddSetting(TabAim, "Team Check", "Aim_TeamCheck")
AddSetting(TabAim, "Show FOV Circle", "Aim_FOV_Show")
AddSlider(TabAim, "FOV Radius", "Aim_FOV_Radius", 10, 600)
AddSlider(TabAim, "Smoothness", "Aim_Smooth", 1, 100)
AddSlider(TabAim, "Prediction", "Aim_Predict", 0, 20)

-- ESP Tab
AddSetting(TabESP, "Master Switch", "ESP_Enabled")
AddSetting(TabESP, "Team Check", "ESP_TeamCheck")
AddSetting(TabESP, "Show on Self", "ESP_ShowSelf")
AddSetting(TabESP, "Draw Boxes", "ESP_Box")
AddSetting(TabESP, "Draw Skeleton", "ESP_Skeleton")
AddSetting(TabESP, "Draw Tracers", "ESP_Tracers")
AddSetting(TabESP, "HP Bar", "ESP_HPBar", nil, "ESP_HPBar_Pos", espPosOpts)
AddSetting(TabESP, "Name", "ESP_Name", nil, "ESP_Name_Pos", espPosOpts)
AddSetting(TabESP, "Distance", "ESP_Dist", nil, "ESP_Dist_Pos", espPosOpts)
AddSetting(TabESP, "HP Text", "ESP_HPText", nil, "ESP_HPText_Pos", espPosOpts)
AddSetting(TabESP, "Faction", "ESP_Faction", nil, "ESP_Faction_Pos", espPosOpts)

-- Movement Tab
AddSetting(TabMovement, "Teleport", "TP_Enabled", "TP_Bind")
AddSetting(TabMovement, "Noclip", "Noclip_Enabled", "Noclip_Bind")
AddSetting(TabMovement, "Fly", "Fly_Enabled", "Fly_Bind")
AddSlider(TabMovement, "Fly Speed", "Fly_Speed", 10, 600)

-- Rivals Tab
AddInfoLabel(TabRivals, "⚔ RIVALS RANKED DETECTION + SCORE TRACKER", Color3.fromRGB(255, 200, 100))
AddInfoLabel(TabRivals, "────────────────────────────────", Color3.fromRGB(50, 50, 60))
AddSetting(TabRivals, "Auto Detect Match", "Auto_Detect_Match")
AddSetting(TabRivals, "Auto Enable ESP", "Auto_Enable_ESP")
AddSetting(TabRivals, "Auto Enable Aim", "Auto_Enable_Aim")
AddSlider(TabRivals, "Detect Sensitivity (studs)", "Match_Detect_Sensitivity", 20, 200)
AddInfoLabel(TabRivals, "────────────────────────────────", Color3.fromRGB(50, 50, 60))

local matchStatusLabel = AddInfoLabel(TabRivals, "● Status: Waiting for match...", Color3.fromRGB(100, 100, 130))
local matchTypeStatusLabel = AddInfoLabel(TabRivals, "● Mode: None detected", Color3.fromRGB(100, 100, 130))
local matchPlayersLabel = AddInfoLabel(TabRivals, "● Nearby Players: 0", Color3.fromRGB(100, 100, 130))
local matchTeleportLabel = AddInfoLabel(TabRivals, "● Teleports Detected: 0", Color3.fromRGB(100, 100, 130))

AddInfoLabel(TabRivals, "────────────────────────────────", Color3.fromRGB(50, 50, 60))
AddInfoLabel(TabRivals, "📊 SCORE TRACKER", Color3.fromRGB(255, 180, 80))

local scoreMyLabel = AddInfoLabel(TabRivals, "● ME: 0", Color3.fromRGB(80, 220, 100))
local scoreEnemyLabel = AddInfoLabel(TabRivals, "● ENEMY: 0", Color3.fromRGB(231, 76, 60))
local scoreRoundsLabel = AddInfoLabel(TabRivals, "● Rounds Played: 0", Color3.fromRGB(140, 140, 170))

AddInfoLabel(TabRivals, "────────────────────────────────", Color3.fromRGB(50, 50, 60))

AddButton(TabRivals, "🔄 Reset Score", function()
    ScoreTracker.MyScore = 0
    ScoreTracker.EnemyScore = 0
    ScoreTracker.MatchRounds = 0
    ScoreTracker.TeleportCount = 0
    RivalsDetection.TeleportCount = 0
    RivalsDetection.IsInMatch = false
    RivalsDetection.MatchType = "None"
end, Color3.fromRGB(60, 30, 30))

AddInfoLabel(TabRivals, "────────────────────────────────", Color3.fromRGB(50, 50, 60))
AddInfoLabel(TabRivals, "ℹ Teleport detected = Round/Match found", Color3.fromRGB(80, 80, 110))
AddInfoLabel(TabRivals, "ℹ Score updates on round win/loss", Color3.fromRGB(80, 80, 110))
AddInfoLabel(TabRivals, "ℹ Notification shows live score", Color3.fromRGB(80, 80, 110))

-- Settings Tab
AddSetting(TabSettings, "Show Watermark", "Watermark_Enabled")
AddSetting(TabSettings, "Show Keybinds", "Keybinds_Enabled")
AddSetting(TabSettings, "Rainbow UI", "Rainbow_Enabled")
AddSlider(TabSettings, "Rainbow Speed", "Rainbow_Speed", 1, 10)
AddRGBPicker(TabSettings, "UI Color 1", "UIColor1")
AddRGBPicker(TabSettings, "UI Color 2", "UIColor2")

-- ════════════════════════════════════════
-- WATERMARK & KEYBINDS
-- ════════════════════════════════════════

local WMScreen = Instance.new("ScreenGui")
WMScreen.Name = "Tex8s_WM"
WMScreen.Parent = CoreGui

local WMFrame = Instance.new("Frame")
WMFrame.AnchorPoint = Vector2.new(1, 0)
WMFrame.Position = UDim2.new(1, -10, 0, 10)
WMFrame.Size = UDim2.new(0, 0, 0, 32)
WMFrame.AutomaticSize = Enum.AutomaticSize.X
WMFrame.BackgroundColor3 = Color3.fromRGB(15, 15, 15)
WMFrame.BorderSizePixel = 0
WMFrame.Parent = WMScreen

local wmCorner = Instance.new("UICorner")
wmCorner.CornerRadius = UDim.new(0, 6)
wmCorner.Parent = WMFrame

local WMPad = Instance.new("UIPadding")
WMPad.PaddingLeft = UDim.new(0, 10)
WMPad.PaddingRight = UDim.new(0, 10)
WMPad.Parent = WMFrame

local WMStroke = Instance.new("UIStroke")
WMStroke.Thickness = 1.5
WMStroke.Parent = WMFrame

local WMGrad = Instance.new("UIGradient")
WMGrad.Parent = WMStroke
table.insert(dynamicGradientObjects, WMGrad)

local WMText = Instance.new("TextLabel")
WMText.Size = UDim2.new(0, 0, 1, 0)
WMText.AutomaticSize = Enum.AutomaticSize.X
WMText.BackgroundTransparency = 1
WMText.TextColor3 = Color3.new(1, 1, 1)
WMText.Font = Enum.Font.GothamSemibold
WMText.TextSize = 14
WMText.Parent = WMFrame

local KBFrame = Instance.new("Frame")
KBFrame.Size = UDim2.new(0, 160, 0, 30)
KBFrame.Position = UDim2.new(0, 10, 0.5, 0)
KBFrame.BackgroundColor3 = Color3.fromRGB(15, 15, 15)
KBFrame.BorderSizePixel = 0
KBFrame.Parent = WMScreen

local kbCorner = Instance.new("UICorner")
kbCorner.CornerRadius = UDim.new(0, 6)
kbCorner.Parent = KBFrame

local KBStroke = Instance.new("UIStroke")
KBStroke.Thickness = 1.5
KBStroke.Parent = KBFrame

local KBGrad = Instance.new("UIGradient")
KBGrad.Parent = KBStroke
table.insert(dynamicGradientObjects, KBGrad)

local KBTitle = Instance.new("TextLabel")
KBTitle.Size = UDim2.new(1, 0, 0, 26)
KBTitle.BackgroundTransparency = 1
KBTitle.Text = "⌨ Keybinds"
KBTitle.TextColor3 = Color3.new(1, 1, 1)
KBTitle.Font = Enum.Font.GothamBold
KBTitle.TextSize = 12
KBTitle.Parent = KBFrame

local KBLayout = Instance.new("UIListLayout")
KBLayout.SortOrder = Enum.SortOrder.LayoutOrder
KBLayout.Padding = UDim.new(0, 2)
KBLayout.HorizontalAlignment = Enum.HorizontalAlignment.Center
KBLayout.Parent = KBFrame

local function MakeDraggable(frame)
    local isDragging, dInput, dStart, dPos

    frame.InputBegan:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1 then
            isDragging = true
            dStart = input.Position
            dPos = frame.Position
        end
    end)

    frame.InputEnded:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1 then
            isDragging = false
        end
    end)

    table.insert(connections, UIS.InputChanged:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseMovement then
            dInput = input
        end
    end))

    table.insert(connections, RunService.Heartbeat:Connect(function()
        if isDragging and dInput then
            frame.Position = UDim2.new(
                dPos.X.Scale, dPos.X.Offset + (dInput.Position.X - dStart.X),
                dPos.Y.Scale, dPos.Y.Offset + (dInput.Position.Y - dStart.Y)
            )
        end
    end))
end

MakeDraggable(WMFrame)
MakeDraggable(KBFrame)

-- ════════════════════════════════════════
-- ESP OLUŞTURMA
-- ════════════════════════════════════════

local function CreateESP(player)
    if player == LocalPlayer then return end
    local esp = {Drawings = {}, Skeleton = {}, BoxLines = {}, BoxOutlines = {}}

    pcall(function()
        for i = 1, 4 do
            esp.BoxOutlines[i] = Drawing.new("Line")
            esp.BoxOutlines[i].Thickness = 3
            esp.BoxOutlines[i].Color = Color3.new(0, 0, 0)

            esp.BoxLines[i] = Drawing.new("Line")
            esp.BoxLines[i].Thickness = 1.5
        end

        esp.Drawings.Tracer = Drawing.new("Line")
        esp.Drawings.Tracer.Thickness = 1.5

        local function MakeText()
            local txt = Drawing.new("Text")
            txt.Size = 13
            txt.Center = true
            txt.Outline = true
            txt.Color = Color3.new(1, 1, 1)
            return txt
        end

        esp.Drawings.Name = MakeText()
        esp.Drawings.Dist = MakeText()
        esp.Drawings.HPText = MakeText()
        esp.Drawings.Faction = MakeText()

        esp.Drawings.HPBarBg = Drawing.new("Square")
        esp.Drawings.HPBarBg.Filled = true
        esp.Drawings.HPBarBg.Color = Color3.new(0, 0, 0)

        esp.Drawings.HPBarFill = Drawing.new("Square")
        esp.Drawings.HPBarFill.Filled = true

        for i = 1, 15 do
            esp.Skeleton[i] = Drawing.new("Line")
            esp.Skeleton[i].Thickness = 1.5
        end
    end)

    espCache[player] = esp
end

local function RemoveESP(player)
    if espCache[player] then
        pcall(function()
            for i = 1, 4 do
                espCache[player].BoxLines[i]:Remove()
                espCache[player].BoxOutlines[i]:Remove()
            end
            espCache[player].Drawings.Tracer:Remove()
            espCache[player].Drawings.Name:Remove()
            espCache[player].Drawings.Dist:Remove()
            espCache[player].Drawings.HPText:Remove()
            espCache[player].Drawings.Faction:Remove()
            espCache[player].Drawings.HPBarBg:Remove()
            espCache[player].Drawings.HPBarFill:Remove()
            for _, b in pairs(espCache[player].Skeleton) do
                b:Remove()
            end
        end)
        espCache[player] = nil
    end
end

for _, p in ipairs(Players:GetPlayers()) do
    if p ~= LocalPlayer then CreateESP(p) end
end

table.insert(connections, Players.PlayerAdded:Connect(function(p)
    if p ~= LocalPlayer then CreateESP(p) end
end))

table.insert(connections, Players.PlayerRemoving:Connect(RemoveESP))

-- ════════════════════════════════════════
-- SKELETON BONE HELPER
-- ════════════════════════════════════════

local function DrawBoneLine(boneObj, char, p1Name, p2Name, color)
    if not boneObj then return end
    local p1 = char:FindFirstChild(p1Name)
    local p2 = char:FindFirstChild(p2Name)
    if p1 and p2 then
        local pos1, vis1 = Camera:WorldToViewportPoint(p1.Position)
        local pos2, vis2 = Camera:WorldToViewportPoint(p2.Position)
        if vis1 or vis2 then
            boneObj.Visible = true
            boneObj.From = Vector2.new(pos1.X, pos1.Y)
            boneObj.To = Vector2.new(pos2.X, pos2.Y)
            boneObj.Color = color
            return
        end
    end
    boneObj.Visible = false
end

-- ════════════════════════════════════════
-- AİMBOT HEDEF
-- ════════════════════════════════════════

local function GetClosestTarget()
    local closestPlayer = nil
    local shortestDistance = Config.Aim_FOV_Radius
    local mousePos = UIS:GetMouseLocation()

    for _, p in ipairs(Players:GetPlayers()) do
        if p ~= LocalPlayer and p.Character then
            local hum = p.Character:FindFirstChild("Humanoid")
            if hum and hum.Health > 0 then
                if Config.Aim_TeamCheck and p.Team and p.Team == LocalPlayer.Team then continue end
                local partName = Config.Aim_Target == "Torso" and "HumanoidRootPart" or "Head"
                local targetPart = p.Character:FindFirstChild(partName)
                if targetPart then
                    local pos, onScreen = Camera:WorldToViewportPoint(targetPart.Position)
                    if onScreen then
                        local dist = (Vector2.new(pos.X, pos.Y) - mousePos).Magnitude
                        if dist <= shortestDistance then
                            shortestDistance = dist
                            closestPlayer = p
                        end
                    end
                end
            end
        end
    end
    return closestPlayer
end

-- ════════════════════════════════════════════════════════
-- RIVALS RANKED TELEPORT ALGILAMA + SKOR TAKİP DÖNGÜSÜ
-- ════════════════════════════════════════════════════════

task.spawn(function()
    local function initPosition()
        local char = LocalPlayer.Character
        if char then
            local hrp = char:WaitForChild("HumanoidRootPart", 5)
            if hrp then
                RivalsDetection.LastPosition = hrp.Position
            end
        end
    end

    initPosition()

    table.insert(connections, LocalPlayer.CharacterAdded:Connect(function(char)
        task.wait(1)
        local hrp = char:WaitForChild("HumanoidRootPart", 5)
        if hrp then
            task.wait(0.5)
            RivalsDetection.LastPosition = hrp.Position
        end
    end))

    while true do
        task.wait(0.25)

        if not Config.Auto_Detect_Match then continue end

        local char = LocalPlayer.Character
        if not char then continue end
        local hrp = char:FindFirstChild("HumanoidRootPart")
        if not hrp then continue end

        local currentPos = hrp.Position
        RivalsDetection.TeleportThreshold = Config.Match_Detect_Sensitivity

        if RivalsDetection.LastPosition then
            local distance = (currentPos - RivalsDetection.LastPosition).Magnitude

            if distance > RivalsDetection.TeleportThreshold and not RivalsDetection.MatchCooldown then
                RivalsDetection.TeleportCount = RivalsDetection.TeleportCount + 1

                task.wait(1.5)

                local nearbyCount = CountNearbyPlayers(150)
                local matchType = GuessMatchType(nearbyCount)
                local rivalsInfo = ScanForRivalsMatchData()

                local detectedType = "Ranked Match"
                if rivalsInfo.inMatch and rivalsInfo.matchType ~= "None" then
                    detectedType = rivalsInfo.matchType
                elseif matchType ~= "Unknown" then
                    detectedType = matchType
                end

                RivalsDetection.IsInMatch = true
                RivalsDetection.MatchType = detectedType
                RivalsDetection.MatchStartTime = tick()

                -- Oyun verisinden skor okumayı dene
                local gameScore = TryReadScoreFromGame()
                if gameScore.found then
                    if gameScore.myScore then ScoreTracker.MyScore = gameScore.myScore end
                    if gameScore.enemyScore then ScoreTracker.EnemyScore = gameScore.enemyScore end
                end

                print("══════════════════════════════════")
                print("  ⚔ MATCH DETECTED!")
                print("  Type: " .. detectedType)
                print("  Distance: " .. math.floor(distance) .. " studs")
                print("  Nearby: " .. nearbyCount .. " players")
                print("  Score: ME " .. ScoreTracker.MyScore .. " - ENEMY " .. ScoreTracker.EnemyScore)
                print("══════════════════════════════════")

                ShowMatchNotification(detectedType)

                if Config.Auto_Enable_ESP then
                    Config.ESP_Enabled = true
                end
                if Config.Auto_Enable_Aim then
                    Config.Aim_Enabled = true
                end

                RivalsDetection.MatchCooldown = true
                task.delay(15, function()
                    RivalsDetection.MatchCooldown = false
                end)
            end
        end

        RivalsDetection.LastPosition = currentPos
    end
end)

-- ════════════════════════════════════════════════════════
-- SKOR TAKİP DÖNGÜSÜ (Round Win/Loss algılama)
-- ════════════════════════════════════════════════════════

task.spawn(function()
    while true do
        task.wait(0.5)

        if not RivalsDetection.IsInMatch then continue end

        -- Oyun verisinden skor okumayı dene
        local gameScore = TryReadScoreFromGame()
        if gameScore.found then
            if gameScore.myScore then ScoreTracker.MyScore = gameScore.myScore end
            if gameScore.enemyScore then ScoreTracker.EnemyScore = gameScore.enemyScore end
        else
            -- Manuel algılama: Round sonu kontrol
            local result = DetectRoundEnd()
            if result then
                print("══════════════════════════════════")
                print("  📊 ROUND RESULT: " .. (result == "win" and "WIN!" or "LOSS"))
                print("  Score: ME " .. ScoreTracker.MyScore .. " - ENEMY " .. ScoreTracker.EnemyScore)
                print("══════════════════════════════════")

                ShowRoundResultNotification(result)
            end
        end
    end
end)

-- ════════════════════════════════════════════════════════
-- ANA RENDER DÖNGÜSÜ
-- ════════════════════════════════════════════════════════

local framesData = {}
local flyBodyVel = nil
local flyGyro = nil
local lastAntiFallTick = tick()

table.insert(connections, RunService.RenderStepped:Connect(function()
    local cTime = tick()

    table.insert(framesData, cTime)
    while framesData[1] and framesData[1] < cTime - 1 do
        table.remove(framesData, 1)
    end

    local color1, color2
    if Config.Rainbow_Enabled then
        local rc = Color3.fromHSV((cTime * (Config.Rainbow_Speed / 10)) % 1, 1, 1)
        color1 = rc
        color2 = rc:Lerp(Color3.new(0, 0, 0), 0.3)
    else
        color1 = Config.UIColor1
        color2 = Config.UIColor2
    end

    local gradSeq = ColorSequence.new({
        ColorSequenceKeypoint.new(0, color1),
        ColorSequenceKeypoint.new(1, color2)
    })
    for _, grad in pairs(dynamicGradientObjects) do
        grad.Rotation = 90
        grad.Color = gradSeq
    end

    -- FOV Circle
    FOV_Circle.Visible = Config.Aim_FOV_Show and Config.Aim_Enabled
    if FOV_Circle.Visible then
        local mPos = UIS:GetMouseLocation()
        FOV_Circle.Position = UDim2.new(0, mPos.X, 0, mPos.Y)
        FOV_Grad.Color = ColorSequence.new(color1)
    end

    -- Hedef bul
    pcall(function()
        CachedTarget = GetClosestTarget()
    end)

    -- Aimbot (normal)
    if Config.Aim_Enabled and not Config.Aim_Silent and IsBindActive(Config.Aim_Bind) then
        local target = CachedTarget
        if target and target.Character then
            local partName = Config.Aim_Target == "Torso" and "HumanoidRootPart" or "Head"
            local targetPart = target.Character:FindFirstChild(partName)
            if targetPart and mousemoverel then
                local aimPos = targetPart.Position + (targetPart.Velocity * (Config.Aim_Predict / 100))
                local pos, onScreen = Camera:WorldToViewportPoint(aimPos)
                if onScreen then
                    local mousePos = UIS:GetMouseLocation()
                    local sm = math.max(Config.Aim_Smooth, 1)
                    mousemoverel(
                        (pos.X - mousePos.X) / sm,
                        (pos.Y - mousePos.Y) / sm
                    )
                end
            end
        end
    end

    -- Maç göstergesi güncelle
    if RivalsDetection.IsInMatch then
        MatchIndicator.Text = "⚔ " .. ScoreTracker.MyScore .. " - " .. ScoreTracker.EnemyScore
        MatchIndicator.TextColor3 = Color3.fromRGB(80, 220, 100)
        MatchIndicator.BackgroundColor3 = Color3.fromRGB(20, 40, 20)
    else
        MatchIndicator.Text = "● LOBBY"
        MatchIndicator.TextColor3 = Color3.fromRGB(100, 100, 100)
        MatchIndicator.BackgroundColor3 = Color3.fromRGB(30, 30, 30)
    end

    -- Rivals tab canlı güncelle
    pcall(function()
        local nearbyCount = CountNearbyPlayers(100)

        if RivalsDetection.IsInMatch then
            matchStatusLabel.Text = "● Status: IN MATCH ⚔"
            matchStatusLabel.TextColor3 = Color3.fromRGB(80, 220, 100)
        else
            matchStatusLabel.Text = "● Status: Waiting for match..."
            matchStatusLabel.TextColor3 = Color3.fromRGB(100, 100, 130)
        end

        matchTypeStatusLabel.Text = "● Mode: " .. RivalsDetection.MatchType
        matchTypeStatusLabel.TextColor3 = RivalsDetection.IsInMatch and Color3.fromRGB(180, 130, 255) or Color3.fromRGB(100, 100, 130)

        matchPlayersLabel.Text = "● Nearby Players: " .. nearbyCount
        matchPlayersLabel.TextColor3 = nearbyCount > 0 and Color3.fromRGB(255, 200, 100) or Color3.fromRGB(100, 100, 130)

        matchTeleportLabel.Text = "● Teleports Detected: " .. RivalsDetection.TeleportCount
        matchTeleportLabel.TextColor3 = RivalsDetection.TeleportCount > 0 and Color3.fromRGB(140, 100, 255) or Color3.fromRGB(100, 100, 130)

        -- Skor labelları güncelle
        scoreMyLabel.Text = "● ME: " .. ScoreTracker.MyScore
        scoreMyLabel.TextColor3 = Color3.fromRGB(80, 220, 100)

        scoreEnemyLabel.Text = "● ENEMY: " .. ScoreTracker.EnemyScore
        scoreEnemyLabel.TextColor3 = Color3.fromRGB(231, 76, 60)

        scoreRoundsLabel.Text = "● Rounds Played: " .. ScoreTracker.MatchRounds
    end)

    -- Watermark
    WMFrame.Visible = Config.Watermark_Enabled
    if Config.Watermark_Enabled then
        local ping = "N/A"
        pcall(function()
            ping = math.floor(Stats.Network.ServerStatsItem["Data Ping"]:GetValue())
        end)
        local scoreStr = RivalsDetection.IsInMatch and (" | ME: " .. ScoreTracker.MyScore .. " - ENEMY: " .. ScoreTracker.EnemyScore) or ""
        WMText.Text = string.format("TEX8S Premium | %s | %d FPS | %s ms%s", LocalPlayer.Name, #framesData, ping, scoreStr)
        WMText.TextColor3 = color1
    end

    -- Keybinds
    KBFrame.Visible = Config.Keybinds_Enabled
    if Config.Keybinds_Enabled then
        for _, child in pairs(KBFrame:GetChildren()) do
            if child:IsA("Frame") then child:Destroy() end
        end

        local activeBinds = {}
        if Config.Aim_Enabled then
            table.insert(activeBinds, {name = "AimBot", key = Config.Aim_Bind.Name})
        end
        if Config.ESP_Enabled then
            table.insert(activeBinds, {name = "ESP Master", key = "Active"})
        end
        if Config.Fly_Enabled then
            table.insert(activeBinds, {name = "Fly", key = Config.Fly_Bind.Name})
        end
        if Config.Noclip_Enabled then
            table.insert(activeBinds, {name = "Noclip", key = Config.Noclip_Bind.Name})
        end
        if RivalsDetection.IsInMatch then
            table.insert(activeBinds, {name = "⚔ " .. ScoreTracker.MyScore .. "-" .. ScoreTracker.EnemyScore, key = "LIVE"})
        end

        for _, bind in ipairs(activeBinds) do
            local row = Instance.new("Frame")
            row.Size = UDim2.new(1, -10, 0, 20)
            row.BackgroundTransparency = 1
            row.Parent = KBFrame

            local kn = Instance.new("TextLabel")
            kn.Size = UDim2.new(0, 30, 1, 0)
            kn.BackgroundColor3 = Color3.fromRGB(30, 30, 30)
            kn.Text = bind.key
            kn.TextColor3 = color1
            kn.Font = Enum.Font.GothamBold
            kn.TextSize = 10
            kn.Parent = row

            local knCorner = Instance.new("UICorner")
            knCorner.CornerRadius = UDim.new(0, 4)
            knCorner.Parent = kn

            local bn = Instance.new("TextLabel")
            bn.Size = UDim2.new(1, -40, 1, 0)
            bn.Position = UDim2.new(0, 35, 0, 0)
            bn.BackgroundTransparency = 1
            bn.Text = bind.name
            bn.TextColor3 = Color3.fromRGB(200, 200, 200)
            bn.Font = Enum.Font.GothamMedium
            bn.TextSize = 11
            bn.TextXAlignment = Enum.TextXAlignment.Left
            bn.Parent = row
        end

        KBFrame.Size = UDim2.new(0, 160, 0, 30 + (#activeBinds * 22))
        KBTitle.TextColor3 = color1
    end

    -- ═══════════════════════
    -- ESP RENDER
    -- ═══════════════════════
    for player, esp in pairs(espCache) do
        local isVisible = false
        pcall(function()
            local char = player.Character
            local isSelf = (player == LocalPlayer)
            if Config.ESP_Enabled and char
                and char:FindFirstChild("HumanoidRootPart")
                and char:FindFirstChild("Head")
                and char:FindFirstChild("Humanoid")
                and char.Humanoid.Health > 0
                and (not isSelf or Config.ESP_ShowSelf) then

                if Config.ESP_TeamCheck and player.Team and player.Team == LocalPlayer.Team and not isSelf then
                    return
                end

                local hrp = char.HumanoidRootPart
                local head = char.Head
                local headPos, onScreen1 = Camera:WorldToViewportPoint(head.Position + Vector3.new(0, 0.5, 0))

                local leftFoot = char:FindFirstChild("LeftFoot") or char:FindFirstChild("Left Leg")
                local rightFoot = char:FindFirstChild("RightFoot") or char:FindFirstChild("Right Leg")
                local bottomY = hrp.Position.Y - 3
                if leftFoot and rightFoot then
                    bottomY = math.min(leftFoot.Position.Y, rightFoot.Position.Y) - 0.2
                end
                local bottomPos, onScreen2 = Camera:WorldToViewportPoint(Vector3.new(hrp.Position.X, bottomY, hrp.Position.Z))

                if onScreen1 or onScreen2 then
                    isVisible = true
                    local height = math.abs(headPos.Y - bottomPos.Y)
                    local width = height * 0.55
                    local minX = headPos.X - width / 2
                    local minY = headPos.Y
                    local maxX = headPos.X + width / 2
                    local maxY = bottomPos.Y

                    -- Box
                    if Config.ESP_Box and esp.BoxLines then
                        local TL = Vector2.new(minX, minY)
                        local TR = Vector2.new(maxX, minY)
                        local BL = Vector2.new(minX, maxY)
                        local BR = Vector2.new(maxX, maxY)

                        local function setLine(l, f, t, v, col)
                            l.Visible = v
                            if v then l.From = f; l.To = t; l.Color = col end
                        end

                        setLine(esp.BoxOutlines[1], TL, TR, true, Color3.new(0, 0, 0))
                        setLine(esp.BoxOutlines[2], TR, BR, true, Color3.new(0, 0, 0))
                        setLine(esp.BoxOutlines[3], BR, BL, true, Color3.new(0, 0, 0))
                        setLine(esp.BoxOutlines[4], BL, TL, true, Color3.new(0, 0, 0))

                        setLine(esp.BoxLines[1], TL, TR, true, color1)
                        setLine(esp.BoxLines[2], TR, BR, true, color1)
                        setLine(esp.BoxLines[3], BR, BL, true, color1)
                        setLine(esp.BoxLines[4], BL, TL, true, color1)
                    else
                        for i = 1, 4 do
                            esp.BoxLines[i].Visible = false
                            esp.BoxOutlines[i].Visible = false
                        end
                    end

                    -- HP Bar
                    if Config.ESP_HPBar then
                        esp.Drawings.HPBarBg.Visible = true
                        esp.Drawings.HPBarFill.Visible = true
                        local hpPct = math.clamp(char.Humanoid.Health / char.Humanoid.MaxHealth, 0, 1)
                        local barT = 3
                        esp.Drawings.HPBarFill.Color = GetCIELUVColor(hpPct)

                        if Config.ESP_HPBar_Pos == "Left" then
                            esp.Drawings.HPBarBg.Position = Vector2.new(minX - barT - 3, minY - 1)
                            esp.Drawings.HPBarBg.Size = Vector2.new(barT + 2, height + 2)
                            esp.Drawings.HPBarFill.Position = Vector2.new(minX - barT - 2, minY + height - (height * hpPct))
                            esp.Drawings.HPBarFill.Size = Vector2.new(barT, height * hpPct)
                        elseif Config.ESP_HPBar_Pos == "Right" then
                            esp.Drawings.HPBarBg.Position = Vector2.new(maxX + 2, minY - 1)
                            esp.Drawings.HPBarBg.Size = Vector2.new(barT + 2, height + 2)
                            esp.Drawings.HPBarFill.Position = Vector2.new(maxX + 3, minY + height - (height * hpPct))
                            esp.Drawings.HPBarFill.Size = Vector2.new(barT, height * hpPct)
                        elseif Config.ESP_HPBar_Pos == "Bottom" then
                            esp.Drawings.HPBarBg.Position = Vector2.new(minX - 1, maxY + 2)
                            esp.Drawings.HPBarBg.Size = Vector2.new(width + 2, barT + 2)
                            esp.Drawings.HPBarFill.Position = Vector2.new(minX, maxY + 3)
                            esp.Drawings.HPBarFill.Size = Vector2.new(width * hpPct, barT)
                        else
                            esp.Drawings.HPBarBg.Position = Vector2.new(minX - 1, minY - barT - 3)
                            esp.Drawings.HPBarBg.Size = Vector2.new(width + 2, barT + 2)
                            esp.Drawings.HPBarFill.Position = Vector2.new(minX, minY - barT - 2)
                            esp.Drawings.HPBarFill.Size = Vector2.new(width * hpPct, barT)
                        end
                    else
                        esp.Drawings.HPBarBg.Visible = false
                        esp.Drawings.HPBarFill.Visible = false
                    end

                    -- Tracer
                    if Config.ESP_Tracers then
                        esp.Drawings.Tracer.Visible = true
                        esp.Drawings.Tracer.Color = color1
                        esp.Drawings.Tracer.From = Vector2.new(Camera.ViewportSize.X / 2, Camera.ViewportSize.Y)
                        esp.Drawings.Tracer.To = Vector2.new(headPos.X, maxY)
                    else
                        esp.Drawings.Tracer.Visible = false
                    end

                    -- Text drawers
                    local function drawT(d, f, p, txt, col)
                        if Config[f] then
                            d.Visible = true
                            d.Text = txt
                            if col then d.Color = col end
                            local b = d.TextBounds
                            if Config[p] == "Top" then
                                d.Position = Vector2.new(headPos.X, minY - b.Y - (Config.ESP_HPBar and Config.ESP_HPBar_Pos == "Top" and 6 or 2))
                            elseif Config[p] == "Bottom" then
                                d.Position = Vector2.new(headPos.X, maxY + (Config.ESP_HPBar and Config.ESP_HPBar_Pos == "Bottom" and 6 or 2))
                            elseif Config[p] == "Left" then
                                d.Position = Vector2.new(minX - b.X / 2 - (Config.ESP_HPBar and Config.ESP_HPBar_Pos == "Left" and 8 or 4), minY + height / 2 - b.Y / 2)
                            elseif Config[p] == "Right" then
                                d.Position = Vector2.new(maxX + b.X / 2 + (Config.ESP_HPBar and Config.ESP_HPBar_Pos == "Right" and 8 or 4), minY + height / 2 - b.Y / 2)
                            end
                        else
                            d.Visible = false
                        end
                    end

                    drawT(esp.Drawings.Name, "ESP_Name", "ESP_Name_Pos", player.Name, Color3.new(1, 1, 1))
                    drawT(esp.Drawings.Faction, "ESP_Faction", "ESP_Faction_Pos",
                        player.Team and ("[" .. player.Team.Name .. "]") or "",
                        player.TeamColor and player.TeamColor.Color or Color3.new(1, 1, 1))
                    drawT(esp.Drawings.HPText, "ESP_HPText", "ESP_HPText_Pos",
                        math.floor(char.Humanoid.Health) .. " HP",
                        GetCIELUVColor(char.Humanoid.Health / char.Humanoid.MaxHealth))

                    local distStr = ""
                    if LocalPlayer.Character and LocalPlayer.Character:FindFirstChild("HumanoidRootPart") then
                        distStr = math.floor((LocalPlayer.Character.HumanoidRootPart.Position - hrp.Position).Magnitude) .. "m"
                    end
                    drawT(esp.Drawings.Dist, "ESP_Dist", "ESP_Dist_Pos", distStr, Color3.new(1, 1, 1))

                    -- Skeleton
                    if Config.ESP_Skeleton and esp.Skeleton then
                        if char:FindFirstChild("UpperTorso") then
                            DrawBoneLine(esp.Skeleton[1], char, "Head", "UpperTorso", color1)
                            DrawBoneLine(esp.Skeleton[2], char, "UpperTorso", "LowerTorso", color1)
                            DrawBoneLine(esp.Skeleton[3], char, "UpperTorso", "LeftUpperArm", color1)
                            DrawBoneLine(esp.Skeleton[4], char, "LeftUpperArm", "LeftLowerArm", color1)
                            DrawBoneLine(esp.Skeleton[5], char, "LeftLowerArm", "LeftHand", color1)
                            DrawBoneLine(esp.Skeleton[6], char, "UpperTorso", "RightUpperArm", color1)
                            DrawBoneLine(esp.Skeleton[7], char, "RightUpperArm", "RightLowerArm", color1)
                            DrawBoneLine(esp.Skeleton[8], char, "RightLowerArm", "RightHand", color1)
                            DrawBoneLine(esp.Skeleton[9], char, "LowerTorso", "LeftUpperLeg", color1)
                            DrawBoneLine(esp.Skeleton[10], char, "LeftUpperLeg", "LeftLowerLeg", color1)
                            DrawBoneLine(esp.Skeleton[11], char, "LeftLowerLeg", "LeftFoot", color1)
                            DrawBoneLine(esp.Skeleton[12], char, "LowerTorso", "RightUpperLeg", color1)
                            DrawBoneLine(esp.Skeleton[13], char, "RightUpperLeg", "RightLowerLeg", color1)
                            DrawBoneLine(esp.Skeleton[14], char, "RightLowerLeg", "RightFoot", color1)
                        elseif char:FindFirstChild("Torso") then
                            DrawBoneLine(esp.Skeleton[1], char, "Head", "Torso", color1)
                            DrawBoneLine(esp.Skeleton[2], char, "Torso", "Left Arm", color1)
                            DrawBoneLine(esp.Skeleton[3], char, "Torso", "Right Arm", color1)
                            DrawBoneLine(esp.Skeleton[4], char, "Torso", "Left Leg", color1)
                            DrawBoneLine(esp.Skeleton[5], char, "Torso", "Right Leg", color1)
                            for i = 6, 15 do esp.Skeleton[i].Visible = false end
                        end
                    else
                        if esp.Skeleton then
                            for _, bone in pairs(esp.Skeleton) do
                                bone.Visible = false
                            end
                        end
                    end
                end
            end
        end)

        if not isVisible then
            pcall(function()
                for i = 1, 4 do
                    esp.BoxLines[i].Visible = false
                    esp.BoxOutlines[i].Visible = false
                end
                esp.Drawings.Tracer.Visible = false
                esp.Drawings.Name.Visible = false
                esp.Drawings.Dist.Visible = false
                esp.Drawings.HPText.Visible = false
                esp.Drawings.Faction.Visible = false
                esp.Drawings.HPBarBg.Visible = false
                esp.Drawings.HPBarFill.Visible = false
                if esp.Skeleton then
                    for _, bone in pairs(esp.Skeleton) do
                        bone.Visible = false
                    end
                end
            end)
        end
    end

    -- ═══════════════════════
    -- FLY
    -- ═══════════════════════
    local char = LocalPlayer.Character
    local hum = char and char:FindFirstChild("Humanoid")
    local hrp = char and char:FindFirstChild("HumanoidRootPart")
    local inVehicle = false
    local vehicleRoot = nil

    if hum and hum.SeatPart then
        inVehicle = true
        local vehicleModel = hum.SeatPart:FindFirstAncestorOfClass("Model")
        vehicleRoot = (vehicleModel and vehicleModel.PrimaryPart) or hum.SeatPart
    end

    local flyPart = inVehicle and vehicleRoot or hrp

    if Config.Fly_Enabled and flyPart then
        if not flyBodyVel then
            flyBodyVel = Instance.new("BodyVelocity")
            flyBodyVel.Name = "Tex8s_Fly"
            flyBodyVel.MaxForce = Vector3.new(math.huge, math.huge, math.huge)
            flyBodyVel.Parent = flyPart
        end
        if not flyGyro then
            flyGyro = Instance.new("BodyGyro")
            flyGyro.Name = "Tex8s_FlyGyro"
            flyGyro.MaxTorque = Vector3.new(math.huge, math.huge, math.huge)
            flyGyro.P = 15000
            flyGyro.D = 500
            flyGyro.Parent = flyPart
        end

        if flyBodyVel.Parent ~= flyPart then flyBodyVel.Parent = flyPart end
        if flyGyro.Parent ~= flyPart then flyGyro.Parent = flyPart end

        local moveDir = Vector3.zero
        if UIS:IsKeyDown(Enum.KeyCode.W) then moveDir = moveDir + Camera.CFrame.LookVector end
        if UIS:IsKeyDown(Enum.KeyCode.S) then moveDir = moveDir - Camera.CFrame.LookVector end
        if UIS:IsKeyDown(Enum.KeyCode.A) then moveDir = moveDir - Camera.CFrame.RightVector end
        if UIS:IsKeyDown(Enum.KeyCode.D) then moveDir = moveDir + Camera.CFrame.RightVector end
        flyBodyVel.Velocity = moveDir * Config.Fly_Speed

        if inVehicle then
            flyGyro.CFrame = Camera.CFrame
        else
            flyGyro.CFrame = CFrame.new(flyPart.Position, flyPart.Position + Camera.CFrame.LookVector * Vector3.new(1, 0, 1))
        end

        if Config.Fly_AntiFall and not inVehicle and cTime - lastAntiFallTick > 0.1 then
            flyPart.CFrame = flyPart.CFrame * CFrame.new(0, 0.05, 0)
            lastAntiFallTick = cTime
        end
    else
        if flyBodyVel then
            if Config.Fly_AntiFall and flyPart and not inVehicle then
                pcall(function() flyPart.Velocity = Vector3.new(0, 0, 0) end)
            end
            flyBodyVel:Destroy()
            flyBodyVel = nil
        end
        if flyGyro then
            flyGyro:Destroy()
            flyGyro = nil
        end
    end
end))

-- ════════════════════════════════════════
-- NOCLIP
-- ════════════════════════════════════════

local lastNoclipState = false
table.insert(connections, RunService.Stepped:Connect(function()
    if Config.Noclip_Enabled then
        local char = LocalPlayer.Character
        if char then
            for _, part in pairs(char:GetDescendants()) do
                if part:IsA("BasePart") and part.CanCollide then
                    part.CanCollide = false
                end
            end
        end
        lastNoclipState = true
    elseif lastNoclipState then
        local char = LocalPlayer.Character
        if char then
            for _, part in pairs(char:GetDescendants()) do
                if part:IsA("BasePart") then
                    part.CanCollide = true
                end
            end
        end
        lastNoclipState = false
    end
end))

-- ════════════════════════════════════════
-- HOOKLAR
-- ════════════════════════════════════════

pcall(function()
    if hookmetamethod then
        local oldNamecall
        oldNamecall = hookmetamethod(game, "__namecall", function(self, ...)
            if not checkcaller() then
                local method = getnamecallmethod()
                if Config.Anti_Kick and self == LocalPlayer and tostring(method):lower() == "kick" then
                    return
                end
            end
            return oldNamecall(self, ...)
        end)

        local oldIndex
        oldIndex = hookmetamethod(game, "__index", function(t, k)
            if not checkcaller() and Config.Aim_Silent and Config.Aim_Enabled
                and typeof(t) == "Instance" and t:IsA("Mouse") and (k == "Hit" or k == "Target") then
                if CachedTarget and CachedTarget.Character then
                    local partName = Config.Aim_Target == "Torso" and "HumanoidRootPart" or "Head"
                    local targetPart = CachedTarget.Character:FindFirstChild(partName)
                    if targetPart then
                        return k == "Hit" and targetPart.CFrame or targetPart
                    end
                end
            end
            return oldIndex(t, k)
        end)
    end
end)

-- ════════════════════════════════════════
-- ANTI-IDLE
-- ════════════════════════════════════════

table.insert(connections, LocalPlayer.Idled:Connect(function()
    if Config.Anti_Idle then
        VirtualInputManager:SendMouseButtonEvent(0, 0, 2, true, nil, 0)
        task.wait(1)
        VirtualInputManager:SendMouseButtonEvent(0, 0, 2, false, nil, 0)
    end
end))

-- ════════════════════════════════════════
-- TUŞA BASMA GİRDİLERİ
-- ════════════════════════════════════════

table.insert(connections, UIS.InputBegan:Connect(function(input, processed)
    local isMouse = string.find(input.UserInputType.Name, "MouseButton")
    local isKey = input.UserInputType == Enum.UserInputType.Keyboard

    if bindingTarget then
        if input.UserInputState == Enum.UserInputState.Begin then
            if isKey and input.KeyCode == Enum.KeyCode.Escape then
                bindingTarget = nil
                return
            end
            if isKey and input.KeyCode ~= Enum.KeyCode.Unknown then
                Config[bindingTarget] = input.KeyCode
                if bindButtons[bindingTarget] then
                    bindButtons[bindingTarget].Text = "Bind: " .. input.KeyCode.Name
                end
                bindingTarget = nil
            elseif isMouse then
                Config[bindingTarget] = input.UserInputType
                if bindButtons[bindingTarget] then
                    bindButtons[bindingTarget].Text = "Bind: " .. input.UserInputType.Name
                end
                bindingTarget = nil
            end
        end
        return
    end

    if processed and not isMouse then return end

    if input.UserInputState == Enum.UserInputState.Begin then
        if IsBindActive(Config.Menu_Bind) then
            MainFrame.Visible = not MainFrame.Visible
        end

        if IsBindActive(Config.Unbind_Key) then
            SelfDestruct()
        end

        if IsBindActive(Config.Noclip_Bind) then
            Config.Noclip_Enabled = not Config.Noclip_Enabled
        end

        if IsBindActive(Config.Fly_Bind) then
            Config.Fly_Enabled = not Config.Fly_Enabled
        end

        if IsBindActive(Config.TP_Bind) and Config.TP_Enabled then
            local myChar = LocalPlayer.Character
            local myHrp = myChar and myChar:FindFirstChild("HumanoidRootPart")
            if myHrp then
                local rp = RaycastParams.new()
                rp.FilterDescendantsInstances = {myChar}
                rp.FilterType = Enum.RaycastFilterType.Exclude
                local ray = Workspace:Raycast(Mouse.UnitRay.Origin, Mouse.UnitRay.Direction * 1000, rp)
                myHrp.CFrame = CFrame.new((ray and ray.Position or Mouse.Hit.Position) + Vector3.new(0, 3, 0))
            end
        end
    end
end))

print("══════════════════════════════════")
print(" tex8s Rivals Premium v5.1 Active!")
print(" Ranked Detection + Score Tracker")
print(" Menu Key: RightShift")
print(" Destroy Key: End")
print(" Auto Menu: DISABLED (manual only)")
print(" Score tracking: ENABLED")
print(" Waiting for match teleport...")
print("══════════════════════════════════")
