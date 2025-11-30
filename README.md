local Players = game:GetService("Players")
local LocalPlayer = Players.LocalPlayer
local Camera = workspace.CurrentCamera
local RunService = game:GetService("RunService")
local ESP_Renders = {}
local toggles = { box = false, name = false, life = false }
local L_SIZE = 12

local function get3DBoxCorners(char)
    local hrp = char:FindFirstChild("HumanoidRootPart")
    if not hrp then return end
    local c = hrp.Position
    local size = char:GetExtentsSize()
    local sx = size.X/2
    local sy = size.Y/2
    local sz = size.Z/2
    local v = {
        TL = c + Vector3.new(-sx, sy, -sz),
        TR = c + Vector3.new(sx, sy, -sz),
        BR = c + Vector3.new(sx, -sy, -sz),
        BL = c + Vector3.new(-sx, -sy, -sz)
    }
    return v
end

local function createESP(char, plr)
    local id = char:GetDebugId()
    if ESP_Renders[id] then return end
    local lines = {}
    for i=1,4 do
        local lh = Drawing.new("Line") lh.Thickness=2 lh.Color=Color3.fromRGB(255,255,0) lh.Visible=false
        local lv = Drawing.new("Line") lv.Thickness=2 lv.Color=Color3.fromRGB(255,255,0) lv.Visible=false
        lines[i.."H"] = lh
        lines[i.."V"] = lv
    end
    ESP_Renders[id.."L"] = lines
    local name = Drawing.new("Text")
    name.Size=14 name.Center=true name.Outline=true name.Color=Color3.fromRGB(255,255,255) name.Visible=false
    ESP_Renders[id.."name"] = name
    local lifebar = Drawing.new("Line")
    lifebar.Thickness=6 lifebar.Color=Color3.fromRGB(0,255,0) lifebar.Visible=false
    ESP_Renders[id.."lifebar"]=lifebar
    local lifenum = Drawing.new("Text")
    lifenum.Size=14 lifenum.Center=true lifenum.Outline=true lifenum.Color=Color3.fromRGB(0,255,0) lifenum.Rotation=180 lifenum.Visible=false
    ESP_Renders[id.."lifenum"]=lifenum
end

local function removeESP(char)
    local id=char:GetDebugId()
    for k,v in pairs(ESP_Renders) do
        if string.find(k,id) then
            if k:find("L") then for _,l in pairs(v) do l:Remove() end else v:Remove() end
            ESP_Renders[k]=nil
        end
    end
end

local function clearESP()
    for k,v in pairs(ESP_Renders) do
        if k:find("L") then for _,l in pairs(v) do l:Remove() end else v:Remove() end
    end
    ESP_Renders = {}
end

local function updateESP()
    for _,plr in ipairs(Players:GetPlayers()) do
        if plr==LocalPlayer then continue end
        local char=plr.Character
        local hum=char and char:FindFirstChildOfClass("Humanoid")
        if hum and hum.Health>0 and char:FindFirstChild("HumanoidRootPart") then
            if toggles.box or toggles.name or toggles.life then createESP(char,plr) end
            local id=char:GetDebugId()
            local Ls=ESP_Renders[id.."L"]
            local name=ESP_Renders[id.."name"]
            local lifebar=ESP_Renders[id.."lifebar"]
            local lifenum=ESP_Renders[id.."lifenum"]
            local v=get3DBoxCorners(char)
            if not v then for i=1,4 do Ls[i.."H"].Visible=false Ls[i.."V"].Visible=false end name.Visible=false lifebar.Visible=false lifenum.Visible=false continue end
            local vp=function(vec) local p,u=Camera:WorldToViewportPoint(vec) return Vector2.new(p.X,p.Y),u end
            local pTL,u1=vp(v.TL) local pTR,u2=vp(v.TR) local pBR,u3=vp(v.BR) local pBL,u4=vp(v.BL)
            if not(u1 and u2 and u3 and u4) then for i=1,4 do Ls[i.."H"].Visible=false Ls[i.."V"].Visible=false end name.Visible=false lifebar.Visible=false lifenum.Visible=false continue end
            -- L nos cantos
            local L=L_SIZE
            -- Top-Left
            Ls["1H"].From = pTL Ls["1H"].To = pTL+Vector2.new(L,0)
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
            for i=1,4 do Ls[i.."H"].Visible=toggles.box Ls[i.."V"].Visible=toggles.box end
            -- Nome no topo centro
            name.Position=Vector2.new((pTL.X+pTR.X)/2, pTL.Y-16)
            name.Text=plr.Name
            name.Visible=toggles.name
            -- Barra de vida
            if toggles.life then
                local ratio=math.max(0,math.min(1,hum.Health/hum.MaxHealth))
                local barYmin = (pTL.Y+pBL.Y)/2
                local barYmax = (pTR.Y+pBR.Y)/2
                local barLen = math.abs(barYmax-barYmin)
                local barTop = pTL
                local barBot = pBL
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
        else
            if char then removeESP(char) end
        end
    end
end

Players.PlayerAdded:Connect(function(plr) plr.CharacterAdded:Connect(function(char) if toggles.box or toggles.name or toggles.life then createESP(char,plr) end end) end)
for _,plr in ipairs(Players:GetPlayers()) do if plr~=LocalPlayer and plr.Character then if toggles.box or toggles.name or toggles.life then createESP(plr.Character,plr) end end plr.CharacterAdded:Connect(function(char) if toggles.box or toggles.name or toggles.life then createESP(char,plr) end end) end
Players.PlayerRemoving:Connect(function(plr) if plr~=LocalPlayer and plr.Character then removeESP(plr.Character) end end)
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
