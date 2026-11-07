local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local UserInputService = game:GetService("UserInputService")
local ContentProvider = game:GetService("ContentProvider")
local Camera = workspace.CurrentCamera
local LocalPlayer = Players.LocalPlayer

-- ================= CONFIGURAÇÃO =================
local AIM_ENABLED = true
local AIM_KEY = Enum.UserInputType.MouseButton2
local TARGET_PART = "Head"

local TEAM_CHECK_ENABLED = true

local MAX_DISTANCE = 1000
local FOV_RADIUS = 150
local ESP_PLAYER_ENABLED = true
local ESP_NPC_ENABLED = true
local FOV_RGB = true
local ESP_RGB = true

local AIM_PREDICTION_ENABLED = true
local PREDICTION_TIME = 0.10
local AGGRESSIVENESS = 1.00
local SMOOTHING = 0.0

local NPC_CACHE = {}
local PLAYER_ESP = {}
local NPC_ESP = {}
local CURRENT_TARGET = nil
local CURRENT_PART = nil

local SCAN_INTERVAL = 0.06
local ESP_UPDATE_INTERVAL = 0.12
local lastScan = 0
local lastESP = 0

local PREV_HISTORY = {}
local MAX_HISTORY_SAMPLES = 6

-- ================= FOV DRAWING =================
local Drawing = Drawing
local FOVCircle = Drawing.new("Circle")
FOVCircle.Radius = FOV_RADIUS
FOVCircle.Thickness = 2
FOVCircle.NumSides = 100
FOVCircle.Filled = false
FOVCircle.Color = Color3.fromRGB(0, 255, 255)
FOVCircle.Visible = true
local hue = 0

-- ================= UTILITÁRIOS =================
local math_clamp = math.clamp
local tickf = tick

local function WorldToScreen(pos)
    local p, onScreen = Camera:WorldToViewportPoint(pos)
    return Vector2.new(p.X, p.Y), onScreen
end

local function GetTeamIdentifier(model)
    if not model then
        return nil
    end

    local player = Players:GetPlayerFromCharacter(model)
    if player then
        if player.Team then
            return player.Team
        end
        if player.TeamColor then
            return player.TeamColor
        end
    end

    local teamObj = model:FindFirstChild("Team")
    if teamObj and teamObj:IsA("Team") then
        return teamObj
    end

    local teamColor = model:FindFirstChild("TeamColor")
    if teamColor and teamColor:IsA("BrickColor") then
        return teamColor
    end

    local teamName = model:FindFirstChild("TeamName")
    if teamName and teamName:IsA("StringValue") then
        return teamName.Value
    end

    return nil
end

local function IsTeammate(targetModel)
    if not TEAM_CHECK_ENABLED then
        return false
    end

    local localCharacter = LocalPlayer and LocalPlayer.Character
    if not localCharacter or not targetModel then
        return false
    end

    local localTeam = GetTeamIdentifier(localCharacter)
    local targetTeam = GetTeamIdentifier(targetModel)

    if not localTeam or not targetTeam then
        return false
    end

    if typeof(localTeam) == "BrickColor" and typeof(targetTeam) == "BrickColor" then
        return localTeam == targetTeam
    end

    if typeof(localTeam) == "Instance" and typeof(targetTeam) == "Instance" then
        return localTeam == targetTeam
    end

    return tostring(localTeam) == tostring(targetTeam)
end

local function IsNPC(model)
    return model and model:IsA("Model") and model:FindFirstChild("Humanoid") and model:FindFirstChild("HumanoidRootPart") and not Players:GetPlayerFromCharacter(model)
end

local function AddHistorySample(part, pos, t)
    if not part then return end
    local hist = PREV_HISTORY[part]
    if not hist then
        hist = { samples = {} }
        PREV_HISTORY[part] = hist
    end
    local s = hist.samples
    s[#s + 1] = { pos = pos, t = t }
    if #s > MAX_HISTORY_SAMPLES then
        table.remove(s, 1)
    end
end

local function EstimateVelAcc(part)
    local hist = PREV_HISTORY[part]
    if not hist or #hist.samples < 2 then
        local ok, v = pcall(function()
            return part and part.Velocity
        end)
        if ok and typeof(v) == "Vector3" then
            return v, Vector3.new(0, 0, 0)
        end
        return Vector3.new(0, 0, 0), Vector3.new(0, 0, 0)
    end

    local velList = {}
    for i = 2, #hist.samples do
        local a = hist.samples[i - 1]
        local b = hist.samples[i]
        local dt = b.t - a.t
        if dt > 0 then
            velList[#velList + 1] = (b.pos - a.pos) / dt
        end
    end

    if #velList == 0 then
        return Vector3.new(0, 0, 0), Vector3.new(0, 0, 0)
    end

    local totalWeight, weightedVel = 0, Vector3.new(0, 0, 0)
    for i = 1, #velList do
        local weight = i / #velList
        weightedVel = weightedVel + velList[i] * weight
        totalWeight = totalWeight + weight
    end
    local avgVel = weightedVel / math.max(1, totalWeight)

    local acc = Vector3.new(0, 0, 0)
    if #velList >= 2 then
        local vprev = velList[#velList - 1]
        local vlast = velList[#velList]
        local dt = hist.samples[#hist.samples].t - hist.samples[#hist.samples - 1].t
        if dt > 0 then
            acc = (vlast - vprev) / dt
        end
    end

    return avgVel, acc
end

-- ================= ESP GUI =================
local BACKGROUND_ASSET_ID = "rbxassetid://102215248035065"

local ScreenGui = Instance.new("ScreenGui")
ScreenGui.Name = "AimESP_UI_Pro"
ScreenGui.ResetOnSpawn = false
ScreenGui.IgnoreGuiInset = true

if LocalPlayer and LocalPlayer:FindFirstChild("PlayerGui") then
    ScreenGui.Parent = LocalPlayer:FindFirstChild("PlayerGui")
else
    ScreenGui.Parent = game:GetService("CoreGui")
end

local function CreatePlayerESPGui()
    local gui = Instance.new("BillboardGui")
    gui.Name = "ESP_PLAYER"
    gui.Size = UDim2.new(0, 200, 0, 60)
    gui.AlwaysOnTop = true
    gui.Adornee = nil
    gui.Enabled = false
    gui.Parent = ScreenGui

    local txt = Instance.new("TextLabel", gui)
    txt.Size = UDim2.new(1, 0, 1, 0)
    txt.BackgroundTransparency = 1
    txt.TextScaled = true
    txt.Font = Enum.Font.SourceSansBold
    txt.TextStrokeTransparency = 0
    txt.TextColor3 = Color3.fromRGB(0, 255, 0)
    txt.Text = ""

    return { Gui = gui, Text = txt, lastText = "" }
end

local function CreateNPCESPGui()
    local gui = Instance.new("BillboardGui")
    gui.Name = "ESP_NPC"
    gui.Size = UDim2.new(0, 180, 0, 48)
    gui.AlwaysOnTop = true
    gui.Adornee = nil
    gui.Enabled = false
    gui.Parent = ScreenGui

    local txt = Instance.new("TextLabel", gui)
    txt.Size = UDim2.new(1, 0, 1, 0)
    txt.BackgroundTransparency = 1
    txt.TextScaled = true
    txt.Font = Enum.Font.SourceSans
    txt.TextStrokeTransparency = 0
    txt.TextColor3 = Color3.fromRGB(200, 200, 200)
    txt.Text = ""

    return { Gui = gui, Text = txt, lastText = "" }
end

local function RegisterPlayer(player)
    if not player or player == LocalPlayer then return end
    if PLAYER_ESP[player] then return end
    PLAYER_ESP[player] = CreatePlayerESPGui()
end

for _, p in ipairs(Players:GetPlayers()) do
    RegisterPlayer(p)
end

Players.PlayerAdded:Connect(RegisterPlayer)
Players.PlayerRemoving:Connect(function(p)
    if PLAYER_ESP[p] then
        PLAYER_ESP[p].Gui:Destroy()
        PLAYER_ESP[p] = nil
    end
end)

local function RegisterNPC(npc)
    if not npc or NPC_ESP[npc] then return end
    NPC_ESP[npc] = CreateNPCESPGui()
end

-- ================= SCAN =================
local function CleanHistory()
    for part, h in pairs(PREV_HISTORY) do
        if not part or not part.Parent then
            PREV_HISTORY[part] = nil
        end
    end
end

local function ScanAndUpdate()
    local now = tickf()
    CleanHistory()

    local localChar = LocalPlayer and LocalPlayer.Character
    local localRoot = localChar and localChar:FindFirstChild("HumanoidRootPart")
    if not localRoot or not Camera then
        CURRENT_TARGET = nil
        CURRENT_PART = nil
        return
    end

    local bestTarget, bestPart
    local bestDist = math.huge
    local vpSize = Camera.ViewportSize
    local center = Vector2.new(vpSize.X / 2, vpSize.Y / 2)

    -- Players
    for _, plr in ipairs(Players:GetPlayers()) do
        if plr ~= LocalPlayer and plr.Character then
            local char = plr.Character
            local hum = char:FindFirstChild("Humanoid")
            local root = char:FindFirstChild("HumanoidRootPart")
            local part = char:FindFirstChild(TARGET_PART)
            if hum and root and part and hum.Health > 0 then
                if TEAM_CHECK_ENABLED and IsTeammate(char) then
                    -- ignora aliados
                else
                    local d = (root.Position - localRoot.Position).Magnitude
                    if d <= MAX_DISTANCE then
                        local sp, on = WorldToScreen(part.Position)
                        if on and (sp - center).Magnitude <= FOV_RADIUS then
                            if d < bestDist then
                                bestDist = d
                                bestTarget = char
                                bestPart = part
                            end
                        end
                    end
                    AddHistorySample(root, root.Position, now)
                end
            end
        end
    end

    -- NPCs
    for _, npc in ipairs(NPC_CACHE) do
        if npc and npc.Parent then
            local hum = npc:FindFirstChild("Humanoid")
            local root = npc:FindFirstChild("HumanoidRootPart")
            local part = npc:FindFirstChild(TARGET_PART) or root
            if hum and root and part and hum.Health > 0 then
                if TEAM_CHECK_ENABLED and IsTeammate(npc) then
                    -- ignora aliados
                else
                    local d = (root.Position - localRoot.Position).Magnitude
                    if d <= MAX_DISTANCE then
                        local sp, on = WorldToScreen(part.Position)
                        if on and (sp - center).Magnitude <= FOV_RADIUS then
                            if d < bestDist then
                                bestDist = d
                                bestTarget = npc
                                bestPart = part
                            end
                        end
                    end
                    AddHistorySample(root, root.Position, now)
                    RegisterNPC(npc)
                end
            end
        end
    end

    -- ESP players
    for player, data in pairs(PLAYER_ESP) do
        local char = player and player.Character
        if data and char and char:FindFirstChild("Head") and char:FindFirstChild("HumanoidRootPart") then
            local hum = char:FindFirstChild("Humanoid")
            local dist = (char.HumanoidRootPart.Position - localRoot.Position).Magnitude

            if TEAM_CHECK_ENABLED and IsTeammate(char) then
                if data.Gui.Enabled then
                    data.Gui.Enabled = false
                    data.lastText = ""
                    data.Text.Text = ""
                    data.Gui.Adornee = nil
                end
            elseif hum and hum.Health > 0 and dist <= MAX_DISTANCE and ESP_PLAYER_ENABLED then
                data.Gui.Adornee = char.Head
                data.Gui.Enabled = true
                local newText = player.Name .. "\n" .. math.floor(dist) .. " m\nHP: " .. math.floor(hum.Health) .. "/" .. math.floor(hum.MaxHealth or 100)
                if newText ~= data.lastText then
                    data.Text.Text = newText
                    data.lastText = newText
                end
            else
                if data.Gui.Enabled then
                    data.Gui.Enabled = false
                    data.lastText = ""
                    data.Text.Text = ""
                    data.Gui.Adornee = nil
                end
            end
        else
            if data.Gui.Enabled then
                data.Gui.Enabled = false
                data.lastText = ""
                data.Text.Text = ""
                data.Gui.Adornee = nil
            end
        end
    end

    -- ESP NPCs
    for npc, data in pairs(NPC_ESP) do
        if npc and npc.Parent and data and data.Gui then
            local hum = npc:FindFirstChild("Humanoid")
            local root = npc:FindFirstChild("HumanoidRootPart")
            if TEAM_CHECK_ENABLED and IsTeammate(npc) then
                if data.Gui.Enabled then
                    data.Gui.Enabled = false
                    data.Text.Text = ""
                    data.lastText = ""
                    data.Gui.Adornee = nil
                end
            elseif hum and root and hum.Health > 0 and ESP_NPC_ENABLED then
                local dist = (root.Position - localRoot.Position).Magnitude
                if dist <= MAX_DISTANCE then
                    data.Gui.Adornee = root
                    data.Gui.Enabled = true
                    local newText = npc.Name .. " | " .. math.floor(dist) .. "m | " .. math.floor(hum.Health)
                    if newText ~= data.lastText then
                        data.Text.Text = newText
                        data.lastText = newText
                    end
                else
                    if data.Gui.Enabled then
                        data.Gui.Enabled = false
                        data.Text.Text = ""
                        data.lastText = ""
                        data.Gui.Adornee = nil
                    end
                end
            else
                if data.Gui.Enabled then
                    data.Gui.Enabled = false
                    data.Text.Text = ""
                    data.lastText = ""
                    data.Gui.Adornee = nil
                end
            end
        end
    end

    CURRENT_TARGET = bestTarget
    CURRENT_PART = bestPart
end

-- ================= CONFIG PANEL =================
local ConfigGui = Instance.new("ScreenGui")
ConfigGui.Name = "AimESP_Config"
ConfigGui.ResetOnSpawn = false
ConfigGui.IgnoreGuiInset = true
ConfigGui.Enabled = true
ConfigGui.Parent = ScreenGui.Parent

local ConfigFrame = Instance.new("Frame")
ConfigFrame.Name = "Main"
ConfigFrame.Size = UDim2.new(0, 280, 0, 300)
ConfigFrame.Position = UDim2.new(0, 20, 0, 20)
ConfigFrame.BackgroundColor3 = Color3.fromRGB(20, 20, 22)
ConfigFrame.BackgroundTransparency = 0.15
ConfigFrame.BorderSizePixel = 0
ConfigFrame.Parent = ConfigGui

local Corner = Instance.new("UICorner")
Corner.CornerRadius = UDim.new(0, 10)
Corner.Parent = ConfigFrame

local Title = Instance.new("TextLabel")
Title.Size = UDim2.new(1, -20, 0, 26)
Title.Position = UDim2.new(0, 10, 0, 10)
Title.BackgroundTransparency = 1
Title.Text = "Aim ESP Config"
Title.TextColor3 = Color3.fromRGB(255, 255, 255)
Title.Font = Enum.Font.SourceSansBold
Title.TextSize = 18
Title.TextXAlignment = Enum.TextXAlignment.Left
Title.Parent = ConfigFrame

local function createNumberField(labelText, defaultValue, yPos)
    local container = Instance.new("Frame")
    container.Size = UDim2.new(1, -20, 0, 42)
    container.Position = UDim2.new(0, 10, 0, yPos)
    container.BackgroundColor3 = Color3.fromRGB(35, 35, 40)
    container.BorderSizePixel = 0
    container.Parent = ConfigFrame

    local corner = Instance.new("UICorner")
    corner.CornerRadius = UDim.new(0, 8)
    corner.Parent = container

    local label = Instance.new("TextLabel")
    label.Size = UDim2.new(0.5, 0, 1, 0)
    label.Position = UDim2.new(0, 8, 0, 0)
    label.BackgroundTransparency = 1
    label.Text = labelText
    label.TextColor3 = Color3.fromRGB(220, 220, 220)
    label.Font = Enum.Font.SourceSans
    label.TextSize = 15
    label.TextXAlignment = Enum.TextXAlignment.Left
    label.Parent = container

    local box = Instance.new("TextBox")
    box.Size = UDim2.new(0.42, 0, 0.7, 0)
    box.Position = UDim2.new(0.58, 0, 0.15, 0)
    box.BackgroundColor3 = Color3.fromRGB(50, 50, 55)
    box.BorderSizePixel = 0
    box.Text = tostring(defaultValue)
    box.TextColor3 = Color3.fromRGB(255, 255, 255)
    box.TextSize = 16
    box.Font = Enum.Font.SourceSans
    box.TextXAlignment = Enum.TextXAlignment.Center
    box.Parent = container

    local boxCorner = Instance.new("UICorner")
    boxCorner.CornerRadius = UDim.new(0, 6)
    boxCorner.Parent = box

    return box
end

local espDistBox = createNumberField("ESP Distância", MAX_DISTANCE, 52)
local fovBox = createNumberField("FOV", FOV_RADIUS, 100)
local predictionBox = createNumberField("Aim Predição", PREDICTION_TIME, 148)

local function createToggleButton(label, xPos, yPos, width, height)
    local button = Instance.new("TextButton")
    button.Size = UDim2.new(width, 0, 0, height)
    button.Position = UDim2.new(0, xPos, 0, yPos)
    button.BackgroundColor3 = Color3.fromRGB(35, 35, 40)
    button.Text = label
    button.TextColor3 = Color3.fromRGB(255, 255, 255)
    button.Font = Enum.Font.SourceSansBold
    button.TextSize = 15
    button.BorderSizePixel = 0
    button.Parent = ConfigFrame

    local btnCorner = Instance.new("UICorner")
    btnCorner.CornerRadius = UDim.new(0, 8)
    btnCorner.Parent = button

    return button
end

local teamToggleButton = createToggleButton("Team Check: ON", 10, 200, 0.90, 30)

local function UpdateTeamToggle()
    if TEAM_CHECK_ENABLED then
        teamToggleButton.Text = "Team Check: ON"
        teamToggleButton.BackgroundColor3 = Color3.fromRGB(30, 120, 255)
    else
        teamToggleButton.Text = "Team Check: OFF"
        teamToggleButton.BackgroundColor3 = Color3.fromRGB(90, 90, 95)
    end
end

teamToggleButton.MouseButton1Click:Connect(function()
    TEAM_CHECK_ENABLED = not TEAM_CHECK_ENABLED
    UpdateTeamToggle()
end)

local ApplyButton = Instance.new("TextButton")
ApplyButton.Size = UDim2.new(0.46, -8, 0, 30)
ApplyButton.Position = UDim2.new(0, 10, 1, -38)
ApplyButton.BackgroundColor3 = Color3.fromRGB(30, 120, 255)
ApplyButton.Text = "Aplicar"
ApplyButton.TextColor3 = Color3.fromRGB(255, 255, 255)
ApplyButton.Font = Enum.Font.SourceSansBold
ApplyButton.TextSize = 15
ApplyButton.BorderSizePixel = 0
ApplyButton.Parent = ConfigFrame

local applyCorner = Instance.new("UICorner")
applyCorner.CornerRadius = UDim.new(0, 8)
applyCorner.Parent = ApplyButton

local ResetButton = Instance.new("TextButton")
ResetButton.Size = UDim2.new(0.46, -8, 0, 30)
ResetButton.Position = UDim2.new(0.5, 4, 1, -38)
ResetButton.BackgroundColor3 = Color3.fromRGB(80, 80, 85)
ResetButton.Text = "Reset"
ResetButton.TextColor3 = Color3.fromRGB(255, 255, 255)
ResetButton.Font = Enum.Font.SourceSansBold
ResetButton.TextSize = 15
ResetButton.BorderSizePixel = 0
ResetButton.Parent = ConfigFrame

local resetCorner = Instance.new("UICorner")
resetCorner.CornerRadius = UDim.new(0, 8)
resetCorner.Parent = ResetButton

local function ApplyConfig()
    local newDist = tonumber(espDistBox.Text) or MAX_DISTANCE
    local newFov = tonumber(fovBox.Text) or FOV_RADIUS
    local newPred = tonumber(predictionBox.Text) or PREDICTION_TIME

    MAX_DISTANCE = math.clamp(newDist, 50, 5000)
    FOV_RADIUS = math.clamp(newFov, 0, 500)
    PREDICTION_TIME = math.clamp(newPred, 0, 1)

    FOVCircle.Radius = FOV_RADIUS

    espDistBox.Text = tostring(MAX_DISTANCE)
    fovBox.Text = tostring(FOV_RADIUS)
    predictionBox.Text = tostring(PREDICTION_TIME)
end

ApplyButton.MouseButton1Click:Connect(ApplyConfig)

ResetButton.MouseButton1Click:Connect(function()
    MAX_DISTANCE = 1000
    FOV_RADIUS = 150
    PREDICTION_TIME = 0.10

    espDistBox.Text = tostring(MAX_DISTANCE)
    fovBox.Text = tostring(FOV_RADIUS)
    predictionBox.Text = tostring(PREDICTION_TIME)

    FOVCircle.Radius = FOV_RADIUS
end)

UserInputService.InputBegan:Connect(function(input, gameProcessedEvent)
    if gameProcessedEvent then return end
    if input.KeyCode == Enum.KeyCode.RightShift then
        ConfigGui.Enabled = not ConfigGui.Enabled
    end
end)

UpdateTeamToggle()

-- ================= PRELOAD =================
local bgImage = BACKGROUND_ASSET_ID
local ok, err = pcall(function()
    ContentProvider:PreloadAsync({ bgImage })
end)

if not ok then
    -- ignora se o asset não carregar
end

-- ================= LOOP PRINCIPAL =================
RunService.RenderStepped:Connect(function(dt)
    if not Camera then return end

    local vp = Camera.ViewportSize
    FOVCircle.Position = Vector2.new(vp.X / 2, vp.Y / 2)

    if FOV_RGB then
        hue = (hue + dt * 0.03) % 1
        FOVCircle.Color = Color3.fromHSV(hue, 1, 1)
    end

    local now = tickf()
    if now - lastScan >= SCAN_INTERVAL then
        lastScan = now
        ScanAndUpdate()
    end

    local localChar = LocalPlayer and LocalPlayer.Character
    local localRoot = localChar and localChar:FindFirstChild("HumanoidRootPart")
    if AIM_ENABLED and UserInputService:IsMouseButtonPressed(AIM_KEY) and localRoot and CURRENT_PART and CURRENT_PART.Parent then
        local predictedPos = CURRENT_PART.Position
        if AIM_PREDICTION_ENABLED and PREDICTION_TIME > 0 then
            local vel, acc = EstimateVelAcc(CURRENT_PART)
            local t = PREDICTION_TIME * AGGRESSIVENESS
            predictedPos = predictedPos + vel * t + 0.5 * acc * (t * t)
        end

        local cameraPos = Camera.CFrame.Position
        local desired = CFrame.lookAt(cameraPos, predictedPos)
        local lerpFactor = math_clamp(1 - SMOOTHING, 0, 1)
        Camera.CFrame = Camera.CFrame:Lerp(desired, lerpFactor)
    end

    if now - lastESP >= ESP_UPDATE_INTERVAL then
        lastESP = now
        if ESP_RGB then
            for _, data in pairs(PLAYER_ESP) do
                if data and data.Gui.Enabled then
                    data.Text.TextColor3 = Color3.fromHSV(hue, 1, 1)
                end
            end
            for _, data in pairs(NPC_ESP) do
                if data and data.Gui.Enabled then
                    data.Text.TextColor3 = Color3.fromHSV((hue + 0.15) % 1, 1, 1)
                end
            end
        end
    end
end)
