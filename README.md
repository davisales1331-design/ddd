local Players = game:GetService("Players")
local LocalPlayer = Players.LocalPlayer
local Camera = workspace.CurrentCamera
local RunService = game:GetService("RunService")
local ESP_Renders = {}
local toggles = { box = false, name = false, life = false }
local BOX_W, BOX_H = 64, 110

local function getScreenBox(char)
    local hrp = char:FindFirstChild("HumanoidRootPart")
    if not hrp then return end
    local center3d = hrp.Position
    local center2d,vis = Camera:WorldToViewportPoint(center3d)
    if not vis then return end
    local x,y = center2d.X, center2d.Y
    local w,h = BOX_W, BOX_H
    return Vector2.new(x-w/2, y-h/2), Vector2.new(x+w/2, y-h/2), Vector2.new(x+w/2, y+h/2), Vector2.new(x-w/2, y+h/2), Vector2.new(x, y-h/2), Vector2.new(x-w/2, y), Vector2.new(x-w/2-16, y)
end

local function createESP(char,plr)
    local id = char:GetDebugId()
    if ESP_Renders[id] then return end
    local corners = {}
    for i=1,4 do
        local lh = Drawing.new("Line") lh.Thickness=2 lh.Color=Color3.fromRGB(255,255,0) lh.Visible=false
        local lv = Drawing.new("Line") lv.Thickness=2 lv.Color=Color3.fromRGB(255,255,0) lv.Visible=false
        corners[i.."H"] = lh
        corners[i.."V"] = lv
    end
    ESP_Renders[id.."corners"] = corners
    local name = Drawing.new("Text")
    name.Size = 14 name.Center = true name.Outline=true name.Color=Color3.fromRGB(255,255,255) name.Visible=false
    ESP_Renders[id.."name"] = name
    local lifebar = Drawing.new("Square")
    lifebar.Filled=true lifebar.Visible=false lifebar.Color=Color3.fromRGB(0,255,0)
    ESP_Renders[id.."lifebar"] = lifebar
    local lifenum = Drawing.new("Text")
    lifenum.Size=14 lifenum.Center=true lifenum.Outline=true lifenum.Color=Color3.fromRGB(0,255,0) lifenum.Rotation=180 lifenum.Visible=false
    ESP_Renders[id.."lifenum"] = lifenum
end

local function removeESP(char)
    local id = char:GetDebugId()
    for k,v in pairs(ESP_Renders) do
        if string.find(k,id) then
            if k:find("corners") then for _,l in pairs(v) do l:Remove() end else v:Remove() end
            ESP_Renders[k] = nil
        end
    end
end

local function clearESP()
    for k,v in pairs(ESP_Renders) do
        if k:find("corners") then for _,l in pairs(v) do l:Remove() end else v:Remove() end
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
            local cns=ESP_Renders[id.."corners"]
            local name=ESP_Renders[id.."name"]
            local lifebar=ESP_Renders[id.."lifebar"]
            local lifenum=ESP_Renders[id.."lifenum"]
            local p1,p2,p3,p4,ptop,plife,plifen=getScreenBox(char)
            if not (p1 and p2 and p3 and p4) then
                for i=1,4 do cns[i.."H"].Visible=false cns[i.."V"].Visible=false end
                name.Visible=false lifebar.Visible=false lifenum.Visible=false
                continue
            end
            local lsize = math.floor((p4.Y-p1.Y)/5)
            cns["1H"].From = p1  cns["1H"].To = Vector2.new(p1.X+lsize,p1.Y)
            cns["1H"].Color = Color3.fromRGB(255,255,0)
            cns["1V"].From = p1  cns["1V"].To = Vector2.new(p1.X,p1.Y+lsize)
            cns["1V"].Color = Color3.fromRGB(0,255,0)
            cns["2H"].From = p2  cns["2H"].To = Vector2.new(p2.X-lsize,p2.Y)
            cns["2H"].Color = Color3.fromRGB(255,255,0)
            cns["2V"].From = p2  cns["2V"].To = Vector2.new(p2.X,p2.Y+lsize)
            cns["2V"].Color = Color3.fromRGB(255,255,0)
            cns["3H"].From = p3  cns["3H"].To = Vector2.new(p3.X-lsize,p3.Y)
            cns["3H"].Color = Color3.fromRGB(255,255,0)
            cns["3V"].From = p3  cns["3V"].To = Vector2.new(p3.X,p3.Y-lsize)
            cns["3V"].Color = Color3.fromRGB(255,255,0)
            cns["4H"].From = p4  cns["4H"].To = Vector2.new(p4.X+lsize,p4.Y)
            cns["4H"].Color = Color3.fromRGB(255,255,0)
            cns["4V"].From = p4  cns["4V"].To = Vector2.new(p4.X,p4.Y-lsize)
            cns["4V"].Color = Color3.fromRGB(0,255,0)
            for i=1,4 do cns[i.."H"].Visible=toggles.box cns[i.."V"].Visible=toggles.box end
            name.Position=Vector2.new((p1.X+p2.X)/2,p1.Y-16)
            name.Text=plr.Name name.Visible=toggles.name
            if toggles.life then
                local perc=math.floor((hum.Health/hum.MaxHealth)*100)
                local barH=math.abs(p3.Y-p1.Y)
                local barX=p1.X-14
                local barY=math.min(p1.Y,p3.Y)
                lifebar.Position=Vector2.new(barX,barY)
                lifebar.Size=Vector2.new(5, barH*(hum.Health/hum.MaxHealth))
                lifebar.Visible=true
                lifenum.Position=Vector2.new(barX-12,barY+barH)
                lifenum.Text=perc.."%"
                lifenum.Visible=true
            else lifebar.Visible=false lifenum.Visible=false end
        else if char then removeESP(char) end end
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
    Title = "ESP Box",
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
