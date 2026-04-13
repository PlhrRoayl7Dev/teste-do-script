--[[
    Sailor Piece Advanced Hub
    Versão: 1.0.0
    Desenvolvido em LuaU para Roblox
--]]

local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local TweenService = game:GetService("TweenService")
local UserInputService = game:GetService("UserInputService")
local VirtualInputManager = game:GetService("VirtualInputManager")
local HttpService = game:GetService("HttpService")
local CoreGui = game:GetService("CoreGui")

local Player = Players.LocalPlayer
local Mouse = Player:GetMouse()
local Camera = workspace.CurrentCamera

-- ================================
-- CONFIGURAÇÕES GLOBAIS
-- ================================

local Settings = {
    Farm = {
        Enabled = false,
        TargetType = "Auto",
        SelectedEnemy = "",
        AttackMode = "M1",
        Distance = 15,
        ComboDelay = 0.3,
        AutoCollect = true,
        TeleportToEnemy = false
    },
    Boss = {
        Enabled = false,
        SelectedBoss = "",
        AutoDodge = true,
        DodgeDistance = 25,
        UseSkills = true
    },
    Quest = {
        Enabled = false,
        AutoAccept = true,
        AutoComplete = true,
        CurrentQuest = nil
    },
    Player = {
        AutoStats = false,
        StatPriority = "Melee",
        WalkSpeed = 16,
        JumpPower = 50,
        InfiniteEnergy = false
    },
    Visual = {
        ESP = false,
        ESPType = "Box",
        ShowName = true,
        ShowDistance = true
    },
    Misc = {
        AntiAFK = true,
        AutoRedeem = false,
        Keybinds = {
            ToggleUI = "RightShift",
            ToggleFarm = "F"
        }
    }
}

-- ================================
-- UTILITÁRIOS E SERVIÇOS
-- ================================

local Utils = {
    SaveConfig = function()
        local data = HttpService:JSONEncode(Settings)
        writefile("SailorPiece_Config.json", data)
        Utils:Notify("Configuração salva!", "Sucesso")
    end,
    
    LoadConfig = function()
        if isfile("SailorPiece_Config.json") then
            local data = readfile("SailorPiece_Config.json")
            local loaded = HttpService:JSONDecode(data)
            for k, v in pairs(loaded) do
                if Settings[k] then
                    Settings[k] = v
                end
            end
            Utils:Notify("Configuração carregada!", "Sucesso")
        end
    end,
    
    Notify = function(text, type)
        local notif = Instance.new("ScreenGui")
        notif.Name = "Notification"
        notif.Parent = CoreGui
        
        local frame = Instance.new("Frame")
        frame.Size = UDim2.new(0, 300, 0, 60)
        frame.Position = UDim2.new(1, -310, 0, 10)
        frame.BackgroundColor3 = Color3.fromRGB(30, 30, 35)
        frame.BorderSizePixel = 0
        frame.BackgroundTransparency = 0.1
        frame.Parent = notif
        
        local corner = Instance.new("UICorner")
        corner.CornerRadius = UDim.new(0, 8)
        corner.Parent = frame
        
        local title = Instance.new("TextLabel")
        title.Size = UDim2.new(1, -20, 0, 25)
        title.Position = UDim2.new(0, 10, 0, 5)
        title.BackgroundTransparency = 1
        title.TextColor3 = Color3.fromRGB(155, 95, 235)
        title.Text = type or "Info"
        title.TextXAlignment = Enum.TextXAlignment.Left
        title.Font = Enum.Font.GothamBold
        title.TextSize = 14
        title.Parent = frame
        
        local msg = Instance.new("TextLabel")
        msg.Size = UDim2.new(1, -20, 0, 30)
        msg.Position = UDim2.new(0, 10, 0, 30)
        msg.BackgroundTransparency = 1
        msg.TextColor3 = Color3.fromRGB(200, 200, 200)
        msg.Text = text
        msg.TextXAlignment = Enum.TextXAlignment.Left
        msg.Font = Enum.Font.Gotham
        msg.TextSize = 12
        msg.Parent = frame
        
        local tween = TweenService:Create(frame, TweenInfo.new(0.5), {Position = UDim2.new(1, -310, 0, 10)})
        tween:Play()
        
        task.wait(3)
        
        local fadeOut = TweenService:Create(frame, TweenInfo.new(0.5), {BackgroundTransparency = 1})
        fadeOut:Play()
        fadeOut.Completed:Wait()
        notif:Destroy()
    end,
    
    GetNearestEnemy = function()
        local nearest = nil
        local shortestDist = math.huge
        
        for _, v in pairs(workspace:GetDescendants()) do
            if v:IsA("Model") and v:FindFirstChild("Humanoid") and v:FindFirstChild("HumanoidRootPart") then
                if v.Name:lower():find("enemy") or v.Name:lower():find("mob") or v.Name:lower():find("pirate") then
                    local hrp = v.HumanoidRootPart
                    local dist = (hrp.Position - Player.Character.HumanoidRootPart.Position).Magnitude
                    
                    if dist < shortestDist and v.Humanoid.Health > 0 then
                        shortestDist = dist
                        nearest = v
                    end
                end
            end
        end
        
        return nearest, shortestDist
    end,
    
    GetAvailableEnemies = function()
        local enemies = {}
        for _, v in pairs(workspace:GetDescendants()) do
            if v:IsA("Model") and v:FindFirstChild("Humanoid") then
                if v.Name:lower():find("enemy") or v.Name:lower():find("mob") or v.Name:lower():find("pirate") or v.Name:lower():find("bandit") then
                    table.insert(enemies, v.Name)
                end
            end
        end
        local unique = {}
        for _, v in ipairs(enemies) do
            if not table.find(unique, v) then
                table.insert(unique, v)
            end
        end
        return unique
    end,
    
    GetAvailableBosses = function()
        local bosses = {}
        for _, v in pairs(workspace:GetDescendants()) do
            if v:IsA("Model") and v:FindFirstChild("Humanoid") then
                if v.Name:lower():find("boss") or v.Name:lower():find("king") or v.Name:lower():find("dragon") then
                    table.insert(bosses, v.Name)
                end
            end
        end
        return bosses
    end,
    
    GetAvailableTeleports = function()
        local teleports = {
            "Marine Base", "Shells Town", "Orange Town", "Syrup Village",
            "Baratie", "Arlong Park", "Loguetown", "Reverse Mountain",
            "Whisky Peak", "Little Garden", "Drum Island", "Alabasta"
        }
        return teleports
    end
}

-- ================================
-- SISTEMA DE AUTOFARM
-- ================================

local AutoFarmSystem = {
    CurrentTarget = nil,
    IsAttacking = false,
    
    Start = function()
        AutoFarmSystem.FarmLoop()
    end,
    
    Stop = function()
        AutoFarmSystem.CurrentTarget = nil
        AutoFarmSystem.IsAttacking = false
    end,
    
    FarmLoop = function()
        while Settings.Farm.Enabled and task.wait(0.1) do
            if not Player.Character or not Player.Character:FindFirstChild("HumanoidRootPart") then
                task.wait(1)
                continue
            end
            
            -- Encontrar alvo
            local target = nil
            if Settings.Farm.TargetType == "Auto" then
                target, _ = Utils.GetNearestEnemy()
            else
                for _, v in pairs(workspace:GetDescendants()) do
                    if v:IsA("Model") and v.Name == Settings.Farm.SelectedEnemy and v:FindFirstChild("Humanoid") then
                        target = v
                        break
                    end
                end
            end
            
            if not target then
                task.wait(0.5)
                continue
            end
            
            AutoFarmSystem.CurrentTarget = target
            
            local hrp = Player.Character.HumanoidRootPart
            local targetHrp = target:FindFirstChild("HumanoidRootPart")
            local humanoid = target:FindFirstChild("Humanoid")
            
            if not targetHrp or not humanoid or humanoid.Health <= 0 then
                continue
            end
            
            local distance = (targetHrp.Position - hrp.Position).Magnitude
            
            -- Movimentação
            if distance > Settings.Farm.Distance then
                if Settings.Farm.TeleportToEnemy then
                    hrp.CFrame = targetHrp.CFrame * CFrame.new(0, 0, 5)
                else
                    local tween = TweenService:Create(hrp, TweenInfo.new(0.5), {CFrame = targetHrp.CFrame * CFrame.new(0, 0, 5)})
                    tween:Play()
                    tween.Completed:Wait()
                end
            end
            
            -- Ataque
            if distance <= Settings.Farm.Distance and not AutoFarmSystem.IsAttacking then
                AutoFarmSystem.IsAttacking = true
                
                if Settings.Farm.AttackMode == "M1" then
                    VirtualInputManager:SendMouseButtonEvent(Enum.UserInputType.MouseButton1, 0, 0, true, game, 0)
                    task.wait(0.05)
                    VirtualInputManager:SendMouseButtonEvent(Enum.UserInputType.MouseButton1, 0, 0, false, game, 0)
                elseif Settings.Farm.AttackMode == "Skills" then
                    -- Ativar skills Z, X, C
                    for _, key in ipairs({"Z", "X", "C"}) do
                        VirtualInputManager:SendKeyEvent(true, key, false, game)
                        task.wait(0.1)
                        VirtualInputManager:SendKeyEvent(false, key, false, game)
                        task.wait(Settings.Farm.ComboDelay)
                    end
                end
                
                task.wait(0.5)
                AutoFarmSystem.IsAttacking = false
            end
            
            -- Auto coletar drops
            if Settings.Farm.AutoCollect then
                for _, v in pairs(workspace:GetDescendants()) do
                    if v:IsA("Model") and v:FindFirstChild("TouchInterest") and (v.Name:lower():find("fruit") or v.Name:lower():find("drop") or v.Name:lower():find("money")) then
                        if v:FindFirstChild("Handle") then
                            hrp.CFrame = v.Handle.CFrame
                        end
                    end
                end
            end
        end
    end
}

-- ================================
-- SISTEMA DE BOSSES
-- ================================

local BossSystem = {
    CurrentBoss = nil,
    
    Start = function()
        BossSystem.BossLoop()
    end,
    
    Stop = function()
        BossSystem.CurrentBoss = nil
    end,
    
    BossLoop = function()
        while Settings.Boss.Enabled and task.wait(0.1) do
            local boss = nil
            for _, v in pairs(workspace:GetDescendants()) do
                if v:IsA("Model") and v:FindFirstChild("Humanoid") and v:FindFirstChild("HumanoidRootPart") then
                    if v.Name:lower():find("boss") and v.Name == Settings.Boss.SelectedBoss then
                        boss = v
                        break
                    end
                end
            end
            
            if not boss then
                Utils:Notify("Boss não encontrado! Procurando em outro servidor...", "Info")
                task.wait(2)
                -- Server hop logic
                continue
            end
            
            BossSystem.CurrentBoss = boss
            
            local hrp = Player.Character.HumanoidRootPart
            local bossHrp = boss.HumanoidRootPart
            local distance = (bossHrp.Position - hrp.Position).Magnitude
            
            -- Teleport para o boss
            if distance > 50 then
                hrp.CFrame = bossHrp.CFrame * CFrame.new(0, 0, 15)
                task.wait(1)
            end
            
            -- Auto dodge
            if Settings.Boss.AutoDodge and distance < 10 then
                local dodgePos = hrp.CFrame * CFrame.new(10, 0, 0)
                local tween = TweenService:Create(hrp, TweenInfo.new(0.3), {CFrame = dodgePos})
                tween:Play()
                tween.Completed:Wait()
            end
            
            -- Ataque
            if distance <= 20 then
                VirtualInputManager:SendMouseButtonEvent(Enum.UserInputType.MouseButton1, 0, 0, true, game, 0)
                task.wait(0.05)
                VirtualInputManager:SendMouseButtonEvent(Enum.UserInputType.MouseButton1, 0, 0, false, game, 0)
                
                if Settings.Boss.UseSkills then
                    for _, key in ipairs({"Z", "X", "C", "V"}) do
                        VirtualInputManager:SendKeyEvent(true, key, false, game)
                        task.wait(0.2)
                        VirtualInputManager:SendKeyEvent(false, key, false, game)
                        task.wait(0.5)
                    end
                end
            end
        end
    end
}

-- ================================
-- SISTEMA DE QUESTS
-- ================================

local QuestSystem = {
    CurrentQuest = nil,
    
    Start = function()
        QuestSystem.QuestLoop()
    end,
    
    Stop = function()
        QuestSystem.CurrentQuest = nil
    end,
    
    QuestLoop = function()
        while Settings.Quest.Enabled and task.wait(1) do
            -- Procurar NPC de missão
            for _, v in pairs(workspace:GetDescendants()) do
                if v:IsA("Model") and v:FindFirstChild("Humanoid") and v.Name:lower():find("npc") then
                    local hrp = Player.Character.HumanoidRootPart
                    local npcPos = v:FindFirstChild("HumanoidRootPart")
                    
                    if npcPos then
                        local distance = (npcPos.Position - hrp.Position).Magnitude
                        
                        if distance < 10 then
                            -- Interagir com NPC
                            VirtualInputManager:SendKeyEvent(true, "E", false, game)
                            task.wait(0.1)
                            VirtualInputManager:SendKeyEvent(false, "E", false, game)
                            task.wait(1)
                        else
                            -- Teleport para NPC
                            hrp.CFrame = npcPos.CFrame * CFrame.new(0, 0, 3)
                        end
                    end
                end
            end
        end
    end
}

-- ================================
-- SISTEMA DO PLAYER
-- ================================

local PlayerSystem = {
    Start = function()
        PlayerSystem.PlayerLoop()
    end,
    
    PlayerLoop = function()
        while task.wait(1) do
            if Settings.Player.WalkSpeed ~= 16 and Player.Character and Player.Character:FindFirstChild("Humanoid") then
                Player.Character.Humanoid.WalkSpeed = Settings.Player.WalkSpeed
            end
            
            if Settings.Player.JumpPower ~= 50 and Player.Character and Player.Character:FindFirstChild("Humanoid") then
                Player.Character.Humanoid.JumpPower = Settings.Player.JumpPower
            end
        end
    end
}

-- ================================
-- SISTEMA VISUAL (ESP)
-- ================================

local ESPSystem = {
    ESPObjects = {},
    
    CreateESP = function(instance)
        local highlight = Instance.new("Highlight")
        highlight.Parent = instance
        highlight.FillTransparency = 0.7
        highlight.OutlineTransparency = 0.3
        highlight.FillColor = Color3.fromRGB(155, 95, 235)
        
        local billboard = Instance.new("BillboardGui")
        billboard.Parent = instance
        billboard.Size = UDim2.new(0, 200, 0, 50)
        billboard.StudsOffset = Vector3.new(0, 3, 0)
        
        local text = Instance.new("TextLabel")
        text.Parent = billboard
        text.Size = UDim2.new(1, 0, 1, 0)
        text.BackgroundTransparency = 1
        text.TextColor3 = Color3.fromRGB(255, 255, 255)
        text.TextStrokeTransparency = 0.3
        text.Font = Enum.Font.GothamBold
        text.TextSize = 14
        text.Text = instance.Name
        
        table.insert(ESPSystem.ESPObjects, {instance, highlight, billboard, text})
    end,
    
    Start = function()
        ESPSystem.ESPLoop()
    end,
    
    Stop = function()
        for _, v in pairs(ESPSystem.ESPObjects) do
            pcall(function()
                v[2]:Destroy()
                v[3]:Destroy()
            end)
        end
        ESPSystem.ESPObjects = {}
    end,
    
    ESPLoop = function()
        while Settings.Visual.ESP and task.wait(0.5) do
            for _, v in pairs(workspace:GetDescendants()) do
                if v:IsA("Model") and v:FindFirstChild("Humanoid") then
                    local hasESP = false
                    for _, esp in pairs(ESPSystem.ESPObjects) do
                        if esp[1] == v then
                            hasESP = true
                            if Settings.Visual.ShowDistance then
                                local dist = (v.HumanoidRootPart.Position - Player.Character.HumanoidRootPart.Position).Magnitude
                                esp[4].Text = v.Name .. " [" .. math.floor(dist) .. "m]"
                            else
                                esp[4].Text = v.Name
                            end
                            break
                        end
                    end
                    
                    if not hasESP then
                        ESPSystem.CreateESP(v)
                    end
                end
            end
        end
    end
}

-- ================================
-- INTERFACE GRÁFICA (UI)
-- ================================

local UI = {
    MainGui = nil,
    Dragging = false,
    DragStart = nil,
    
    Create = function()
        local screenGui = Instance.new("ScreenGui")
        screenGui.Name = "SailorPieceHub"
        screenGui.ZIndexBehavior = Enum.ZIndexBehavior.Sibling
        screenGui.Parent = CoreGui
        
        -- Main Frame
        local mainFrame = Instance.new("Frame")
        mainFrame.Size = UDim2.new(0, 500, 0, 600)
        mainFrame.Position = UDim2.new(0.5, -250, 0.5, -300)
        mainFrame.BackgroundColor3 = Color3.fromRGB(25, 25, 30)
        mainFrame.BorderSizePixel = 0
        mainFrame.ClipsDescendants = true
        mainFrame.Parent = screenGui
        
        local mainCorner = Instance.new("UICorner")
        mainCorner.CornerRadius = UDim.new(0, 12)
        mainCorner.Parent = mainFrame
        
        local shadow = Instance.new("UIStroke")
        shadow.Color = Color3.fromRGB(0, 0, 0)
        shadow.Thickness = 2
        shadow.Transparency = 0.5
        shadow.Parent = mainFrame
        
        -- Title Bar
        local titleBar = Instance.new("Frame")
        titleBar.Size = UDim2.new(1, 0, 0, 40)
        titleBar.BackgroundColor3 = Color3.fromRGB(35, 35, 40)
        titleBar.BorderSizePixel = 0
        titleBar.Parent = mainFrame
        
        local titleCorner = Instance.new("UICorner")
        titleCorner.CornerRadius = UDim.new(0, 12)
        titleCorner.Parent = titleBar
        
        local titleText = Instance.new("TextLabel")
        titleText.Size = UDim2.new(1, -40, 1, 0)
        titleText.Position = UDim2.new(0, 20, 0, 0)
        titleText.BackgroundTransparency = 1
        titleText.Text = "SAILOR PIECE HUB"
        titleText.TextColor3 = Color3.fromRGB(155, 95, 235)
        titleText.TextXAlignment = Enum.TextXAlignment.Left
        titleText.Font = Enum.Font.GothamBold
        titleText.TextSize = 18
        titleText.Parent = titleBar
        
        -- Close Button
        local closeBtn = Instance.new("TextButton")
        closeBtn.Size = UDim2.new(0, 30, 1, 0)
        closeBtn.Position = UDim2.new(1, -30, 0, 0)
        closeBtn.BackgroundTransparency = 1
        closeBtn.Text = "✕"
        closeBtn.TextColor3 = Color3.fromRGB(255, 100, 100)
        closeBtn.Font = Enum.Font.GothamBold
        closeBtn.TextSize = 18
        closeBtn.Parent = titleBar
        
        closeBtn.MouseButton1Click:Connect(function()
            screenGui:Destroy()
        end)
        
        -- Tab Container
        local tabContainer = Instance.new("Frame")
        tabContainer.Size = UDim2.new(1, 0, 0, 45)
        tabContainer.Position = UDim2.new(0, 0, 0, 40)
        tabContainer.BackgroundTransparency = 1
        tabContainer.Parent = mainFrame
        
        local tabs = {"AutoFarm", "Bosses", "Quests", "Player", "Teleport", "Visual", "Misc"}
        local tabButtons = {}
        
        for i, tabName in ipairs(tabs) do
            local btn = Instance.new("TextButton")
            btn.Size = UDim2.new(0, 70, 1, 0)
            btn.Position = UDim2.new(0, (i-1) * 72, 0, 0)
            btn.BackgroundTransparency = 1
            btn.Text = tabName
            btn.TextColor3 = Color3.fromRGB(200, 200, 200)
            btn.Font = Enum.Font.Gotham
            btn.TextSize = 12
            btn.Parent = tabContainer
            
            tabButtons[tabName] = btn
        end
        
        -- Content Container
        local contentContainer = Instance.new("Frame")
        contentContainer.Size = UDim2.new(1, -20, 1, -95)
        contentContainer.Position = UDim2.new(0, 10, 0, 90)
        contentContainer.BackgroundTransparency = 1
        contentContainer.Parent = mainFrame
        
        -- Criar abas
        local autoFarmTab = UI.CreateAutoFarmTab(contentContainer)
        local bossesTab = UI.CreateBossesTab(contentContainer)
        local questsTab = UI.CreateQuestsTab(contentContainer)
        local playerTab = UI.CreatePlayerTab(contentContainer)
        local teleportTab = UI.CreateTeleportTab(contentContainer)
        local visualTab = UI.CreateVisualTab(contentContainer)
        local miscTab = UI.CreateMiscTab(contentContainer)
        
        -- Esconder todas as abas inicialmente
        for _, tab in pairs({autoFarmTab, bossesTab, questsTab, playerTab, teleportTab, visualTab, miscTab}) do
            tab.Visible = false
        end
        autoFarmTab.Visible = true
        
        -- Tab switching
        for tabName, btn in pairs(tabButtons) do
            btn.MouseButton1Click:Connect(function()
                for _, tab in pairs({autoFarmTab, bossesTab, questsTab, playerTab, teleportTab, visualTab, miscTab}) do
                    tab.Visible = false
                end
                
                if tabName == "AutoFarm" then autoFarmTab.Visible = true
                elseif tabName == "Bosses" then bossesTab.Visible = true
                elseif tabName == "Quests" then questsTab.Visible = true
                elseif tabName == "Player" then playerTab.Visible = true
                elseif tabName == "Teleport" then teleportTab.Visible = true
                elseif tabName == "Visual" then visualTab.Visible = true
                elseif tabName == "Misc" then miscTab.Visible = true
                end
                
                for _, b in pairs(tabButtons) do
                    b.TextColor3 = Color3.fromRGB(200, 200, 200)
                end
                btn.TextColor3 = Color3.fromRGB(155, 95, 235)
            end)
        end
        
        UI.MainGui = screenGui
        
        -- Draggable
        local dragToggle = false
        local dragInput
        local dragStart
        
        titleBar.InputBegan:Connect(function(input)
            if input.UserInputType == Enum.UserInputType.MouseButton1 then
                dragToggle = true
                dragStart = input.Position
            end
        end)
        
        titleBar.InputEnded:Connect(function(input)
            if input.UserInputType == Enum.UserInputType.MouseButton1 then
                dragToggle = false
            end
        end)
        
        UserInputService.InputChanged:Connect(function(input)
            if input.UserInputType == Enum.UserInputType.MouseMovement and dragToggle then
                local delta = input.Position - dragStart
                mainFrame.Position = mainFrame.Position + UDim2.new(0, delta.X, 0, delta.Y)
                dragStart = input.Position
            end
        end)
    end,
    
    CreateToggle = function(parent, text, settingPath)
        local frame = Instance.new("Frame")
        frame.Size = UDim2.new(1, 0, 0, 35)
        frame.BackgroundTransparency = 1
        frame.Parent = parent
        
        local label = Instance.new("TextLabel")
        label.Size = UDim2.new(0.7, 0, 1, 0)
        label.Position = UDim2.new(0, 10, 0, 0)
        label.BackgroundTransparency = 1
        label.Text = text
        label.TextColor3 = Color3.fromRGB(220, 220, 220)
        label.TextXAlignment = Enum.TextXAlignment.Left
        label.Font = Enum.Font.Gotham
        label.TextSize = 13
        label.Parent = frame
        
        local toggleBtn = Instance.new("TextButton")
        toggleBtn.Size = UDim2.new(0, 50, 0, 25)
        toggleBtn.Position = UDim2.new(1, -60, 0.5, -12.5)
        toggleBtn.BackgroundColor3 = Color3.fromRGB(60, 60, 65)
        toggleBtn.BorderSizePixel = 0
        toggleBtn.Text = ""
        toggleBtn.Parent = frame
        
        local toggleCorner = Instance.new("UICorner")
        toggleCorner.CornerRadius = UDim.new(1, 0)
        toggleCorner.Parent = toggleBtn
        
        local toggleIndicator = Instance.new("Frame")
        toggleIndicator.Size = UDim2.new(0, 20, 0, 20)
        toggleIndicator.Position = UDim2.new(0, 3, 0.5, -10)
        toggleIndicator.BackgroundColor3 = Color3.fromRGB(200, 200, 200)
        toggleIndicator.BorderSizePixel = 0
        toggleIndicator.Parent = toggleBtn
        
        local indicatorCorner = Instance.new("UICorner")
        indicatorCorner.CornerRadius = UDim.new(1, 0)
        indicatorCorner.Parent = toggleIndicator
        
        local function updateToggle()
            local value = Settings
            for part in settingPath:gmatch("[^.]+") do
                value = value[part]
                if not value then break end
            end
            
            if value then
                toggleBtn.BackgroundColor3 = Color3.fromRGB(100, 70, 150)
                toggleIndicator.Position = UDim2.new(1, -23, 0.5, -10)
                toggleIndicator.BackgroundColor3 = Color3.fromRGB(255, 255, 255)
            else
                toggleBtn.BackgroundColor3 = Color3.fromRGB(60, 60, 65)
                toggleIndicator.Position = UDim2.new(0, 3, 0.5, -10)
                toggleIndicator.BackgroundColor3 = Color3.fromRGB(200, 200, 200)
            end
        end
        
        toggleBtn.MouseButton1Click:Connect(function()
            local parts = {}
            for part in settingPath:gmatch("[^.]+") do
                table.insert(parts, part)
            end
            
            local current = Settings
            for i = 1, #parts - 1 do
                current = current[parts[i]]
            end
            
            current[parts[#parts]] = not current[parts[#parts]]
            updateToggle()
        end)
        
        updateToggle()
        
        return frame
    end,
    
    CreateAutoFarmTab = function(parent)
        local tab = Instance.new("ScrollingFrame")
        tab.Size = UDim2.new(1, 0, 1, 0)
        tab.BackgroundTransparency = 1
        tab.ScrollBarThickness = 6
        tab.CanvasSize = UDim2.new(0, 0, 0, 400)
        tab.Parent = parent
        
        UI.CreateToggle(tab, "Auto Farm", "Farm.Enabled")
        
        local targetTypeFrame = UI.CreateDropdown(tab, "Target Type", {"Auto", "Manual"}, function(value)
            Settings.Farm.TargetType = value
        end)
        targetTypeFrame.Position = UDim2.new(0, 0, 0, 45)
        
        local enemyDropdown = UI.CreateDropdown(tab, "Select Enemy", Utils.GetAvailableEnemies(), function(value)
            Settings.Farm.SelectedEnemy = value
        end)
        enemyDropdown.Position = UDim2.new(0, 0, 0, 95)
        
        local attackModeFrame = UI.CreateDropdown(tab, "Attack Mode", {"M1", "Skills", "Both"}, function(value)
            Settings.Farm.AttackMode = value
        end)
        attackModeFrame.Position = UDim2.new(0, 0, 0, 145)
        
        UI.CreateSlider(tab, "Farm Distance", 10, 50, Settings.Farm.Distance, function(value)
            Settings.Farm.Distance = value
        end)
        
        UI.CreateToggle(tab, "Auto Collect Drops", "Farm.AutoCollect")
        UI.CreateToggle(tab, "Teleport to Enemy", "Farm.TeleportToEnemy")
        
        return tab
    end,
    
    CreateBossesTab = function(parent)
        local tab = Instance.new("ScrollingFrame")
        tab.Size = UDim2.new(1, 0, 1, 0)
        tab.BackgroundTransparency = 1
        tab.ScrollBarThickness = 6
        tab.CanvasSize = UDim2.new(0, 0, 0, 300)
        tab.Parent = parent
        
        UI.CreateToggle(tab, "Auto Boss Farm", "Boss.Enabled")
        
        local bossDropdown = UI.CreateDropdown(tab, "Select Boss", Utils.GetAvailableBosses(), function(value)
            Settings.Boss.SelectedBoss = value
        end)
        bossDropdown.Position = UDim2.new(0, 0, 0, 45)
        
        UI.CreateToggle(tab, "Auto Dodge", "Boss.AutoDodge")
        UI.CreateToggle(tab, "Use Skills", "Boss.UseSkills")
        UI.CreateSlider(tab, "Dodge Distance", 15, 40, Settings.Boss.DodgeDistance, function(value)
            Settings.Boss.DodgeDistance = value
        end)
        
        return tab
    end,
    
    CreateQuestsTab = function(parent)
        local tab = Instance.new("ScrollingFrame")
        tab.Size = UDim2.new(1, 0, 1, 0)
        tab.BackgroundTransparency = 1
        tab.ScrollBarThickness = 6
        tab.CanvasSize = UDim2.new(0, 0, 0, 150)
        tab.Parent = parent
        
        UI.CreateToggle(tab, "Auto Quest", "Quest.Enabled")
        UI.CreateToggle(tab, "Auto Accept Quests", "Quest.AutoAccept")
        UI.CreateToggle(tab, "Auto Complete Quests", "Quest.AutoComplete")
        
        return tab
    end,
    
    CreatePlayerTab = function(parent)
        local tab = Instance.new("ScrollingFrame")
        tab.Size = UDim2.new(1, 0, 1, 0)
        tab.BackgroundTransparency = 1
        tab.ScrollBarThickness = 6
        tab.CanvasSize = UDim2.new(0, 0, 0, 350)
        tab.Parent = parent
        
        UI.CreateToggle(tab, "Auto Stats", "Player.AutoStats")
        
        local statDropdown = UI.CreateDropdown(tab, "Stat Priority", {"Melee", "Defense", "Sword", "Fruit"}, function(value)
            Settings.Player.StatPriority = value
        end)
        statDropdown.Position = UDim2.new(0, 0, 0, 45)
        
        UI.CreateSlider(tab, "Walk Speed", 16, 100, Settings.Player.WalkSpeed, function(value)
            Settings.Player.WalkSpeed = value
        end)
        
        UI.CreateSlider(tab, "Jump Power", 50, 200, Settings.Player.JumpPower, function(value)
            Settings.Player.JumpPower = value
        end)
        
        UI.CreateToggle(tab, "Infinite Energy", "Player.InfiniteEnergy")
        
        return tab
    end,
    
    CreateTeleportTab = function(parent)
        local tab = Instance.new("ScrollingFrame")
        tab.Size = UDim2.new(1, 0, 1, 0)
        tab.BackgroundTransparency = 1
        tab.ScrollBarThickness = 6
        tab.CanvasSize = UDim2.new(0, 0, 0, 500)
        tab.Parent = parent
        
        local teleports = Utils.GetAvailableTeleports()
        
        for i, loc in ipairs(teleports) do
            local btn = Instance.new("TextButton")
            btn.Size = UDim2.new(1, -20, 0, 35)
            btn.Position = UDim2.new(0, 10, 0, (i-1) * 40)
            btn.BackgroundColor3 = Color3.fromRGB(40, 40, 45)
            btn.BorderSizePixel = 0
            btn.Text = "Teleport to " .. loc
            btn.TextColor3 = Color3.fromRGB(220, 220, 220)
            btn.Font = Enum.Font.Gotham
            btn.TextSize = 13
            btn.Parent = tab
            
            local btnCorner = Instance.new("UICorner")
            btnCorner.CornerRadius = UDim.new(0, 6)
            btnCorner.Parent = btn
            
            btn.MouseButton1Click:Connect(function()
                Utils:Notify("Teleporting to " .. loc, "Info")
                -- Teleport logic here
            end)
        end
        
        tab.CanvasSize = UDim2.new(0, 0, 0, #teleports * 40 + 20)
        
        return tab
    end,
    
    CreateVisualTab
