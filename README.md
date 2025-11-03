local RunService = game:GetService("RunService")
local Players = game:GetService("Players")
local LocalPlayer = Players.LocalPlayer
local Camera = workspace.CurrentCamera
local UserInputService = game:GetService("UserInputService")

local ESPEnabled = false

local BoxFolder = Instance.new("Folder", workspace)
BoxFolder.Name = "ESPBoxes"

local function createBox()
    local box = Instance.new("BoxHandleAdornment")
    box.ZIndex = 10
    box.AlwaysOnTop = true
    box.Visible = false
    box.Adornee = nil
    box.Size = Vector3.new(4, 6, 1)
    box.Transparency = 0.5
    box.Parent = BoxFolder
    return box
end

local Boxes = {}

local function isOnScreen(pos)
    local screenPos, inView = Camera:WorldToViewportPoint(pos)
    return inView, screenPos
end

local function createArrow()
    local arrow = Drawing.new("Triangle")
    arrow.Filled = true
    arrow.Transparency = 1
    arrow.Color = Color3.fromRGB(255,255,255)
    return arrow
end

local Arrows = {}

local FOVRadius = 150

-- Ativa ou desativa com a tecla P
UserInputService.InputBegan:Connect(function(input, gameProcessed)
    if not gameProcessed and input.KeyCode == Enum.KeyCode.P then
        ESPEnabled = not ESPEnabled
        BoxFolder.Parent = ESPEnabled and workspace or nil
        for _, box in pairs(Boxes) do
            box.Visible = ESPEnabled
        end
        for _, arrow in pairs(Arrows) do
            arrow.Visible = ESPEnabled
        end
    end
end)

local function update()
    if not ESPEnabled then return end

    local localTeam = LocalPlayer.Team

    for _, player in pairs(Players:GetPlayers()) do
        if player ~= LocalPlayer and player.Character and player.Character:FindFirstChild("HumanoidRootPart") then
            local root = player.Character.HumanoidRootPart
            local box = Boxes[player] or createBox()
            Boxes[player] = box
            box.Adornee = player.Character

            local inTeam = player.Team == localTeam
            box.Color3 = inTeam and Color3.fromRGB(0, 0, 255) or Color3.fromRGB(255, 0, 0)
            box.Visible = true

            -- FOV Highlight - só mostra se estiver dentro da distância do FOV
            local distance = (root.Position - LocalPlayer.Character.HumanoidRootPart.Position).Magnitude
            local screenPos, onScreen = Camera:WorldToViewportPoint(root.Position)
            local inFOV = (Vector2.new(screenPos.X, screenPos.Y) - Vector2.new(Camera.ViewportSize.X/2, Camera.ViewportSize.Y/2)).Magnitude <= FOVRadius

            -- Atualiza ou cria flechas para inimigos fora do FOV
            local arrow = Arrows[player] or createArrow()
            Arrows[player] = arrow
            if not inTeam and not inFOV and distance <= 1000 then
                arrow.Visible = true
                arrow.Color = Color3.fromRGB(255, 0, 0)
                local center = Vector2.new(Camera.ViewportSize.X/2, Camera.ViewportSize.Y/2)
                local direction = (Vector2.new(screenPos.X, screenPos.Y) - center).Unit
                local arrowPos = center + direction * (FOVRadius + 20)
                arrow.PointA = arrowPos + Vector2.new(0, -10)
                arrow.PointB = arrowPos + Vector2.new(5, 10)
                arrow.PointC = arrowPos + Vector2.new(-5, 10)
            else
                arrow.Visible = false
            end

            -- Esconde box se fora do FOV e muito longe
            box.Visible = distance <= 1000 and (inTeam or inFOV)
        elseif Boxes[player] then
            Boxes[player].Visible = false
            if Arrows[player] then Arrows[player].Visible = false end
        end
    end
end

RunService.RenderStepped:Connect(update)