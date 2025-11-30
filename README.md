local Players = game:GetService("Players")
local LocalPlayer = Players.LocalPlayer
local Camera = workspace.CurrentCamera
local RunService = game:GetService("RunService")
local L_SIZE = 12
local LIFE_BAR_OFFSET = 30
local ESP_Renders = {}
local toggles = { box = false, name = false, life = false }

local function getBoxAroundHRP(char)
    local hrp = char:FindFirstChild("HumanoidRootPart")
    if not hrp then return end
    local size = char:GetExtentsSize()
    local c = hrp.Position
    local sx, sy = size.X/2, size.Y/2
    return {
        TL = c + Vector3.new(-sx, sy, 0),
        TR = c + Vector3.new(sx, sy, 0),
        BR = c + Vector3.new(sx, -sy, 0),
        BL = c + Vector3.new(-sx, -sy, 0),
        Center = c,
    }
end

local function isDrawing(obj)
    return typeof(obj) == "Drawing" or (typeof(obj) == "Instance" and pcall(function() return obj.Remove end))
end

local function createESP(char,plr)
    local id = char:GetDebugId()
    if ESP_Renders[id] then return end
    local lines = {}
    for i=1,4 do
        local lh = Drawing.new("Line") lh.Thickness=2 lh.Color=Color3.fromRGB(255,255,0) lh.Visible=false
        local lv = Drawing.new("Line") lv.Thickness=2 lv.Color=Color3.fromRGB(255,255,0) lv.Visible=false
        lines[i.."H"] = lh
        lines[i.."V"] = lv
    end
    local name = Drawing.new("Text")
    name.Size=14 name.Center=true name.Outline=true name.Color=Color3.fromRGB(255,255,255) name.Visible=false
    local lifebar = Drawing.new("Line")
    lifebar.Thickness=6 lifebar.Color=Color3.fromRGB(0,255,0) lifebar.Visible=false
    local lifenum = Drawing.new("Text")
    lifenum.Size=14 lifenum.Center=true lifenum.Outline=true lifenum.Color=Color3.fromRGB(0,255,0) lifenum.Rotation=180 lifenum.Visible=false
    ESP_Renders[id] = {
        L=lines,
        name=name,
        lifebar=lifebar,
        lifenum=lifenum,
        char=char
    }
end

local function removeESP(char)
    local id = char:GetDebugId()
    local tbl = ESP_Renders[id]
    if not tbl then return end
    for k,v in pairs(tbl) do
        if typeof(v)=="table" then for _,d in pairs(v) do if isDrawing(d) then d:Remove() end end
        elseif isDrawing(v) then v:Remove() end
    end
    ESP_Renders[id]=nil
end

local function clearESP()
    for _,tbl in pairs(ESP_Renders) do
        for k,v in pairs(tbl) do
            if typeof(v)=="table" then for _,d in pairs(v) do if isDrawing(d) then d:Remove() end end
            elseif isDrawing(v) then v:Remove() end
        end
    end
    ESP_Renders = {}
end

local function updateESP()
    local exists = {}
    for _,plr in ipairs(Players:GetPlayers()) do
        if plr==LocalPlayer then continue end
        local char = plr.Character
        local hum = char and char:FindFirstChildOfClass("Humanoid")
        if hum and hum.Health > 0 and char:FindFirstChild("HumanoidRootPart") then
            if toggles.box or toggles.name or toggles.life then createESP(char,plr) end
            local id = char:GetDebugId()
            exists[id] = true
            local tbl = ESP_Renders[id]
            if not tbl then continue end
            local Ls=tbl.L
            local name=tbl.name
            local lifebar=tbl.lifebar
            local lifenum=tbl.lifenum
            local v=getBoxAroundHRP(char)
            local vp=function(vec) local p,u=Camera:WorldToViewportPoint(vec) return Vector2.new(p.X,p.Y),u end
            local pTL,uTL=vp(v.TL) local pTR,uTR=vp(v.TR) local pBR,uBR=vp(v.BR) local pBL,uBL=vp(v.BL)
            local pCenter,uCenter=vp(v.Center)
            if not uCenter then
                for i=1,4 do Ls[i.."H"].Visible=false Ls[i.."V"].Visible=false end
                name.Visible=false lifebar.Visible=false lifenum.Visible=false
                continue
            end
            -- Mesmo se corners invisíveis, sempre desenha no HRP!
            local L = L_SIZE
            local showBox = toggles.box
            for i=1,4 do Ls[i.."H"].Visible=false Ls[i.."V"].Visible=false end
            if showBox and uTL and uTR and uBL and uBR then
                -- Top-Left
                Ls["1H"].From = pTL Ls["1H"].To = pTL+Vector2.new(L,0)
                Ls["1H"].Color = Color3.fromRGB(255,255,0)
                Ls["1V"].From = pTL Ls["1V"].To = pTL+Vector2.new(0,L)
                Ls["1V"].Color = Color3.fromRGB(0,255,0)
                -- Top-Right
                Ls["2H"].From = pTR Ls["2H"].To = pTR-Vector2.new(L,0)
                Ls["2V"].From = pTR Ls["2V"].To = pTR+Vector2.new(0,L)
                Ls["2V"].Color = Color3.fromRGB(255,255,0)
                -- Bottom-Right
                Ls["3H"].From = pBR Ls["3H"].To = pBR-Vector2.new(L,0)
                Ls["3V"].From = pBR Ls["3V"].To = pBR-Vector2.new(0,L)
                Ls["3V"].Color = Color3.fromRGB(255,255,0)
                -- Bottom-Left
                Ls["4H"].From = pBL Ls["4H"].To = pBL+Vector2.new(L,0)
                Ls["4V"].From = pBL Ls["4V"].To = pBL-Vector2.new(0,L)
                Ls["4V"].Color = Color3.fromRGB(0,255,0)
                for i=1,4 do Ls[i.."H"].Visible=true Ls[i.."V"].Visible=true end
            end
            name.Position=Vector2.new((pTL.X+pTR.X)/2, pTL.Y-16)
            name.Text=plr.Name
            name.Visible=toggles.name and uTL and uTR
            if toggles.life then
                local ratio=math.max(0,math.min(1,hum.Health/hum.MaxHealth))
                local barTop = Vector2.new(pTL.X-LIFE_BAR_OFFSET, pTL.Y)
                local barBot = Vector2.new(pBL.X-LIFE_BAR_OFFSET, pBL.Y)
                local filledEnd = Vector2.new(barTop.X, barTop.Y + (barBot.Y-barTop.Y)*ratio)
                lifebar.From = barTop
                lifebar.To = filledEnd
                lifebar.Visible=uTL and uBL
                lifenum.Position=Vector2.new(barTop.X-14,barBot.Y)
                lifenum.Text = math.floor(ratio*100).."%"
                lifenum.Visible=uBL
            else
                lifebar.Visible=false
                lifenum.Visible=false
            end
        end
    end
    for id,_ in pairs(ESP_Renders) do
        if not exists[id] then
            local tbl = ESP_Renders[id]
            if tbl and tbl.char then removeESP(tbl.char) end
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
