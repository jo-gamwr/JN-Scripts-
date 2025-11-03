-- main.lua
-- ESP Tester / Simulator + Team Check + UI
-- Para uso local / testes somente.

-- CONFIG
local WINDOW_W, WINDOW_H = 1280, 720
local TEAM_FRIEND = 1
local TEAM_ENEMY  = 2

-- State
local players = {}
local localPlayer = {id = 0, x = WINDOW_W/2, y = WINDOW_H/2, team = TEAM_FRIEND, name = "You", alive = true}
local ui = {
    showESP = true,
    teamCheck = true,
    showBoxes = true,
    showNames = true,
    maxRange = 600,
    boxColorFriend = {0, 1, 0, 0.6},
    boxColorEnemy  = {1, 0, 0, 0.8},
    highlightSuspect = true,
    detectionSensitivity = 0.5,
}
local logs = {}
local timeAccumulator = 0

-- Utility
local function log(msg)
    table.insert(logs, 1, string.format("[%s] %s", os.date("%H:%M:%S"), msg))
    if #logs > 8 then
        for i = 9, #logs do logs[i] = nil end
    end
    print(msg)
end

local function distance(a, b)
    return math.sqrt((a.x - b.x)^2 + (a.y - b.y)^2)
end

-- Create simulated players (some normal, some 'cheaters' for detection testing)
local function spawnSimulatedPlayers()
    players = {}
    local id = 1
    math.randomseed(os.time())
    for i=1,18 do
        local px = math.random(50, WINDOW_W-50)
        local py = math.random(50, WINDOW_H-50)
        local team = (i % 3 == 0) and TEAM_FRIEND or TEAM_ENEMY
        local p = {
            id = id,
            x = px,
            y = py,
            team = team,
            name = "P" .. id,
            hp = math.random(30,100),
            alive = true,
            vx = math.random(-40,40)/10,
            vy = math.random(-40,40)/10,
            -- simulation flags to act like "cheater" for testing detection:
            is_cheater = (i == 4 or i == 11), -- two simulated cheaters
            -- for detection metrics:
            aim_stability = math.random(),   -- 0..1, lower -> more jitter
            last_positions = {},
            last_update_time = love.timer.getTime(),
        }
        table.insert(players, p)
        id = id + 1
    end
end

-- Simple "detection" heuristics (for testing).
-- Not intended for production detection in games; just a demo of rule-based flags.
local function runDetection(dt)
    for _, p in ipairs(players) do
        if not p.alive then goto cont end

        -- record positions for velocity / teleport detection
        table.insert(p.last_positions, 1, {x=p.x, y=p.y, t=love.timer.getTime()})
        if #p.last_positions > 6 then table.remove(p.last_positions, 7) end

        -- compute average move speed
        if #p.last_positions >= 2 then
            local a = p.last_positions[1]
            local b = p.last_positions[#p.last_positions]
            local dtpos = a.t - b.t
            if dtpos > 0 then
                local spd = distance(a, b) / dtpos
                p._sim_speed = spd
            end
        end

        -- heuristic 1: teleport (sudden large movement)
        local teleport_threshold = 600 -- px / s (tunable)
        if p._sim_speed and p._sim_speed > teleport_threshold * (1 + ui.detectionSensitivity) then
            p._sus = p._sus or {}
            p._sus.teleport = true
            if ui.highlightSuspect then log("Suspected teleport: " .. p.name .. " speed=" .. string.format("%.1f", p._sim_speed)) end
        else
            if p._sus then p._sus.teleport = nil end
        end

        -- heuristic 2: aim stability (very stable aim might suggest assistance)
        -- We simulate by using aim_stability property.
        if p.aim_stability < 0.12 * (1 - ui.detectionSensitivity) then
            p._sus = p._sus or {}
            p._sus.stable_aim = true
            if ui.highlightSuspect then log("Suspected aim assist (sim): " .. p.name) end
        else
            if p._sus then p._sus.stable_aim = nil end
        end

        -- heuristic 3: very small movement while rotating (simulated): we'll treat low vx/vy + low jitter as suspicious
        local move_threshold = 6
        if math.abs(p.vx) + math.abs(p.vy) < move_threshold * (1 - ui.detectionSensitivity) then
            -- small movement; combine with other hints
            p._sus = p._sus or {}
            p._sus.low_movement = true
        else
            if p._sus then p._sus.low_movement = nil end
        end

        ::cont::
    end
end

-- LOVE callbacks

function love.load()
    love.window.setMode(WINDOW_W, WINDOW_H, {resizable=false})
    love.window.setTitle("ESP Tester + TeamCheck (Simulado)")
    spawnSimulatedPlayers()
    log("Simulador iniciado. Dois jogadores vêm marcados como 'cheaters' para teste.")
end

local font = love.graphics.newFont(12)
local bigfont = love.graphics.newFont(16)
love.graphics.setFont(font)

function love.update(dt)
    timeAccumulator = timeAccumulator + dt
    -- Move players
    for _, p in ipairs(players) do
        if p.alive then
            -- simulated movement; cheaters may teleport/jump
            if p.is_cheater and math.random() < 0.02 then
                -- teleport burst
                p.x = math.random(50, WINDOW_W-50)
                p.y = math.random(50, WINDOW_H-50)
            else
                p.x = p.x + p.vx * 60 * dt
                p.y = p.y + p.vy * 60 * dt
                -- keep on screen
                p.x = math.max(20, math.min(WINDOW_W-20, p.x))
                p.y = math.max(20, math.min(WINDOW_H-20, p.y))
            end
        end
    end

    -- run detection once per 0.1s to simulate analyzer
    if timeAccumulator > 0.08 then
        runDetection(timeAccumulator)
        timeAccumulator = 0
    end
end

-- UI helpers
local function drawRoundedRectOutline(x,y,w,h, r)
    love.graphics.rectangle("line", x+1, y+1, w-2, h-2, r, r)
end

local function drawToggle(x,y,label, value)
    local w, h = 14, 14
    love.graphics.setColor(0.9,0.9,0.9)
    love.graphics.rectangle("line", x, y, w, h)
    if value then
        love.graphics.setColor(0.2, 0.7, 0.2)
        love.graphics.rectangle("fill", x+2, y+2, w-4, h-4)
    end
    love.graphics.setColor(1,1,1)
    love.graphics.print(label, x + w + 6, y-2)
    return w, h
end

-- Handle simple mouse toggles
function love.mousepressed(mx, my, button)
    -- toggles area (left top UI)
    local ux, uy = 10, 10
    local rowH = 20
    local toggles = {
        {key="showESP", label="ESP"},
        {key="teamCheck", label="Team Check"},
        {key="showBoxes", label="Caixas"},
        {key="showNames", label="Nomes"},
        {key="highlightSuspect", label="Highlight Suspeitos"},
    }
    for i, t in ipairs(toggles) do
        local rx = ux; local ry = uy + (i-1)*rowH
        if mx >= rx and mx <= rx+20 and my >= ry and my <= ry+14 then
            ui[t.key] = not ui[t.key]
            log(("UI: %s => %s"):format(t.label, tostring(ui[t.key])))
            return
        end
    end

    -- slider region for maxRange (simple)
    local sx, sy, sw, sh = 10, uy + #toggles*rowH + 8, 160, 12
    if mx >= sx and mx <= sx+sw and my >= sy and my <= sy+sh then
        local rel = (mx - sx) / sw
        ui.maxRange = math.floor(100 + rel * 900)
        log("UI: Range => " .. ui.maxRange)
    end

    -- sensitivity slider
    local sx2, sy2, sw2 = 10, sy + 24, 160
    if mx >= sx2 and mx <= sx2+sw2 and my >= sy2 and my <= sy2+12 then
        local rel = (mx - sx2) / sw2
        ui.detectionSensitivity = math.max(0, math.min(1, rel))
        log("UI: Sensitivity => " .. string.format("%.2f", ui.detectionSensitivity))
    end
end

-- Draw player ESP box + name
local function drawESPForPlayer(p)
    local dist = distance(localPlayer, p)
    if dist > ui.maxRange then return end
    -- team check
    if ui.teamCheck and p.team == localPlayer.team then
        -- friend; depending on preferences we may skip or color differently
        -- we still can show if showBoxes true
    end

    -- determine visible color
    local col = (p.team == localPlayer.team) and ui.boxColorFriend or ui.boxColorEnemy
    if p._sus and (p._sus.teleport or p._sus.stable_aim) then
        -- override for suspects
        col = {1,0.4,0,0.95}
    end

    local w, h = 48, 64
    local x = p.x - w/2
    local y = p.y - h/2

    if ui.showBoxes then
        love.graphics.setColor(col)
        love.graphics.rectangle("line", x, y, w, h, 6,6)
    end
    if ui.showNames then
        love.graphics.setColor(1,1,1)
        love.graphics.print(p.name .. " [" .. (p.hp) .. "]", x, y - 16)
    end

    -- draw simple health bar
    local hbW, hbH = w, 6
    local hbX, hbY = x, y + h + 4
    love.graphics.setColor(0.2,0.2,0.2,0.8)
    love.graphics.rectangle("fill", hbX, hbY, hbW, hbH)
    love.graphics.setColor(0.1, 0.9, 0.3)
    love.graphics.rectangle("fill", hbX, hbY, hbW * (p.hp/100), hbH)

    -- small indicator for suspicion
    if p._sus then
        local s = ""
        for k,v in pairs(p._sus) do if v then s = s .. k .. " " end end
        if s ~= "" then
            love.graphics.setColor(1,0.5,0)
            love.graphics.print("SUSP: "..s, x, hbY + 10)
        end
    end
end

function love.draw()
    -- background
    love.graphics.clear(0.08, 0.08, 0.09)
    love.graphics.setFont(bigfont)
    love.graphics.setColor(1,1,1)
    love.graphics.print("ESP Tester (Simulado) - TeamCheck + Detector (Somente teste)", 14, WINDOW_H-34)
    love.graphics.setFont(font)

    -- draw players
    for _, p in ipairs(players) do
        -- draw simple avatar
        love.graphics.setColor(0.7,0.7,0.7)
        love.graphics.circle("fill", p.x, p.y, 10)
        if p.team == TEAM_FRIEND then
            love.graphics.setColor(0.2,0.7,0.2)
        else
            love.graphics.setColor(0.8,0.2,0.2)
        end
        love.graphics.circle("line", p.x, p.y, 12)
    end

    -- draw ESP overlay
    if ui.showESP then
        for _, p in ipairs(players) do
            if p.alive then
                if ui.teamCheck and p.team == localPlayer.team then
                    -- show friend if want; here we still draw but tinted green
                end
                drawESPForPlayer(p)
            end
        end
    end

    -- draw UI panel
    local ux, uy = 10, 10
    love.graphics.setColor(0,0,0,0.5)
    love.graphics.rectangle("fill", ux-6, uy-6, 220, 220, 6,6)
    love.graphics.setColor(1,1,1)
    love.graphics.print("Controls:", ux, uy)
    local toggles = {
        {key="showESP", label="ESP"},
        {key="teamCheck", label="Team Check"},
        {key="showBoxes", label="Caixas"},
        {key="showNames", label="Nomes"},
        {key="highlightSuspect", label="Highlight Suspeitos"},
    }
    for i, t in ipairs(toggles) do
        local rx = ux; local ry = uy + 18 + (i-1)*20
        drawToggle(rx, ry, t.label, ui[t.key])
    end

    love.graphics.print("Max Range: " .. tostring(ui.maxRange) .. " px", ux, uy + 18 + #toggles*20 + 10)
    love.graphics.setColor(0.3,0.3,0.3)
    love.graphics.rectangle("fill", ux, uy + 18 + #toggles*20 + 28, 160, 12)
    love.graphics.setColor(0.8,0.8,0.2)
    local rx = (ui.maxRange - 100) / 900
    love.graphics.rectangle("fill", ux, uy + 18 + #toggles*20 + 28, 160 * rx, 12)

    love.graphics.setColor(1,1,1)
    love.graphics.print("Detection Sensitivity", ux, uy + 18 + #toggles*20 + 28 + 20)
    love.graphics.setColor(0.3,0.3,0.3)
    love.graphics.rectangle("fill", ux, uy + 18 + #toggles*20 + 28 + 40, 160, 12)
    love.graphics.setColor(0.2,0.6,1)
    love.graphics.rectangle("fill", ux, uy + 18 + #toggles*20 + 28 + 40, 160 * ui.detectionSensitivity, 12)

    -- logs
    local lx = WINDOW_W - 340
    local ly = 12
    love.graphics.setColor(0,0,0,0.5)
    love.graphics.rectangle("fill", lx-6, ly-6, 330, 160, 6,6)
    love.graphics.setColor(1,1,1)
    love.graphics.print("Logs:", lx, ly)
    for i=1, #logs do
        love.graphics.print(logs[i], lx, ly + 16 * i)
    end

    -- Legend
    love.graphics.setColor(1,1,1)
    love.graphics.print("Legend: green friend, red enemy, orange suspect", ux, WINDOW_H - 60)
end

-- Keyboard: quick controls
function love.keypressed(key)
    if key == "f1" then ui.showESP = not ui.showESP; log("ESP toggled: "..tostring(ui.showESP)) end
    if key == "f2" then ui.teamCheck = not ui.teamCheck; log("TeamCheck toggled: "..tostring(ui.teamCheck)) end
    if key == "r" then spawnSimulatedPlayers(); log("Respawn simulated players") end
    if key == "escape" then love.event.quit() end
end