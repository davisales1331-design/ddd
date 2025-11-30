local Players = game:GetService("Players")
local LocalPlayer = Players.LocalPlayer
local Camera = workspace.CurrentCamera
local RunService = game:GetService("RunService")
local UserInputService = game:GetService("UserInputService")
local ESP_Renders = {}
local toggles = { box = false, name = false, life = false }
local L_SIZE = 12
local LIFE_BAR_OFFSET = 28

local function get3DBoxCorners(char)
    local hrp = char:FindFirstChild("HumanoidRootPart")
    if not hrp then return end
    local c = hrp.Position
    local size = char:GetExtentsSize()
    local sx, sy, sz = size.X/2, size.Y/2, size.Z/2
    return {
        TL = c + Vector3.new(-sx, sy, -sz),
        TR = c + Vector3.new(sx, sy, -sz),
        BR = c + Vector3.new(sx, -sy, -sz),
        BL = c + Vector3.new(-sx, -sy, -sz)
    }
end

local function createESP(char, plr)
    local id = char:GetDebugId()
    if ESP_Renders[id] then return end
    local lines = {}
    for i=1,4 do
        local lh = Drawing.new("Line") lh.Thickness=2 lh.Color=Color3.fromRGB(255,255,0) lh.Visible=false
        local lv = Drawing.new("Line") lv.Thickness=2 lv.Color=Color3.fromRGB(255,255,0) lv.Visible=false
        lines[i.."H"]=lh lines[i.."V"]=lv
    end
    local name = Drawing.new("Text")
    name.Size=14 name.Center=true name.Outline=true name.Color=Color3.fromRGB(255,255,255) name.Visible=false
    local lifebar = Drawing.new("Line")
    lifebar.Thickness=6 lifebar.Color=Color3.fromRGB(0,255,0) lifebar.Visible=false
    local lifenum = Drawing.new("Text")
    lifenum.Size=14 lifenum.Center=true lifenum.Outline=true lifenum.Color=Color3.fromRGB(0,255,0) lifenum.Rotation=180 lifenum.Visible=false
    ESP_Renders[id] = {L=lines, name=name, lifebar=lifebar, lifenum=lifenum, Char=char}
end

local function removeESP(char)
    local id = char:GetDebugId()
    local tbl = ESP_Renders[id]
    if not tbl then return end
    for k,v in pairs(tbl) do
        if typeof(v)=="table" then for _,d in pairs(v) do d:Remove() end 
        elseif typeof(v)=="Instance" or typeof(v)=="Drawing" then v:Remove() end
    end
    ESP_Renders[id]=nil
end

local function clearESP()
    for _,tbl in pairs(ESP_Renders) do
        for k,v in pairs(tbl) do
            if typeof(v)=="table" then for _,d in pairs(v) do d:Remove() end 
            elseif typeof(v)=="Instance" or typeof(v)=="Drawing" then v:Remove() end
        end
    end
    ESP_Renders = {}
end

local function updateESP()
    for id, tbl in pairs(ESP_Renders) do
        local char = tbl.Char
        local hum = char and char:FindFirstChildOfClass("Humanoid")
        if not char or not hum or hum.Health <= 0 or not char:FindFirstChild("HumanoidRootPart") then
            removeESP(char)
        end
    end
    for _,plr in ipairs(Players:GetPlayers()) do
        if plr == LocalPlayer then continue end
        local char = plr.Character
        local hum = char and char:FindFirstChildOfClass("Humanoid")
        if hum and hum.Health > 0 and char:FindFirstChild("HumanoidRootPart") then
            if toggles.box or toggles.name or toggles.life then createESP(char, plr) end
            local id = char:GetDebugId()
            local tbl = ESP_Renders[id]
            if not tbl then continue end
            local Ls, name, lifebar, lifenum = tbl.L, tbl.name, tbl.lifebar, tbl.lifenum
            local v = get3DBoxCorners(char)
            if not v then for i=1,4 do Ls[i.."H"].Visible=false Ls[i.."V"].Visible=false end name.Visible=false lifebar.Visible=false lifenum.Visible=false continue end
            local vp = function(vec) local p,u=Camera:WorldToViewportPoint(vec) return Vector2.new(p.X,p.Y),u end
            local pTL,u1 = vp(v.TL) local pTR,u2 = vp(v.TR) local pBR,u3 = vp(v.BR) local pBL,u4 = vp(v.BL)
            if not(u1 and u2 and u3 and u4) then for i=1,4 do Ls[i.."H"].Visible=false Ls[i.."V"].Visible=false end name.Visible=false lifebar.Visible=false lifenum.Visible=false continue end
            local L = L_SIZE
            Ls["1H"].From = pTL Ls["1H"].To = pTL+Vector2.new(L,0)
            Ls["1H"].Color = Color3.fromRGB(255,255,0)
            Ls["1V"].From = pTL Ls["1V"].To = pTL+Vector2.new(0,L)
            Ls["1V"].Color = Color3.fromRGB(0,255,0)
            Ls["2H"].From = pTR Ls["2H"].To = pTR-Vector2.new(L,0)
            Ls["2V"].From = pTR Ls["2V"].To = pTR+Vector2.new(0,L)
            Ls["2V"].Color = Color3.fromRGB(255,255,0)
            Ls["3H"].From = pBR Ls["3H"].To = pBR-Vector2.new(L,0)
            Ls["3V"].From = pBR Ls["3V"].To = pBR-Vector2.new(0,L)
            Ls["3V"].Color = Color3.fromRGB(255,255,0)
            Ls["4H"].From = pBL Ls["4H"].To = pBL+Vector2.new(L,0)
            Ls["4V"].From = pBL Ls["4V"].To = pBL-Vector2.new(0,L)
            Ls["4V"].Color = Color3.fromRGB(0,255,0)
            for i=1,4 do Ls[i.."H"].Visible=toggles.box Ls[i.."V"].Visible=toggles.box end
            name.Position=Vector2.new((pTL.X+pTR.X)/2, pTL.Y-16)
            name.Text=plr.Name
            name.Visible=toggles.name
            if toggles.life then
                local ratio = math.max(0, math.min(1, hum.Health/hum.MaxHealth))
                local barYmin = (pTL.Y+pBL.Y)/2
                local barYmax = (pTR.Y+pBR.Y)/2
                local barTop = Vector2.new(pTL.X-LIFE_BAR_OFFSET, pTL.Y)
                local barBot = Vector2.new(pBL.X-LIFE_BAR_OFFSET, pBL.Y)
                local filledEnd = Vector2.new(barTop.X, barTop.Y + (barBot.Y-barTop.Y)*ratio)
                lifebar.From = barTop
                lifebar.To = filledEnd
                lifebar.Visible=true
                lifenum.Position=Vector2.new(barTop.X-14,barBot.Y)
                lifenum.Text = math.floor(ratio*100).."%"
                lifenum.Visible=true
            else
                lifebar.Visible=false
                lifenum.Visible=false
            end
        end
    end
end

Players.PlayerAdded:Connect(function(plr)
    plr.CharacterAdded:Connect(function(char)
        if toggles.box or toggles.name or toggles.life then createESP(char,plr) end
    end)
end)
for _,plr in ipairs(Players:GetPlayers()) do
    if plr~=LocalPlayer and plr.Character then
        if toggles.box or toggles.name or toggles.life then createESP(plr.Character,plr) end
    end
    plr.CharacterAdded:Connect(function(char)
        if toggles.box or toggles.name or toggles.life then createESP(char,plr) end
    end)
end
Players.PlayerRemoving:Connect(function(plr)
    if plr~=LocalPlayer and plr.Character then removeESP(plr.Character) end
end)
RunService.RenderStepped:Connect(updateESP)

local Fluent = loadstring(game:HttpGet("https://github.com/dawid-scripts/Fluent/releases/latest/download/main.lua"))()
local Window = Fluent:CreateWindow{
    Title = "GS STORE",
    SubTitle = "SALES DEVELOPER",
    TabWidth = 160,
    Size = UDim2.fromOffset(500, 400),
    Acrylic = true,
    Theme = "Darker",
    MinimizeKey = Enum.KeyCode.Insert
}

local Tabs = {
    Visual = Window:AddTab{ Title = "Visual", Icon = "" },
    Mira = Window:AddTab{ Title = "Mira", Icon = "" },
    Player = Window:AddTab{ Title = "Player", Icon = "" }
}

Tabs.Visual:AddToggle("ESPBoxToggle", {
    Title = "ESP Box L",
    Callback = function(state)
        toggles.box = state
        clearESP()
        for _,plr in ipairs(Players:GetPlayers()) do
            if plr~=LocalPlayer and plr.Character and (state or toggles.name or toggles.life) then
                createESP(plr.Character,plr)
            end
        end
    end
})
Tabs.Visual:AddToggle("ESPNameToggle", {
    Title = "ESP Nome",
    Callback = function(state)
        toggles.name = state
        clearESP()
        for _,plr in ipairs(Players:GetPlayers()) do
            if plr~=LocalPlayer and plr.Character and (toggles.box or state or toggles.life) then
                createESP(plr.Character,plr)
            end
        end
    end
})
Tabs.Visual:AddToggle("ESPLifeToggle", {
    Title = "ESP Vida",
    Callback = function(state)
        toggles.life = state
        clearESP()
        for _,plr in ipairs(Players:GetPlayers()) do
            if plr~=LocalPlayer and plr.Character and (toggles.box or toggles.name or state) then
                createESP(plr.Character,plr)
            end
        end
    end
})

local FOV_Circle = Drawing.new("Circle")
FOV_Circle.Transparency = 1
FOV_Circle.Thickness = 2
FOV_Circle.Filled = false
FOV_Circle.NumSides = 100
FOV_Circle.Color = Color3.fromRGB(255,255,255)
local FOV_Size = 120
FOV_Circle.Visible = true

Tabs.Mira:AddSlider("SliderFOV", {
    Title = "Tamanho do FOV",
    Min = 50, Max = 500, Default = FOV_Size, Rounding = 0,
    Callback = function(val)
        FOV_Size = val
        FOV_Circle.Radius = val
    end
})
Tabs.Mira:AddColorpicker("FOV_Color", {
    Title = "Cor do FOV",
    Default = Color3.fromRGB(255,255,255),
}):OnChanged(function(col)
    FOV_Circle.Color = col
end)

local AimAssistStrengths = {["Fraco"]=0.15,["Médio"]=0.35,["Forte"]=0.6,["Muito Forte"]=1}
local aimAssistActive, aimAssistStrength, aiming = false, AimAssistStrengths["Médio"], false

Tabs.Mira:AddDropdown("AimAssistStrength", {
    Title = "Auxiliar de Mira - Força",
    Values = { "Fraco", "Médio", "Forte", "Muito Forte" },
    Default = "Médio",
    Callback = function(val)
        aimAssistStrength = AimAssistStrengths[val]
    end
})
Tabs.Mira:AddToggle("AimAssistToggle", {
    Title = "Auxiliar de Mira",
    Description = "Segure botão direito do mouse",
    Default = false,
    Callback = function(state) aimAssistActive = state end
})

UserInputService.InputBegan:Connect(function(input,_)
    if input.UserInputType == Enum.UserInputType.MouseButton2 then aiming = true end
end)
UserInputService.InputEnded:Connect(function(input,_)
    if input.UserInputType == Enum.UserInputType.MouseButton2 then aiming = false end
end)

local lockedTarget, lastDist = nil, math.huge
local function GetClosestHeadInFOV()
    local center = Vector2.new(Camera.ViewportSize.X / 2, Camera.ViewportSize.Y / 2)
    local closest, minDist = nil, math.huge
    for _,plr in ipairs(Players:GetPlayers()) do
        if plr ~= LocalPlayer and plr.Character and plr.Character:FindFirstChild("Head")
        and plr.Character:FindFirstChildOfClass("Humanoid") and plr.Character:FindFirstChildOfClass("Humanoid").Health > 0 then
            local head = plr.Character.Head
            local sp,visible = Camera:WorldToViewportPoint(head.Position)
            if visible then
                local dist = (Vector2.new(sp.X, sp.Y) - center).Magnitude
                if dist < FOV_Size and dist < minDist then
                    minDist = dist
                    closest = head
                end
            end
        end
    end
    return closest,minDist
end

RunService.RenderStepped:Connect(function()
    updateESP()
    FOV_Circle.Radius = FOV_Size
    FOV_Circle.Position = Vector2.new(Camera.ViewportSize.X/2, Camera.ViewportSize.Y/2)
    FOV_Circle.Visible = aimAssistActive
    if aimAssistActive and aiming then
        local targetHead, dist = GetClosestHeadInFOV()
        if targetHead ~= lockedTarget or (lockedTarget and dist < lastDist-1) then
            lockedTarget = targetHead
            lastDist = dist or math.huge
        end
        if lockedTarget then
            local screenPos,visible = Camera:WorldToViewportPoint(lockedTarget.Position)
            if visible then
                local centerScreen = Vector2.new(Camera.ViewportSize.X/2, Camera.ViewportSize.Y/2)
                local toTarget = Vector2.new(screenPos.X, screenPos.Y) - centerScreen
                local moveX = toTarget.X * aimAssistStrength
                local moveY = toTarget.Y * aimAssistStrength
                pcall(function()
                    if getgenv and getgenv().mousemoverel then
                        getgenv().mousemoverel(moveX, moveY)
                    end
                end)
            end
        end
    else
        lockedTarget = nil
        lastDist = math.huge
    end
end)

Tabs.Player:AddLabel("Bem-vindo ao menu do jogador!")
Tabs.Player:AddButton("Resetar personagem", function()
    if LocalPlayer.Character and LocalPlayer.Character:FindFirstChildOfClass("Humanoid") then
        LocalPlayer.Character:FindFirstChildOfClass("Humanoid").Health = 0
    end
end)
Tabs.Player:AddButton("Dar velocidade 50", function()
    if LocalPlayer.Character and LocalPlayer.Character:FindFirstChildOfClass("Humanoid") then
        LocalPlayer.Character:FindFirstChildOfClass("Humanoid").WalkSpeed = 50
    end
end)
Tabs.Player:AddButton("Restaurar velocidade padrão", function()
    if LocalPlayer.Character and LocalPlayer.Character:FindFirstChildOfClass("Humanoid") then
        LocalPlayer.Character:FindFirstChildOfClass("Humanoid").WalkSpeed = 16
    end
end)
Tabs.Player:AddButton("Pular muito alto (JumpPower 100)", function()
    if LocalPlayer.Character and LocalPlayer.Character:FindFirstChildOfClass("Humanoid") then
        LocalPlayer.Character:FindFirstChildOfClass("Humanoid").JumpPower = 100
    end
end)
Tabs.Player:AddButton("Restaurar pulo padrão (JumpPower 50)", function()
    if LocalPlayer.Character and LocalPlayer.Character:FindFirstChildOfClass("Humanoid") then
        LocalPlayer.Character:FindFirstChildOfClass("Humanoid").JumpPower = 50
    end
end)
Tabs.Player:AddToggle("GodmodeToggle", {
    Title = "Godmode (invencível)",
    Description = "Ativar Invencibilidade",
    Default = false,
    Callback = function(state)
        if LocalPlayer.Character and LocalPlayer.Character:FindFirstChildOfClass("Humanoid") then
            if state then
                LocalPlayer.Character:FindFirstChildOfClass("Humanoid").MaxHealth = math.huge
                LocalPlayer.Character:FindFirstChildOfClass("Humanoid").Health = math.huge
            else
                LocalPlayer.Character:FindFirstChildOfClass("Humanoid").MaxHealth = 100
                LocalPlayer.Character:FindFirstChildOfClass("Humanoid").Health = 100
            end
        end
    end
})
